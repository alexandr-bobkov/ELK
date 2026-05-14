
# Домашнее задание к занятию  «ELK» - Бобков Александр
<details>
<summary><b>Задание 1. Elasticsearch</b></summary>

- Установите и запустите Elasticsearch, после чего поменяйте параметр cluster_name на случайный. 

*Приведите скриншот команды 'curl -X GET 'localhost:9200/_cluster/health?pretty', сделанной на сервере с установленным Elasticsearch. Где будет виден нестандартный cluster_name*.

### ОТВЕТ:

**Установка и запуск (с использованием зеркала Yandex Cloud):**
```bash
# 1. Подключение доверенного зеркала Яндекса 
echo "deb [trusted=yes] yandex.ru stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

# 2. Обновление списков пакетов и установка Elasticsearch
sudo apt update && sudo apt install elasticsearch -y

# 3. Настройка имени кластера в /etc/elasticsearch/elasticsearch.yml
# Изменена директива: cluster.name: my-dz

# 4. Запуск службы
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```

**Сброс и генерация пароля администратора (X-Pack Security):**
Поскольку в версии 8.x пароль генерируется автоматически при первой установке пакета, для явного назначения учетных данных пользователя `elastic` выполнена процедура принудительного сброса через встроенную CLI-утилиту:
```bash
/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -a
```
*Результат выполнения:* Сгенерирован новый постоянный токен доступа.

**Проверка состояния кластера:**
Диагностический запрос отправлен через утилиту `curl` по защищенному протоколу HTTPS с флагом `-k` (игнорирование самоподписанного сертификата) и передачей актуальных учетных данных:
```bash
curl -k -u elastic:Kz4ym_Xoj8OhHhQv-Hxr -X GET 'https://localhost:9200/_cluster/health?pretty'
```

**Результат выполнения команды в терминале:**
```json
{
  "cluster_name" : "my-dz",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 3,
  "active_shards" : 3,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "unassigned_primary_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

**Скриншот вывода команды curl с уникальным cluster_name и статусом green:**
![Здоровье кластера Elasticsearch](./img/1.jpg)


</details>

------
------


<details>
<summary><b>Задание 2. Kibana</b></summary>

- Установите и запустите Kibana.

*Приведите скриншот интерфейса Kibana на странице http://<ip вашего сервера>:5601/app/dev_tools#/console, где будет выполнен запрос GET /_cluster/health?pretty*.

------

### ОТВЕТ:
**Установка и запуск:**
```bash
sudo apt update && sudo apt install kibana -y
```

**Обеспечение сетевой доступности:**
Для предоставления внешнего доступа к веб-интерфейсу с хост-машины в файле конфигурации `/etc/kibana/kibana.yml` адрес прослушивания был изменен с `localhost` на все доступные интерфейсы:
```yaml
server.host: "0.0.0.0"
```

Перезапуск и добавление службы в автозагрузку:
```bash
sudo systemctl daemon-reload
sudo systemctl enable kibana
sudo systemctl start kibana
```

**Первоначальная интерактивная настройка (Interactive Setup):**
При первом запуске служба перешла в режим ожидания конфигурации (`interactiveSetup holding setup`). Безопасное связывание выполнено по следующему алгоритму:
1. Выполнен переход в браузере по адресу `http://<IP_сервера>:5601`.
2. В поле проверки введен шестизначный код верификации, полученный из системного лога (`journalctl -u kibana`): `283236`.
3. На сервере Elasticsearch сгенерирован одноразовый токен регистрации для автоматического обмена SSL-сертификатами:
   ```bash
   /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
   ```
4. Полученный токен вставлен в веб-интерфейс Kibana.
5. Финальная авторизация в системе пройдена под суперпользователем `elastic` с использованием пароля `Kz4ym_Xoj8OhHhQv-Hxr`.

Диагностический запрос `GET /_cluster/health?pretty` выполнен во встроенной интерактивной консоли разработчика Dev Tools.

**Скриншот интерфейса Kibana (страница Dev Tools Console):**
![Запрос в консоли Kibana](./img/2.jpg)



</details>

-------
-------

<details>
<summary><b>Задание 3. Logstash</b></summary>

- Установите и запустите Logstash и Nginx. С помощью Logstash отправьте access-лог Nginx в Elasticsearch. 

*Приведите скриншот интерфейса Kibana, на котором видны логи Nginx.*
-------

### ОТВЕТ:

### Задание 3. Logstash

**1. Установка веб-сервера Nginx и конвейера обработки данных Logstash:**
```bash
# Установка пакетов из подключенного зеркала Яндекса и стандартных репозиториев
sudo apt update && sudo apt install logstash nginx -y

# Добавление служб в автозагрузку операционной системы
sudo systemctl enable logstash nginx
sudo systemctl start nginx
```

