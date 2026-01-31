# Linux для PHP разработчика

## Основные команды

### Навигация и файлы

```bash
# Текущая директория
pwd

# Список файлов
ls               # Простой список
ls -la           # Детальный список (включая скрытые)
ls -lh           # С человеко-читаемыми размерами
ls -ltr          # Сортировка по времени (новые внизу)

# Переход в директорию
cd /var/www/html
cd ..            # На уровень выше
cd ~             # В домашнюю директорию
cd -             # В предыдущую директорию

# Создание/удаление
mkdir mydir      # Создать директорию
mkdir -p a/b/c   # Создать вложенные директории
rm file.txt      # Удалить файл
rm -r mydir      # Удалить директорию рекурсивно
rm -rf mydir     # Удалить без подтверждения (осторожно!)

# Копирование/перемещение
cp source.txt dest.txt           # Копировать файл
cp -r source_dir dest_dir        # Копировать директорию
mv old_name.txt new_name.txt     # Переименовать/переместить
```

### Просмотр содержимого файлов

```bash
# Вывести весь файл
cat file.txt

# Постраничный просмотр
less file.txt    # Space - вперед, q - выход
more file.txt

# Первые/последние строки
head file.txt          # Первые 10 строк
head -n 20 file.txt    # Первые 20 строк
tail file.txt          # Последние 10 строк
tail -n 50 file.txt    # Последние 50 строк
tail -f log.txt        # Следить за файлом в реальном времени (логи!)

# Подсчет строк
wc -l file.txt   # Количество строк
```

### Поиск

```bash
# Поиск файлов
find /var/www -name "*.php"                    # По имени
find /var/www -name "*.php" -type f            # Только файлы
find /var/www -name "*.log" -mtime -7          # Измененные за последние 7 дней
find /var/www -name "*.log" -size +100M        # Больше 100MB

# Поиск в содержимом
grep "error" log.txt                           # Простой поиск
grep -i "error" log.txt                        # Без учета регистра
grep -r "function" /var/www                    # Рекурсивный поиск
grep -n "error" log.txt                        # С номерами строк
grep -v "debug" log.txt                        # Инвертировать (все кроме "debug")
grep -A 5 "error" log.txt                      # +5 строк после
grep -B 5 "error" log.txt                      # +5 строк до
grep -C 5 "error" log.txt                      # +5 строк до и после

# Продвинутый grep
grep -E "error|warning" log.txt                # Regex (OR)
grep -P "\d{3}-\d{3}-\d{4}" log.txt           # Perl regex
```

---

## Права доступа

### Понимание прав

```bash
ls -l
# -rw-r--r-- 1 www-data www-data 1234 Jan 28 12:00 file.txt
# │││ │││ │││
# │││ │││ │││
# │││ │││ └─► Other (все остальные)
# │││ └─────► Group
# └─────────► Owner
#
# r = read (4)
# w = write (2)
# x = execute (1)
```

**Расшифровка:**
- `-` - файл, `d` - директория, `l` - символическая ссылка
- `rw-` (owner) = 6 (read + write)
- `r--` (group) = 4 (read)
- `r--` (other) = 4 (read)

### Изменение прав

```bash
# Символьная нотация
chmod u+x script.sh      # Добавить execute для owner
chmod g-w file.txt       # Убрать write для group
chmod o+r file.txt       # Добавить read для others
chmod a+x script.sh      # Добавить execute для all

# Числовая нотация (чаще используется)
chmod 644 file.txt       # rw-r--r--
chmod 755 script.sh      # rwxr-xr-x
chmod 777 file.txt       # rwxrwxrwx (ОПАСНО! Не используйте в production)
chmod 600 secret.key     # rw------- (только owner может читать/писать)

# Рекурсивно
chmod -R 755 /var/www/html
```

### Типичные права для веб-приложений

```bash
# Директории - 755 (rwxr-xr-x)
find /var/www/html -type d -exec chmod 755 {} \;

# Файлы - 644 (rw-r--r--)
find /var/www/html -type f -exec chmod 644 {} \;

# Хранилище Laravel - 775 (rwxrwxr-x)
chmod -R 775 storage bootstrap/cache

# Владелец - www-data (пользователь веб-сервера)
chown -R www-data:www-data /var/www/html
```

### Изменение владельца

```bash
# Сменить владельца
chown user file.txt
chown user:group file.txt
chown -R www-data:www-data /var/www

# Только группу
chgrp group file.txt
```

---

## Процессы

### Просмотр процессов

```bash
# Список процессов
ps aux                   # Все процессы
ps aux | grep php        # Фильтр по имени
ps -ef                   # Альтернативный формат

# Интерактивный мониторинг
top                      # Обновляется в реальном времени
htop                     # Улучшенная версия top (нужно установить)

# Процессы по пользователю
ps -u www-data

# Дерево процессов
pstree                   # Дерево всех процессов
pstree -p                # С PID
```

