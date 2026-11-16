# Сборка и публикация APK

## Автоматическая сборка

Workflow `.github/workflows/main-release.yml` запускается при каждом push в
`main` и вручную через GitHub Actions. Он:

1. устанавливает Java 17, Android SDK 35 и NDK;
2. собирает Rust `cdylib` для `arm64-v8a` и `armeabi-v7a`;
3. собирает universal, arm64 и arm32 release APK;
4. обновляет rolling prerelease с тегом `main` в разделе **Releases**.

Актуальные APK доступны в релизе **Main build**. Это CI-сборки для проверки
последнего состояния ветки, а не версионные релизы.

Для запуска вручную откройте **Actions → Build Android APK (main) → Run workflow**.

## Версионный релиз

Для публичного релиза измените `versionName` и `versionCode` в
`app/build.gradle.kts`, создайте тег и отправьте его:

```powershell
git tag v1.2.4
git push origin v1.2.4
```

Текущий workflow запускается только для `main`, поэтому версионный релиз можно
создать через **Releases → Draft a new release**, выбрать тег и прикрепить APK
из CI-artifact или собрать их локально.

## Локальная сборка

Нужны Rust, JDK 17, Android SDK/NDK и Gradle wrapper проекта:

```powershell
.\build_so.bat
.\build_apk.bat
```

Готовые APK будут в `app/release/`. Каталоги `target/`, `app/build/`,
`app/release/`, `local.properties`, ключи и native `.so` не следует коммитить.

## Подпись

CI создаёт временный ключ только для автоматической сборки. Для настоящего
релиза используйте собственный keystore и задайте в `local.properties`:

```properties
KEYSTORE_FILE=../release.keystore
KEYSTORE_PASSWORD=...
KEY_ALIAS=...
KEY_PASSWORD=...
```

Файл keystore и пароли должны храниться в GitHub Secrets или локально, но не в
репозитории.
