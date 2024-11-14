# **Мониторинг в операционной системе RedOS на основе связки Node exporter, Prometheus и Grafana в докере.**

<image src="./images/banner.jpg" alt="banner"> 

### **Задача:**

Необходимо развернуть мониторинг на линукс хосте под управлением RedOS с использованием Node exporter, Prometheus и Grafana в докере.

Cхема такая: node_exporter снимает метрики с хоста, prometheus забирает их из node_exporter каждые 5 секунд, grafana обращается к prometheus и рисует графики на основе метрик.

В grafana подключи Prometheus как источник данных (datasource).
Создай дашборд Base metrics с 3 панелями, которые будут демонстрировать нам ключевые аспекты состояния нашего сервера:
- отображение свободной памяти;
- наличие свободного дискового пространства, в процентном соотношении от общего объема;
- отображать субьективную нагрузку на процессор. Для базиса мы возьмем метрику load average за минуту — это LA, однако чтобы данные были обьективными, нам нужно эту метрику разделить на общее количество ядер, а затем умножить на 100.
- найди какие есть готовые дашборды и разберись как их устанавливать в графану, найди самый удачный на твой взгляд дашборд и установи.
- настрой ретеншен данных на 1 месяц.

### **Для решения задачи будут использоваться:**

- redos-MUROM-7.3.4-20231220.0-Everything-x86_64-DVD1
- node_exporter, version 1.8.2
- prometheus, version 2.55
- Grafana v11.3.0

### 1. Установка RedOS, Node exporter, Prometheus, Docker.

#### 1.1 Устанавливаем RedOS на виртуальную машину.

Устанавливаем по инструкции отсюда:

https://redos.red-soft.ru/base/redos-7_3/7_3-administation/7_3-virt-and-emul/7_3-virtualbox/7_3-virtualbox-os/?nocache=1731589652258

#### 1.2 Установка Node exporter и Prometheus.

Устанавливаем по инструкции отсюда:

https://redos.red-soft.ru/base/redos-7_3/7_3-administation/7_3-monitoring/7_3-prometheus-grafana/?nocache=1731589827625

#### 1.3 Установка Docker.

Устанавливаем по инструкции отсюда:

https://redos.red-soft.ru/base/redos-7_3/7_3-administation/7_3-containers/7_3-docker-install/?nocache=1731589892191

### 2. Установка Grafana в Docker контейнере.

#### 2.1 Установка с помощью консольной команды Docker.

Запускаем команду `docker run -d --network host --name=grafana \
--volume grafana-storage:/var/lib/grafana \
grafana/grafana-enterprise`

В этой команде:

- `docker run` — команда для запуска нового контейнера Docker на основе образа. В данно случае запускается контейнер на основе образа Grafana Enterprise.

- `-d` — это сокращение от "detached mode" (фоновый режим).
- `--network host` — параметр указывает, что контейнер должен использовать сетевую конфигурацию хоста. В этом режиме контейнер получает доступ к сети так же, как если бы он работал непосредственно на хосте. Режим не совсем безопасный, т.к. этот режим делает контейнер менее изолированным с точки зрения сети, но для тестов пойдёт.
- `--name=grafana` — этот параметр задает имя контейнера. Имя "grafana" поможет легко идентифицировать контейнер при выполнении других команд.
- `--volume grafana-storage:/var/lib/grafana` — этот флаг монтирует том `grafana-storage`
(который будет создан автоматически, если его не существует) к директории
`/var/lib/grafana` внутри контейнера. Это обеспечивает постоянное хранение данных Grafana, таких как настройки, информация о пользователях и сохраненные панели мониторинга (dashboards), даже если контейнер будет перезапущен. Без этого монтирования данные были бы потеряны при остановке или удалении контейнера.
- `grafana/grafana-enterprise` — это имя образа Docker, из которого будет запущен контейнер. В данном случае используется образ "grafana/grafana-enterprise" из Docker Hub.

#### 2.2 Установка с помощью Docker-Сompose.

Создаём файл `docker-compose.yml` со следующим содержимым:

```yml 
services:
  grafana:
    image: grafana/grafana-enterprise
    container_name: grafana_compose
    network_mode: host
    volumes:
      - grafana-storage:/var/lib/grafana
    restart: always

volumes:
  grafana-storage:
    external: true
```

и запускаем контейнер командой:

`docker-compose up -d`

Параметр `external: true` позволяет не создавать новый volume, а использовать уже имеющийся.

Таким образом мы можем запускать контейнеры с разными именами, используя одно и то же хранилище. У всех контейнеров на скриншоте ниже один и тот же volume.