**2. Конфигурация прав доступа в ОС Linux:**
По умолчанию файлы логов веб-сервера Nginx закрыты для чтения сторонними пользователями. Для того чтобы служба Logstash (работающая под изолированной учетной записью `logstash`) могла беспрепятственно читать поступающие события, были скорректированы права доступа на директорию и сам файл лога:
```bash
# Предоставление прав на чтение директории и файла конфигурации логов
sudo chmod 755 /var/log/nginx
sudo chmod 644 /var/log/nginx/access.log
```

**3. Конфигурация конвейера обработки данных (Pipeline):**
Для автоматического отслеживания изменений в access-логах веб-сервера Nginx, их парсинга (парсинга строки в JSON) и последующей безопасной передачи в Elasticsearch, был создан конфигурационный файл `/etc/logstash/conf.d/nginx.conf`:

```text
input {
  file {
    path => "/var/log/nginx/access.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}

filter {
  # Использование встроенного Grok-паттерна для разбора Combined Log Format веб-сервера Nginx
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
  # Перезапись системного таймстампа датой и временем реального совершения события
  date {
    match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts => ["https://localhost:9200"]
    ssl => true
    ssl_certificate_verification => false
    user => "elastic"
    password => "Kz4ym_Xoj8OhHhQv-Hxr"
    index => "logstash-nginx-%{+YYYY.MM.dd}"
  }
}
```

**4. Инициализация конвейера и генерация тестового трафика:**
```bash
# Запуск конвейера Logstash
sudo systemctl start logstash

# Искусственная генерация логов (имитация 10 последовательных запросов к веб-серверу)
for i in {1..10}; do curl -I http://localhost/; sleep 0.5; done
```

**5. Проверка поступления данных и визуализация в Kibana:**
Успешное создание индекса верифицировано в Dev Tools запросом `GET _cat/indices?v`. Для отображения структурированных логов на вкладке `Discover` в Kibana было создано представление данных (**Data View**) со следующими параметрами:
* **Name:** `Логи Nginx`
* **Index pattern:** `logstash-nginx-*`
* **Timestamp field:** `@timestamp`

Сырые текстовые строки лога были успешно декомпозированы на атомарные аналитические поля: `clientip`, `verb`, `request`, `http.response.status_code` (`response`), `bytes`.

**Скриншот интерфейса Kibana с логами Nginx, отправленными через Logstash:**

![Логи Nginx через Logstash](./img/3.jpg)



</details>

------
------


<details>
<summary><b>Задание 4. Filebeat.</b></summary>

- Установите и запустите Filebeat. Переключите поставку логов Nginx с Logstash на Filebeat. 

*Приведите скриншот интерфейса Kibana, на котором видны логи Nginx, которые были отправлены через Filebeat.*

### ОТВЕТ:

**1. Отключение предыдущего конвейера и установка Filebeat:**
```bash
sudo systemctl stop logstash
sudo apt update && sudo apt install filebeat -y
```

**2. Конфигурация защищенного агента доставки:**
Вся конфигурация сбора и прямой отправки данных настроена в файле `/etc/filebeat/filebeat.yml`. Для обхода проблем парсинга YAML-отступов на стороне агента, авторизация интегрирована методом HTTP Basic Auth напрямую в строку адреса хоста:

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log

output.elasticsearch:
  hosts: ["https://elastic:Kz4ym_Xoj8OhHhQv-Hxr@localhost:9200"]
  ssl.verification_mode: "none"
  index: "filebeat-nginx-%{[agent.version]}-%{+yyyy.MM.dd}"

setup.template.name: "filebeat"
setup.template.pattern: "filebeat-*"
setup.ilm.enabled: false
```

**3. Запуск агента и генерация новых событий:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable filebeat
sudo systemctl start filebeat

# Искусственная генерация свежих логов для проверки доставки через Filebeat
for i in {1..5}; do curl -I http://localhost/; done
```

Регистрация данных в Elasticsearch успешно подтверждена диагностической командой проверки выходного сетевого буфера `filebeat test output` со статусом `talk to server... OK`.

**Скриншот интерфейса Kibana с логами Nginx от Filebeat:**
![Логи Nginx через Filebeat](./img/4.jpg)




</details>

-----
-----


<details>
<summary><b>Задание 5*. Работа с числами</b></summary>

- Запишите в Redis ключ key5 со значением типа "int" равным числу 5. Увеличьте его на 5, чтобы в итоге в значении лежало число 10.  

*Приведите скриншот, где будут проделаны все операции и будет видно, что значение key5 стало равно 10.*

### ОТВЕТ:
 
- Решение задачи атомарного изменения целочисленного значения счетчика с 5 до 10:

```text

$ redis-cli
# Запись ключа со значением 5
127.0.0.1:6379> SET key5 5
OK

# Увеличение значения счетчика на 5 единиц
127.0.0.1:6379> INCRBY key5 5
(integer) 10

# Проверка итогового значения
127.0.0.1:6379> GET key5
"10"
```
**Скриншот операций записи и чтения в Redis:**
![Операции в Redis CLI](./img/4.jpg)




</details>
