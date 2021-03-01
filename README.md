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

4. Логи аудита уходят на удаленную систему при изменении конфигурационных файлов nginx:

Активировать плагин для auditd:

`vi /etc/audisp/plugins.d/syslog.conf`

 ```
 active = yes
direction = out
path = builtin_syslog
type = builtin
args = LOG_INFO LOG_LOCAL3
format = string
```
Перезапустить auditd

`service auditd restart`

Добавить правила:

`auditctl -a exit,always -F path=/etc/nginx/nginx.conf -F perm=wa`

`auditctl -a exit,always -F path=/etc/nginx/conf.d/nginx_default.conf -F perm=wa`
