# logs_elk_rsyslog_stand01

**RSYSLOG NGINX**

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

**ELK**

ELK stack (Elasticsearch, Kibana, Logstash, Filebeat)
CentOS 7 2004

1. Установка Elasticsearch (хранение логов)

Обновление пакетов и установка java:

`yum update`

`yum install java -y`

`java - version`

Добавление ключа и репозитория Elasticsearch:

`rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch`

`vi /etc/yum.repos.d/elasticsearch.repo`

```
[elasticsearch]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

Установка Elasticsearch:

`yum install elasticsearch -y`

Конфигурация Elasticsearch:

`vi /etc/elasticsearch/elasticsearch.yml`

```
bootstrap.memory_lock: true

network.host: localhost # только локальный интерфейс!
http.port: 9200

# Для кластера
# network.host: 0.0.0.0
# discovery.seed_hosts: ["127.0.0.1", "[::1]"]

path.data: /data/elasticsearch # директория для хранения данных

# Стандартная директория для хранения данных
# path.data: /var/lib/elasticsearch
```

`mkdir /data/elasticsearch`

`chmod 2750 /data/elasticsearch/`

`vi /etc/sysconfig/elasticsearch`

```
MAX_LOCKED_MEMORY=unlimited
```

`systemctl edit elasticsearch`

```
[Service]
LimitMEMLOCK=infinity
```

Запуск сервиса Elasticsearch:

`systemctl daemon-reload`

`systemctl enable elasticsearch`

`systemctl start elasticsearch`

Запрос статуса Elasticsearch:

`curl localhost:9200`

2. Установка Kibana (web-интерфейс)

`rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch`

```
[kibana-7.x]
name=Kibana repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

Установка Kibana:

`yum install kibana -y`

Конфигурация Kibana:

`vi /etc/kibana/kibana.yml`

```
server.host: "0.0.0.0" # Только для теста! В проде использовать nginx reverse proxy!
```

Запуск сервиса Kibana:

`systemctl daemon-reload`

`systemctl enable kibana.service`

`systemctl start kibana.service`

Проверка порта Kibana:

`ss -tulnp | grep 5601`

3. Установка Logstash (сбор и отправка логов в Elasticsearch)

`yum install logstash -y # Repository elasticsearch`

Конфигурация Logstash:

`vi /etc/logstash/conf.d/input.conf`

```
# Описываем приём информации с beats агентов (без ssl)
input {
  beats {
    port => 5044
  }
}
```

`vi /etc/logstash/conf.d/output.conf`

```
# Описываем передачу информации в Elasticsearch (без ssl)
output {
        elasticsearch {
            hosts    => "localhost:9200"
            index    => "nginx-%{+YYYY.MM.dd}" # Разбивка индексов по дням и по типам данных
        }
	#stdout { codec => rubydebug } # Дополнительно отправлять в /var/log/messages
}
```

`vi /etc/logstash/conf.d/output.conf/filter.conf`

```
# Описываем обработку данных (nginx)
filter {
 if [type] == "nginx_access" {
    grok {
        match => { "message" => "%{IPORHOST:remote_ip} - %{DATA:user} \[%{HTTPDATE:access_time}\] \"%{WORD:http_method} %{DATA:url} HTTP/%{NUMBER:http_version}\" %{NUMBER:response_code} %{NUMBER:body_sent_bytes} \"%{DATA:referrer}\" \"%{DATA:agent}\"" }
    }
  }
  date {
        match => [ "timestamp" , "dd/MMM/YYYY:HH:mm:ss Z" ]
  }
}
```

https://grokdebug.herokuapp.com/ - деббагер фильтра grok
https://dev.maxmind.com/geoip/geoip2/geolite2/ - если использовать фильтр geoip

Запуск сервиса Logstash:

`systemctl daemon-reload`

`systemctl enable elasticsearch`

`systemctl start elasticsearch`


4. Установка Filebeat (агент сбор и отправка логов в Logstash)

Добавление ключа и репозитория Elasticsearch:

`rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch`

`vi /etc/yum.repos.d/elasticsearch.repo`

```
[elasticsearch]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

Установка Filebeat:

`yum install filebeat -y`

Конфигурация Filebeat. Отправка логов nginx в logstash

`vi /etc/filebeat/filebeat.yml`

```
filebeat.inputs:
- type: log
  enabled: true
  paths:
      - /var/log/nginx/access.log*
  fields:
    type: nginx_access
  fields_under_root: true
  scan_frequency: 5s

- type: log
  enabled: true
  paths:
      - /var/log/nginx/error.log*
  fields:
    type: nginx_error
  fields_under_root: true
  scan_frequency: 5s

output.logstash:
  hosts: ["192.168.50.203:5044"]
```

Запуск сервиса Filebeat:

`systemctl daemon-reload`

`systemctl enable filebeat`

`systemctl start filebeat`