### Управление процессами

```bash
# Запуск в фоне
php artisan queue:work &

# Просмотр фоновых процессов
jobs

# Вернуть процесс на передний план
fg %1

# Убить процесс
kill 1234                # По PID
kill -9 1234             # Принудительно
killall php-fpm          # По имени (все процессы)
pkill -f "queue:work"    # По части команды

# Сигналы
kill -HUP 1234           # Reload (SIGHUP)
kill -TERM 1234          # Graceful shutdown (SIGTERM)
kill -KILL 1234          # Force kill (SIGKILL)
```

### Мониторинг ресурсов

```bash
# Использование памяти
free -h                  # Общая память
free -h -s 5             # Обновлять каждые 5 секунд

# Использование диска
df -h                    # Все диски
du -sh /var/www          # Размер директории
du -sh /var/www/*        # Размер каждой поддиректории
du -sh /var/www/* | sort -h   # Отсортировать по размеру

# Load average
uptime
top
```

---

## Файловая система

### Важные директории

```
/                    # Корень
├── bin/            # Основные исполняемые файлы
├── etc/            # Конфигурационные файлы
│   ├── nginx/      # Конфиг Nginx
│   ├── php/        # Конфиг PHP
│   └── supervisor/ # Конфиг Supervisor
├── home/           # Домашние директории пользователей
├── opt/            # Дополнительное ПО
├── tmp/            # Временные файлы
├── usr/            # Пользовательские программы
│   ├── bin/        # Исполняемые файлы
│   └── local/      # Локально установленное ПО
└── var/            # Переменные данные
    ├── log/        # Логи
    └── www/        # Веб-файлы
```

### Символические ссылки

```bash
# Создать симлинк
ln -s /path/to/original /path/to/link

# Laravel storage
ln -s /var/www/html/storage/app/public /var/www/html/public/storage

# Просмотр цели симлинка
ls -l link
readlink link
```

---

## Сеть

### Проверка соединения

```bash
# Ping
ping google.com
ping -c 4 google.com     # 4 пакета

# Traceroute
traceroute google.com

# DNS lookup
nslookup google.com
dig google.com
host google.com

# Проверка порта
telnet localhost 3306
nc -zv localhost 3306    # netcat
```

### Сетевые соединения

```bash
# Открытые порты
netstat -tuln                    # Listening ports
netstat -tulnp                   # С процессами (нужен root)
ss -tuln                         # Современная альтернатива netstat

# Активные соединения
netstat -an | grep ESTABLISHED

# Кто слушает порт 80?
lsof -i :80
netstat -tulnp | grep :80

# Все PHP-FPM соединения
netstat -an | grep php-fpm
```

### Firewall (UFW)

```bash
# Статус
sudo ufw status

# Разрешить порт
sudo ufw allow 80
sudo ufw allow 443
sudo ufw allow 22

# Запретить
sudo ufw deny 3306

# Разрешить с конкретного IP
sudo ufw allow from 192.168.1.100 to any port 22

# Включить/выключить
sudo ufw enable
sudo ufw disable
```

---

## Логи

### Основные логи

```bash
# Системные логи
tail -f /var/log/syslog
tail -f /var/log/messages

# Nginx
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# PHP-FPM
tail -f /var/log/php8.2-fpm.log

# Laravel
tail -f storage/logs/laravel.log

# MySQL
tail -f /var/log/mysql/error.log
```

### Анализ логов

```bash
# Количество запросов по IP
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10

# Самые частые URL
cat access.log | awk '{print $7}' | sort | uniq -c | sort -rn | head -10

# Коды ответов
cat access.log | awk '{print $9}' | sort | uniq -c | sort -rn

# 500 ошибки
grep " 500 " access.log | tail -20

# За последний час
grep "$(date -d '1 hour ago' '+%d/%b/%Y:%H')" access.log
```

---

## Архивация и сжатие

```bash
# TAR архив
tar -czf archive.tar.gz /path/to/dir     # Создать (gzip)
tar -xzf archive.tar.gz                  # Распаковать
tar -tzf archive.tar.gz                  # Просмотр содержимого

# ZIP
zip -r archive.zip /path/to/dir          # Создать
unzip archive.zip                        # Распаковать
unzip -l archive.zip                     # Просмотр содержимого

# Бэкап базы данных
mysqldump -u root -p database_name | gzip > backup.sql.gz

# Восстановление
gunzip < backup.sql.gz | mysql -u root -p database_name
```

---

## Текстовые редакторы

### Nano (простой)

```bash
nano file.txt

# Горячие клавиши (внизу экрана):
# Ctrl+O - сохранить
# Ctrl+X - выход
# Ctrl+K - вырезать строку
# Ctrl+U - вставить
```

### Vim (мощный, но сложный)

