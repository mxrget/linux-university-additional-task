# Домашнее задание
## Управление логами и логгирование
### Введение
Давайте познакомимся с утилитой `journalctl` для просмотра системного журнала.

#### 1. Просмотр логов системы

Первым делом просто посмотрим все логи системы:
`journalctl`

#### 2. Фильтрация логов

Есть несколько способов, как отфильтровать логи. Первый - по сервису:
`journalctl -u snap.docker.dockerd.service` - логи докера (изучить, логи каких сервисов ещё можно посмотреть - `systemctl list-units --type=service`)

Второй - по времени:
`journalctl --since "10 minutes ago"` - логи не старше, чем 10-минутной давности

Третий - по уровню критичности:
`journalctl -p err`

#### 3. Очистка логов

Очень тяжело, когда так много сообщений в логах, но их можно почистить. Давайте очистим логи старше, чем 2-дневной давности:
`sudo journalctl --vacuum-time=2d`
*Это полезно ещё и тем, что освобождает место на диске: у меня вычистилось 390 МБ логов, в которые я вряд ли когда-то посмотрю.*

#### 4. Настройка `rsyslog` для отправки логов в файл

Для этого нужно открыть конфигурационный файл `/etc/rsyslog.conf`:
`sudo nano /etc/rsyslog.conf`

Давайте добавим инструкцию для записи логов уровня "информационные" в новый файл:
`if $syslogseverity-text == 'info' then {
    action(type="omfile" file="/var/log/custom_info.log")
}
`
И перезапустим сервис:
`sudo systemctl restart rsyslog`

#### 5. Проверка настроек

Запишем вручную сообщение в логи:
`logger "Это тестовое сообщение уровня INFO"`

И убеждаемся, что это сообщение появилось в новом файле, в который мы сами и настроили записывать логи:
`cat /var/log/custom_info.log`

### Задача для выполнения

Теперь само задание:
1. Сконфигурировать `rsyslog` так, чтобы все логи `nginx` записывались в файл `/var/log/nginx.log`, ограничить размер файла до 50 МБ и настроить автоматическое удаление старых логов.
2. Создать скрипт для поиска ошибок в логах за последние 7 дней и записи их в отдельный файл.

### Решение

1. Устанавливаем nginx:
`sudo apt update
sudo apt install linux`.
2. Настраиваем rsyslog для записи логов:
Открываем файл конфигурации - `sudo nano /etc/rsyslog.d/nginx.conf`, добавляем в него:
`if $programname == 'nginx' then /var/log/nginx.log
& stop`, и перезапускаем сервис - `sudo systemctl restart rsyslog`.
3. Перенаправляем логи nginx в syslog:
Открываем конфигурацию nginx - `sudo nano /etc/nginx/nginx.conf`, в секции http добавляем строки:
`access_log syslog:server=unix:/dev/log;
error_log syslog:server=unix:/dev/log;`, и перезапускаем nginx.
4. Проверяем логирование - выполняем запрос к серверу `curl http://localhost`, проверяем логи - `tail -f /var/log/nginx.log`.
5. Чтобы ограничить размер логов, настраиваем управление их размером:
Создаём файл конфигурации - `sudo nano /etc/logrotate.d/nginx`, добавляем в него строку `size 50M`. Чтобы проверить работу - `sudo logrotate -f /etc/logrotate.d/nginx`. Если не выводит ничего - значит, всё правильно, никаких ошибок же ещё не было, нечего логировать.
6. Создаём скрипт для поиска ошибок в логах:
`sudo nano find_nginx_errors.sh`.
7. Вставляем в него следующий код:
```
#!/bin/bash

# Файл логов nginx
LOG_FILE="/var/log/nginx.log"

# Файл для ошибок
ERROR_FILE="/var/log/nginx/error.log"

# Поиск ошибок за последние 7 дней
grep -i "error" "$LOG_FILE" | awk -v date="$(date -d '7 days ago' '+%Y-%m-%d')" '$0 ~ date || $0 > date' > "$ERROR_FILE"

echo "Ошибки за последние 7 дней сохранены в $ERROR_FILE"
```
8. Делаем скрипт исполняемым: `sudo chmod +x find_nginx_errors.sh`, запускаем: `./find_nginx_errors.sh`
9. По идее, теперь у нас есть файл с логами (в котором записи автоудаляются), есть скрипт, который ищет в логах записи и автоматически переправляет их в нужные файлы. Но это надо проверить!

### Проверка
1. Заходим в `/etc/nginx/conf.d/test.conf` и пишем в него несуществующий IP сервера, до которого мы не сможем достучаться:
`
http {
    server {
        listen 80;
        server_name localhost;
        location /test-error {
            proxy_pass http://127.0.0.2;
        }
    }
}
`
2. Перезагружаем nginx: `sudo systemctl reload nginx`.
3. Пробуем достучаться до нашего несуществующего адреса: `curl http://localhost/test-error` - получаем ошибку 404. Ошибку 502 у меня смоделировать не получилось, поэтому она не появится в `errors.log`, а вот в `access.log` оно появляется, как и должно быть:
![image](https://github.com/mxrget/linux-university-additional-task/blob/main/access_logs.png)
4. Готово!
