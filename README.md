# Отчёт по домашнему заданию  
## «Сборка и модификация Android»

**Автор:** Мелешкин Кирилл  
**Курс:** Nuclear System 2026  

---

## Ход работы

### Подготовка окружения

1. Установка зависимостей:
   ```
   sudo apt install git-core gnupg flex bison build-essential zip curl zlib1g-dev libc6-dev-i386 libncurses5-dev x11proto-core-dev libx11-dev libreadline6-dev libgl1-mesa-dev python3 python3-pip python3-venv openjdk-17-jdk repo
   ```
2. Установка `repo`:
   ```
   curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo && chmod a+x ~/bin/repo
   ```
3. Настройка `ccache`:
   ```
   export USE_CCACHE=1 && ccache -M 50G
   ```
4. Проверка KVM:
   ```
   grep -c -w "vmx|svm" /proc/cpuinfo
   ```
   Вывод: `16`

---

### Синхронизация исходного кода

Загрузка исходного кода выполнялась командой:
```
repo sync -j8
```

Во время синхронизации возникли ошибки:
- HTTP 429
- зависимые процессы git и файлы `shallow.lock`
- локальные изменения в `prebuilts/build-tools/linux-x86/bin/nsjail`

Для устранения проблем были уменьшены количество потоков, удалены оставшиеся lock-файлы и восстановлены изменённые файлы через Git. После повторного запуска синхронизация успешно завершилась. 

---

### Сборка AOSP

После настройки окружения выбрана конфигурация:
```
source build/envsetup.sh
lunch aosp_cf_x86_64_phone aosp_current userdebug
m -j8
```

Во время сборки возникли следующие проблемы:
- `nsjail` не запускался из-за ограничений AppArmor (Permission denied при создании namespace).
- После устранения этой ошибки появились сбои `ccache`, связанные с невозможностью записи во временный каталог (Read-only file system).
- После отключения `ccache` процесс `soong_build` завершался с ошибкой `Killed (exit 137)`.

Предпринимались попытки изменить настройки `ccache`, использовать отдельный временный каталог и уменьшить количество потоков сборки, однако полностью завершить сборку за отведённое время не удалось.

Также во время сборки были проблемы с `kotlinx.serialization` и некоторыми другими незначительными проблемами, которые удалось решить.

**Сборка AOSP завершилась на 85%.**

---

### Сборка LineageOS

Также была попытка собрать LineageOS. Сборка завершилась на 99%, но в силу ограниченности времени было принято решение вернуться на AOSP. Ошибка была связана с неполной синхронизацией `frameworks/base`.

**Сборка LineageOS завершилась на 99%.**

---

### Подготовка модификаций

Параллельно с основными этапами сборки были подготовлены три модификации, которые планировалось применить после запуска эмулятора. Файлы были подготовлены в `~/tmp/mods`.

#### 1. Цветовой фильтр в SurfaceFlinger

Подготовлен патч: `~/tmp/mods/surfacefinger/grayscale.patch`

Добавление цветовой матрицы (усреднение каналов) для перевода изображения в чёрно-белый режим.

#### 2. Добавление своего APK в сборку

Структура: `~/tmp/mods/myapp/` с файлами:
- `MyApp.apk` (ProtectedApp)
- `Android.mk` (описание сборки)

Приложение встраивается как системное (PRESIGNED) через `PRODUCT_PACKAGES += MyApp` в конфигурацию устройства.

<p align="center">
  <a href="https://github.com/alt255c/ProtectedApp">
    <img src="https://github.com/alt255c.png" width="48" style="border-radius:50%;" alt="alt255c"/>
  </a>
  <br>
  <a href="https://github.com/alt255c/ProtectedApp"><strong>alt255c/ProtectedApp</strong></a>
  <br><br>
  <a href="https://github.com/alt255c/ProtectedApp/releases/latest">
    <img src="https://img.shields.io/github/v/release/alt255c/ProtectedApp" alt="Latest Release">
  </a>
  <a href="https://github.com/alt255c/ProtectedApp/releases">
    <img src="https://img.shields.io/github/downloads/alt255c/ProtectedApp/total" alt="Total Downloads">
  </a>
  <a href="https://github.com/alt255c/ProtectedApp/releases/latest">
    <img src="https://img.shields.io/github/downloads/alt255c/ProtectedApp/latest/total" alt="Downloads of Latest Release">
  </a>
  <a href="https://github.com/alt255c/ProtectedApp/blob/main/LICENSE">
    <img src="https://img.shields.io/github/license/alt255c/ProtectedApp" alt="GitHub License">
  </a>
</p>

#### 3. Замена bootanimation

Структура: `~/tmp/mods/bootanimation/`:
- `desc.txt` (разрешение 1080*1920, 30 fps)
- `part0/` и `part1/` с кадрами

Анимация заменяется через `PRODUCT_COPY_FILES += $(LOCAL_PATH)/bootanimation.zip:system/media/bootanimation.zip`.

---

## Результат

- Сборка AOSP завершилась на **85%**.
- Сборка LineageOS завершилась на **99%**.

Хотя полностью собрать систему за отведённое время не получилось, удалось поэтапно решить ошибки синхронизации, проблемы с `repo`, `nsjail` и `ccache`, что сильно приблизило сборку к завершению.