```bash
vim file.txt

# Режимы:
# i     - режим вставки (Insert mode)
# Esc   - командный режим
# :w    - сохранить
# :q    - выход
# :wq   - сохранить и выйти
# :q!   - выйти без сохранения
# /text - поиск
# dd    - удалить строку
# yy    - копировать строку
# p     - вставить
```

---

## Переменные окружения

```bash
# Просмотр
env                      # Все переменные
echo $PATH               # Конкретная переменная
echo $USER
echo $HOME

# Установка (временно)
export VAR_NAME="value"

# Установка (постоянно)
echo 'export VAR_NAME="value"' >> ~/.bashrc
source ~/.bashrc         # Применить изменения
```

---

## Cron (планировщик задач)

```bash
# Редактировать crontab
crontab -e

# Формат:
# ┌───────────── минута (0 - 59)
# │ ┌───────────── час (0 - 23)
# │ │ ┌───────────── день месяца (1 - 31)
# │ │ │ ┌───────────── месяц (1 - 12)
# │ │ │ │ ┌───────────── день недели (0 - 6, 0 = воскресенье)
# │ │ │ │ │
# * * * * * команда

# Примеры:
# Каждую минуту
* * * * * php /var/www/artisan schedule:run

# Каждый час
0 * * * * /path/to/script.sh

# Каждый день в 2:30
30 2 * * * /path/to/backup.sh

# Каждый понедельник в 9:00
0 9 * * 1 /path/to/script.sh

# Каждые 5 минут
*/5 * * * * /path/to/script.sh

# Просмотр crontab
crontab -l
```

---

## Системные сервисы (systemd)

```bash
# Статус сервиса
systemctl status nginx
systemctl status php8.2-fpm
systemctl status mysql

# Запуск/остановка/перезапуск
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx       # Перечитать конфиг без остановки

# Автозапуск
systemctl enable nginx       # Включить автозапуск
systemctl disable nginx      # Выключить автозапуск

# Логи сервиса
journalctl -u nginx          # Все логи
journalctl -u nginx -f       # Следить в реальном времени
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx --since "2024-01-28 10:00"
```

---

## Полезные команды для PHP разработчика

### Composer

```bash
# Установка зависимостей
composer install
composer install --no-dev    # Production (без dev-зависимостей)

# Обновление
composer update
composer update vendor/package

# Autoload
composer dump-autoload
composer dump-autoload -o    # Оптимизированный (production)
```

### Laravel Artisan

```bash
# Очистка кешей
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear

# Оптимизация (production)
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan optimize

# Миграции
php artisan migrate
php artisan migrate:rollback
php artisan migrate:fresh --seed

# Queue workers
php artisan queue:work
php artisan queue:work --daemon
php artisan queue:restart
```

### Git

```bash
# Клонирование
git clone https://github.com/user/repo.git

# Статус
git status
git log --oneline

# Ветки
git branch                   # Список веток
git checkout -b feature      # Создать и переключиться
git branch -d feature        # Удалить ветку

# Коммиты
git add .
git commit -m "Message"
git push origin main

# Pull
git pull origin main
git fetch && git rebase origin/main
```

---

## Мониторинг производительности

```bash
# CPU и память
top
htop

# Disk I/O
iostat
iotop

# Сетевая активность
iftop
nethogs

# PHP-FPM статус
# Настроить в pool.d/www.conf:
# pm.status_path = /status
curl http://localhost/status

# MySQL процессы
mysql -e "SHOW PROCESSLIST;"

# Долгие запросы
mysql -e "SHOW FULL PROCESSLIST;" | grep -v "Sleep"
```

---

## Безопасность

### Обновления

```bash
# Ubuntu/Debian
sudo apt update              # Обновить список пакетов
sudo apt upgrade             # Установить обновления
sudo apt dist-upgrade        # Обновления с зависимостями

# Автоматические обновления безопасности
sudo apt install unattended-upgrades
```

### SSH ключи

```bash
# Генерация ключа
ssh-keygen -t ed25519 -C "your_email@example.com"

# Копирование на сервер
ssh-copy-id user@server

# SSH с конкретным ключом
ssh -i ~/.ssh/id_rsa user@server
```

### Fail2Ban (защита от bruteforce)

```bash
# Статус
sudo fail2ban-client status
sudo fail2ban-client status sshd

# Разбанить IP
sudo fail2ban-client unban 192.168.1.100
```

---

## Ключевые вопросы для интервью

- Как проверить какой процесс использует порт 80?
- Как найти большие файлы на диске?
- Объясните права 755 и 644
- Как следить за логами в реальном времени?
- Какая разница между `kill` и `kill -9`?
- Как настроить cron job для Laravel scheduler?
- Где находятся конфигурационные файлы Nginx/PHP-FPM?
- Как перезапустить PHP-FPM без даунтайма?
- Что такое load average и как его интерпретировать?
- Как проверить использование памяти процессами?