<image src="./images/containers.jpg" alt="containers">

### 3. Настройки системы мониторинга по заданию.

#### 3.1 Prometheus должен забирать метрики из Node Exporter каждые 5 секунд.

Чтобы Prometheus забирал метрики с Node Exporter каждые 5 секунд, необходимо изменить конфигурацию Prometheus в файле `prometheus.yml`. Смотрим где он там лежит через `systemctl`:

<image src="./images/prometheus.yml.jpg" alt="prometheus.yml">

Заходим в `prometheus.yml` и меняем параметры `scrape_interval` - интервал запроса метрик и `scrape_timeout` - таймаут запроса метрик. Устанавливаем оба параметра 5 сек.

<image src="./images/prometheus_config.jpg" alt="prometheus_config">

#### 3.2 Настройка retention данных на 1 месяц.

По умолчанию `retention` установлен 15 дней. Чтобы поменять на другое значение необходимо добавить строку конфигурации `--storage.tsdb.retention.time` в файл `prometheus.service`.

Смотрим где он там лежит через `systemctl`:  

<image src="./images/prom_service.jpg" alt="prom_service">  
  
Согласно документации Prometheus отсюда https://prometheus.io/docs/prometheus/latest/storage/#operational-aspects 
нет возможности установить retention именно 1 месяц, поэтому заходим в `prometheus.service` и добавляем строку `--storage.tsdb.retention.time` со значением `30d`. Не забываем добавить backslash в конце предпоследней строки иначе параметр не сработает.  

<image src="./images/retention.jpg" alt="retention">

Перезагружаем Prometheus и смотрим применились ли настройки по адресу `http://localhost:9090/status` в браузере.  

<image src="./images/storage_retention.jpg" alt="storage_retention">

### 4. Установка и создание дашбордов в Grafana.

#### 4.1 Установка дашбордов.

Для примера установим один из самых популярных дашбордов в Grafana.
Заходим на `https://grafana.com/grafana/dashboards/` и выбираем дашборд `Node Exporter Full`. Скачиваем JSON файл. В Grafana нажимаем `import` и выбираем наш JSON. Получаем красивый дашборд за 20 секунд.

<image src="./images/dashboard_node.jpg" alt="dashboard_node">

#### 4.2 Создание дашбордов.

##### 4.2.1 Создание дашборда для отображения свободной памяти (RAM).

Для создания нового дашборда заходим слева в `Dashboards`, затем справа выбираем `New` ->`New dashboard`. Задаём название дашборда и описание. Далее создаём новую панель меню справа `Add` -> `Visualisation`. Выбираем справа сверху `Gauge` и добавляем параметры для отображения `Metrics browser`. Для вычисления свободной памяти в процентах вводим следующие параметры:

- `(node_memory_MemFree_bytes / node_memory_MemTotal_bytes ) * 100`

Добавляем заголовок панели и описание справа. Там же выставляем параметры для нужного отображения диаграммы. Получаем визуализацию, которая отображает количество свободной памяти (RAM) в процентах.

<image src="./images/ram_free.jpg" alt="ram_free">

##### 4.2.2 Создание дашборда, отображающего наличие свободного дискового пространства, в процентном соотношении от общего объема.

Создаём новую панель в том же дашборде по описанию из пункта 4.2.1. В качестве `Metrics browser` добавляем следующие параметры:

- `(node_filesystem_avail_bytes{device="/dev/mapper/ro_redos-root"} / node_filesystem_size_bytes{device="/dev/mapper/ro_redos-root"}) * 100`

Получаем панель, отображающую свободное место на диске в процентах.

<image src="./images/disk_free.jpg" alt="disk_free">

##### 4.2.3 Создание дашборда отображающего субьективную нагрузку на процессор в процентах.

Создаём новую панель в том же дашборде по описанию из пункта 4.2.1. В качестве `Metrics browser` добавляем следующие параметры:

- `avg(node_load1) / count(count(node_cpu_seconds_total) by (cpu)) * 100`

Получаем панель, отображающую  субьективную нагрузку на процессор в процентах.

<image src="./images/cpu_load.jpg" alt="cpu_load">

### Заключение
В
В итоге было установлено и настроено программное обеспечение согласно заданию и был создан дашборд с тремя панелями, которые демонстрируют нам ключевые аспекты состояния нашего сервера

- отображение свободной памяти
- наличие свободного дискового пространства, в процентном соотношении от общего объема
- отображать субьективную нагрузку на процессор

<image src="./images/dash.jpg" alt="dash">