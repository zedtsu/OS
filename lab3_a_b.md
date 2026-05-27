**Выбранный вариант:** №3  
**Формулировка задания:** Добавить в изображения формата `.jpg` подписи в виде текста даты съемки (изменения файла) или текущей даты компьютера. Изменение изображения выполняется программой `convert` / `magick` из пакета ImageMagick. Операция выполняется в целевом каталоге и всех его подкаталогах.

---

## Лабораторная 3a. Реализация скрипта на Bash (Linux ВМ)

### 1. Подготовка окружения в Linux ВМ
Перед запуском скрипта необходимо установить пакет `imagemagick` внутри созданной виртуальной машины.

```bash
# Для Debian / Ubuntu:
sudo apt update && sudo apt install -y imagemagick

# Для Arch Linux:
sudo pacman -Syu --noconfirm imagemagick
```

### 2. Скрипт на Bash (`watermark.sh`)
Скрипт принимает один необязательный параметр — путь к каталогу. Если параметр не задан, обрабатывается текущий каталог. Поиск файлов выполняется рекурсивно (включая подкаталоги).

```bash
#!/bin/bash

# Переменная для целевого каталога (по умолчанию - текущий)
TARGET_DIR="\${1:-.}"
LOG_FILE="./bash_script_log.txt"

echo "=== Запуск Bash-скрипта: \((date) ===" >> "\)LOG_FILE"
echo "Целевой каталог: \(TARGET_DIR" >> "\)LOG_FILE"

# Включаем globstar для рекурсивного поиска подкаталогов (**/*.jpg)
shopt -s globstar
shopt -s nocaseglob

# Проверяем существование каталога
if [ ! -d "\$TARGET_DIR" ]; then
    echo "Ошибка: Каталог \(TARGET_DIR не существует!" >> "\)LOG_FILE"
    exit 1
fi

# Проход по всем файлам .jpg во всех подкаталогах
for file in "\$TARGET_DIR"/**/*.jpg; do
    # Проверка на случай, если файлы не найдены
    [ -e "\$file" ] || continue
    
    echo "Обработка файла: \(file" >> "\)LOG_FILE"
    
    # Получаем дату изменения файла в формате ГГГГ-ММ-ДД
    # Если дата файла недоступна, используется текущая дата
    file_date=\((date -r "\)file" +"%Y-%m-%d" 2>/dev/null)
    if [ -z "\$file_date" ]; then
        file_date=\$(date +"%Y-%m-%d")
    fi
    
    # Наложение текста с помощью ImageMagick
    # Параметры: шрифт 36pt, белый цвет, полупрозрачный черный фон, размещение в правом нижнем углу (SouthEast)
    convert "\$file" \
        -gravity SouthEast \
        -font Courier \
        -pointsize 36 \
        -fill white \
        -background '#00000080' \
        -splice 0x50 \
        -annotate +20+10 "\$file_date" \
        "\(file" 2>> "\)LOG_FILE"
        
    if [ \$? -eq 0 ]; then
        echo "Успешно добавлена дата (\$file_date) на: \(file" >> "\)LOG_FILE"
    else
        echo "Ошибка обработки файла: \(file" >> "\)LOG_FILE"
    fi
done

echo "=== Работа Bash-скрипта завершена ===" >> "\$LOG_FILE"
```

### 3. Запуск и проверка в Linux
```bash
# Выдача прав на исполнение
chmod +x watermark.sh

# Запуск в текущем каталоге
./watermark.sh

# Или запуск с указанием конкретной папки
./watermark.sh /home/user/my_photos

# Просмотр результатов работы
cat bash_script_log.txt
```

---

## Лабораторная 3b. Реализация скрипта на Windows PowerShell

### 1. Подготовка окружения в Windows
1. Скачайте и установите **ImageMagick для Windows** с официального сайта. Убедитесь, что при установке отмечена галочка *«Add application directory to your system path»*.
2. Откройте PowerShell от имени Администратора и разрешите выполнение локальных скриптов:
   ```powershell
   Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
   ```

### 2. Скрипт на PowerShell (`watermark.ps1`)
Скрипт также принимает путь к каталогу через параметр `-Path`. Если параметр опущен, обрабатывается текущая рабочая директория. Поиск происходит рекурсивно во всех подпапках.

```powershell
param (
    [string]\$Path = "."
)

\$LogFile = ".\powershell_script_log.txt"
\$CurrentTime = Get-Date -Format "yyyy-MM-dd HH:mm:ss"

Add-Content -Path \(LogFile -Value "=== Запуск PowerShell-скрипта: \)CurrentTime ==="
Add-Content -Path \(LogFile -Value "Целевой каталог: \)Path"

# Проверяем существование пути
if (-not (Test-Path \$Path)) {
    Add-Content -Path \(LogFile -Value "Ошибка: Каталог \)Path не найден."
    exit
}

# Рекурсивный поиск всех файлов .jpg
\(Images = Get-ChildItem -Path\)Path -Filter "*.jpg" -Recurse

if (\$Images.Count -eq 0) {
    Add-Content -Path \$LogFile -Value "Файлы .jpg для обработки не найдены."
    exit
}

# Цикл обработки каждого изображения
foreach (\(file in\)Images) {
    Add-Content -Path \$LogFile -Value "Обработка файла: \((\)file.FullName)"
    
    # Получаем дату последнего изменения файла в формате ГГГГ-ММ-ДД
    \(FileDate =\)file.LastWriteTime.ToString("yyyy-MM-dd")
    if (-not \(FileDate) {\)FileDate = (Get-Date).ToString("yyyy-MM-dd")
    }
    
    # Вызов ImageMagick (в современных версиях используется команда 'magick')
    # Используем оператор '&' для безопасного вызова внешней утилиты с аргументами
    & magick "\((\)file.FullName)" -gravity SouthEast -font Courier -pointsize 36 -fill white -background "#00000080" -splice 0x50 -annotate +20+10 "\(FileDate" "\)(\(file.FullName)" 2>> \)LogFile
    
    if (\$LASTEXITCODE -eq 0) {
        Add-Content -Path \(LogFile -Value "Успешно добавлена дата (\)FileDate) на: \((\)file.Name)"
    } else {
        Add-Content -Path \$LogFile -Value "Ошибка обработки файла: \((\)file.Name)"
    }
}

Add-Content -Path \$LogFile -Value "=== Работа PowerShell-скрипта завершена ==="
```

### 3. Запуск и проверка в Windows
Откройте консоль PowerShell в папке со скриптом и выполните:

```powershell
# Запуск для текущего каталога
.\watermark.ps1

# Запуск для конкретной папки с картинками
.\watermark.ps1 -Path "C:\Users\User\Pictures"

# Просмотр лога работы
Get-Content .\powershell_script_log.txt
```

---

## Заключение (Сравнение подходов)
1. **Рекурсия**: В Bash для глубокого поиска подкаталогов использовалась опция `shopt -s globstar`, в то время как в PowerShell эта задача лаконично решается стандартным флагом `-Recurse` у командлета `Get-ChildItem`.
2. **Работа с метаданными**: Получение даты изменения файла в Linux завязано на системную утилиту `date -r`, а в Windows PowerShell свойства файла доступны напрямую как объектные атрибуты (`$file.LastWriteTime`).
3. **ImageMagick**: Команды синтаксиса наложения текста (`-gravity`, `-annotate`) идентичны на обеих платформах, что обеспечивает легкую переносимость логики утилит автоматизации.
