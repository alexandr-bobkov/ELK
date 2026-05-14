
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
<summary><b>Задание 2. Memcached</b></summary>

- Установите и запустите memcached.

*Приведите скриншот systemctl status memcached, где будет видно, что memcached запущен.*

------

### ОТВЕТ:
Установка и запуск сервиса в Debian выполнены с помощью команд:

```bash
sudo apt update && sudo apt install memcached -y
sudo systemctl start memcached
sudo systemctl enable memcached
sudo systemctl status memcached
```
**Скриншот статуса службы memcached:**
![Статус Memcached](./img/1.jpg)




</details>

-------
-------

<details>
<summary><b>Задание 3. Удаление по TTL в Memcached</b></summary>

- Запишите в memcached несколько ключей с любыми именами и значениями, для которых выставлен TTL 5. 

*Приведите скриншот, на котором видно, что спустя 5 секунд ключи удалились из базы.*
-------

### ОТВЕТ:
Для исключения ошибок интерпретации скрытых символов терминала (`CLIENT_ERROR bad data chunk`), запись и проверка удаления по TTL (5 секунд) выполнены атомарной отправкой корректных CRLF-строк через `printf`:

```bash
printf "set key1 0 5 4\r\nval1\r\nget key1\r\n" | nc localhost 11211 && sleep 6 && printf "get key1\r\n" | nc localhost 11211
```

#### Подробный разбор работы команды:

1. **Первый блок (`printf "..." | nc ...`)**:
   * Команда `printf` формирует точные текстовые строки со служебными символами переноса `\r\n` (CRLF), которые строго требуются протоколом Memcached.
   * Строка `set key1 0 5 4` сообщает серверу: создать ключ `key1` с временем жизни **TTL = 5 секунд** и размером значения **4 байта**.
   * Строка `val1` передает само значение (**Value**).
   * Команда `get key1` отправляется следом в этом же пакете, чтобы мгновенно проверить запись.
   * Конвейер `|` перенаправляет этот текст утилите Netcat (`nc`), которая отправляет данные на порт Memcached (`11211`). Сервер успешно сохраняет ключ и сразу же возвращает его значение.

2. **Оператор соединения (`&&`)**:
   * Связывает команды в цепочку. Вторая часть запустится только после успешного завершения первой.

3. **Второй блок (`sleep 6`)**:
   * Терминал засыпает на **6 секунд**. За это время внутри Memcached истекают отведенные 5 секунд жизни ключа (`TTL`), и данные автоматически помечаются как удаленные.

4. **Третий блок (`printf "get key1\r\n" | nc ...`)**:
   * После пробуждения терминала отправляется повторный запрос `get key1`. 
   * Так как время жизни ключа вышло, Memcached окончательно стирает его из оперативной памяти и возвращает маркер `END` (данные отсутствуют).

**Результат выполнения команды в терминале:**
```text
STORED
VALUE key1 0 4
val1
END
END
```

**Скриншот фиксации удаления ключей по TTL:**
![Удаление ключей по TTL](./img/2.jpg)






</details>

------
------


<details>
<summary><b>Задание 4. Запись данных в Redis</b></summary>

### ОТВЕТ:

**Установка и запуск Redis-сервера:**
```bash
sudo apt update && sudo apt install redis-server -y
sudo systemctl start redis-server
sudo systemctl enable redis-server
sudo systemctl status redis-server
```

После успешного запуска выполнена множественная запись тестовых ключей через команду `MSET` и их последующее извлечение из базы с помощью интерфейса командной строки `redis-cli`:

```text
$ redis-cli

127.0.0.1:6379> MSET user:1 "Alex" user:2 "Ivan" user:3 "Olga"
OK

127.0.0.1:6379> KEYS *
1) "user:2"
2) "user:3"
3) "user:1"

127.0.0.1:6379> MGET user:1 user:2 user:3
1) "Alex"
2) "Ivan"
3) "Olga"
```

**Скриншот операций записи и чтения в Redis:**
![Операции в Redis CLI](./img/3.jpg)


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
