# logs_elk_rsyslog_stand01
ELK RSYSLOG NGINX

1. Развёрнуто 4 виртуалки - Ansible, Nginx, Rsyslog, ELK
2. Все логи с web собираются и локально и удаленно

Для отправки логов на удаленный сервер добавляем в конфиг /etc/rsyslog.conf строку:

`*.*    @192.168.50.202:514`

3. Все логи с nginx уходят на удаленный сервер (локально только критичные)

Для локального сбора только критичных логов nginx правим в конфиге /etc/nginx/nginx.conf строку:

`error_log  /var/log/nginx/error.log crit;`

Для отправки всех логов nginx добавляем в конфиге /etc/nginx/nginx.conf строку:

`error_log syslog:server=192.168.50.202:514 debug;`

`access_log syslog:server=192.168.50.202:514,severity=debug;`

Логи аудита также уходят на удаленную систему
