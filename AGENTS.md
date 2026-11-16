# AGENTS.md

## Обзор проекта

Проект `TgWsProxy` — Android-приложение на Kotlin/Jetpack Compose с нативным
движком на Rust. Приложение поднимает локальный MTProto-прокси (по умолчанию
`127.0.0.1:1443`) и передаёт трафик Telegram через WebSocket/WSS напрямую или
через Cloudflare-прокси.

Основные части:

- `app/src/main/java/com/amurcanov/tgwsproxy/` — Android UI, настройки,
  foreground service, Quick Settings tile и вызовы native API.
- `app/src/main/java/com/amurcanov/tgwsproxy/ui/` — экраны и компоненты
  Jetpack Compose.
- `src/` — Rust `cdylib`: MTProto handshake, криптография, WebSocket/TLS,
  Cloudflare-домены, балансировщик и пул соединений.
- `app/src/main/jniLibs/` — сгенерированные `.so`; они намеренно исключены из
  Git и создаются скриптом `build_so.bat`.
- `build_so.bat` — сборка Rust для `arm64-v8a` и `armeabi-v7a`.
- `build_apk.bat` — сборка и переименование трёх release APK.
- `old_src_golang/` — историческая Go-реализация; рабочий native-движок сейчас
  находится в `src/` на Rust.

## Рабочий процесс

Перед изменением сначала проверь `git status --short` и прочитай соседний код.
Не коммить локальные настройки, ключи, APK, `.so`, каталоги `target/` и
`app/build/`. Не переписывай незнакомые пользовательские изменения.

Изменения, затрагивающие Rust exports, должны одновременно проверяться с
Kotlin-обёрткой `NativeProxy.kt` и интерфейсом JNA `ProxyLibrary`. Сохраняй
совместимые имена и типы C ABI (`StartProxy`, `StopProxy`, `Set*`, `Get*`);
указатели, возвращённые Rust, освобождаются через `FreeString`.

Изменения настроек прокси должны учитывать весь путь значения:
Compose/UI → `SettingsStore` → `ProxyService` → `NativeProxy` → глобальное
состояние Rust. Для фоновой работы не удаляй `START_REDELIVER_INTENT`,
foreground notification или обработку `WakeLock` без проверки поведения на
Android 8+ и Android 14+.

## Сборка и проверки

Для обычной проверки Rust-кода:

```powershell
cargo fmt --all -- --check
cargo check
```

Для Android-кода и APK (нужны JDK 17, Android SDK/NDK и доступный `cargo`):

```powershell
\.\gradlew.bat :app:assembleUniversalDebug
\.\gradlew.bat :app:assembleArm64Debug
\.\gradlew.bat :app:assembleArm32Debug
```

Сначала собери native-библиотеки, если их нет:

```powershell
\.\build_so.bat
```

Полный release-процесс для всех ABI:

```powershell
\.\build_apk.bat
```

`build_so.bat` ищет SDK через `ANDROID_SDK_ROOT`, `ANDROID_HOME` или
`local.properties`, выбирает установленный NDK и копирует библиотеки в
`app/src/main/jniLibs/<abi>/`. `build_apk.bat` ожидает эти файлы и складывает
результаты в игнорируемый каталог `app/release/`.

Перед отправкой изменений запускай подходящие проверки. Если сборка невозможна
из-за отсутствующего SDK/NDK, укажи это явно и отдельно выполни доступные
`cargo fmt --check`/`cargo check` или Gradle-задачи. В репозитории нет отдельного
набора прикладных unit-тестов, поэтому для изменений протокола особенно важны
компиляция обеих частей и ручная проверка запуска/остановки сервиса.

## Архитектурные правила Rust

- `src/lib.rs` содержит глобальный Tokio runtime, состояние единственного
  запущенного прокси и C ABI exports.
- `src/proxy.rs` отвечает за локальный TCP listener, MTProto bridge и пул
  WebSocket-соединений.
- `src/ws.rs` реализует raw WebSocket framing, TLS/plain transport,
  fragmentation, ping/pong и timeouts.
- `src/config.rs` содержит глобальные настройки, DC mapping, cooldowns и
  размеры буферов.
- `src/cfproxy.rs` и `src/balancer.rs` управляют доменами Cloudflare, кешем,
  fallback и выбором маршрута.
- `src/crypto.rs` и `src/faketls.rs` содержат протокольную криптографию и
  fake-TLS подготовку.

Сохраняй отмену через `CancellationToken`, ограничения pool/backoff и очистку
глобального состояния в `StopProxy`. Не блокируй Tokio worker длительными
синхронными сетевыми операциями. При изменении handshake или frame parsing
проверяй границы буферов, таймауты и обработку частичных чтений.

В `src/ws.rs` серверные сертификаты намеренно не проверяются (`NoVerify`) для
совместимости с текущим протоколом. Это чувствительная часть поведения:
изменения TLS должны быть осознанными и сопровождаться проверкой всех режимов
direct/fronting.

## Android-правила

- UI использует Kotlin и Jetpack Compose Material 3; строки добавляй в
  `res/values/strings.xml` и при необходимости в `res/values-ru/strings.xml`.
- Долгие операции выполняй вне главного потока. Native start/stop уже вызываются
  из фоновых потоков; не превращай их в синхронные вызовы из Compose.
- Изменения notification/service должны учитывать разрешение
  `POST_NOTIFICATIONS`, foreground service type `specialUse`, restart после
  убийства процесса и освобождение WakeLock.
- Не добавляй секреты в исходники. Release signing берётся из `local.properties`
  (`KEYSTORE_FILE`, `KEYSTORE_PASSWORD`, `KEY_ALIAS`, `KEY_PASSWORD`), сам файл
  хранилища и `local.properties` не коммитятся.
- Не редактируй сгенерированные `build/`, `target/` и `.so` вручную.

## Стиль и документация

Следуй существующему стилю файла и минимальному объёму изменений. Комментарии
добавляй для ABI, протокольных форматов, таймаутов и нетривиальной синхронизации,
а не для очевидного кода. В описании изменения укажи, какие проверки выполнены
и какие локальные зависимости (SDK/NDK, signing key, устройство) отсутствовали.

