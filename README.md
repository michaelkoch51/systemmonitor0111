# Домашнее задание к занятию "13. Системы мониторинга"

### 1. Минимальный набор метрик для платформы вычислений
Для платформы, которая делает расчеты (нагружает CPU), отдает текстовые отчеты (пишет на диск) и работает по HTTP, минимальный набор метрик будет следующим:
*   **CPU:** `usage_system`, `usage_user`, `load1` (средняя нагрузка). *Почему:* Вычисления загружают процессор, нужно понимать, когда мощностей перестает хватать.
*   **Disk (Диск):** `disk_used_percent` (свободное место), `diskio_write_bytes` (скорость записи), `inodes_free`. *Почему:* Отчеты сохраняются на диск. Если закончится место или inodes, платформа перестанет работать. Высокий I/O wait может затормозить всё приложение.
*   **RAM (Память):** `mem_used_percent`. *Почему:* Вычисления могут потреблять много оперативной памяти, нужно отслеживать риск OOM (Out Of Memory).
*   **HTTP/Network:** `http_response_time`, `http_2xx_rate` (доля успешных ответов). *Почему:* Это показывает доступность сервиса для клиентов.

### 2. Как объяснить метрики менеджеру и показать качество обслуживания (SLI/SLO)
Менеджеру не нужны технические метрики (RAM/CPUla). Ему нужно предложить концепцию **SLI/SLO**:
*   **SLI (Service Level Indicator):** Показатель уровня сервиса. Например, "Доля успешных HTTP-ответов (2xx)" или "Время ответа 95% запросов менее 2 секунд".
*   **SLO (Service Level Objective):** Внутренняя цель. Например, "Обеспечивать доступность 99.9% в месяц".
*   **Что предложить:** Создать дашборд бизнес-метрик (Golden Signals), где будет отображаться:
    1. Доступность сервиса (Uptime).
    2. Процент ошибок (Error Rate).
    3. Скорость ответа (Latency).
    4. Количество обработанных заявок (Throughput).

### 3. Сбор логов без финансирования
Если нет денег на ELK/EFK (Elasticsearch, Logstash, Kibana), можно использовать бесплатные и легковесные решения:
1.  **Grafana Loki:** Очень легковесная система сбора логов, которая отлично работает с Prometheus и не требует огромных ресурсов для индексации.
2.  **Sentry (Self-hosted или Free Tier):** Идеально для отлова именно **ошибок приложений** (исключений), а не всех логов подряд.
3.  **Логирование в stdout + сбор через Docker/Systemd:** Настроить отправку логов через `journald` и перенаправлять их в бесплатный облачный сервис (например, Papertrail).

### 4. Ошибка в формуле расчета SLA
Формула `summ_2xx_requests / summ_all_requests` показывает долю успешных ответов, но если она ниже 70%, а 4xx и 5xx нет, значит, остальные 30% запросов **вообще не дошли до приложения или не вернули HTTP-код**. 
**Где ошибка:** Вы не учитываете коды **3xx (редиректы)**. Кроме того, могут быть **таймауты (timeouts)** или обрывы соединений (например, код 499 в Nginx «Client Closed Request»). Если клиент отвалился по таймауту, сервер не отдал ни 2xx, ни 4xx, ни 5xx. Формулу нужно дополнить: учитывать коды 3xx как успешные и мониторить количество оборванных соединений/таймаутов.

### 5. Плюсы и минусы Pull и Push систем мониторинга
*   **Pull (Вытягивание):** Сервер мониторинга сам опрашивает агенты (например, Prometheus).
    *   *Плюсы:* Легко проверять статус (если не ответил — узел упал), централизованная конфигурация, проще контролировать нагрузку на сеть.
    *   *Минусы:* Сложно мониторить короткоживущие задачи (batch/cron), проблемы с файрволами (сервер должен иметь доступ к клиентам).
*   **Push (Отправка):** Агенты сами отправляют метрики на сервер (например, Zabbix Agent active, TICK).
    *   *Плюсы:* Отлично подходит для короткоживущих задач и динамических сред (Kubernetes), проще преодолевать NAT/файрволы.
    *   *Минусы:* Сервер не знает, упал клиент или просто перестал слать данные (нужны "heartbeat" алерты), нужно настраивать каждого агента.

### 6. Классификация систем (Push / Pull / Гибрид)
*   **Prometheus** — **Pull** (хотя есть Pushgateway для исключений).
*   **TICK (Telegraf, InfluxDB, Chronograf, Kapacitor)** — **Push** (Telegraf сам отправляет метрики в InfluxDB).
*   **Zabbix** — **Гибрид** (может работать в пассивном режиме - Pull, и активном - Push).
*   **VictoriaMetrics** — **Гибрид** (поддерживает PromQL pull и push-протоколы).
*   **Nagios** — **Pull** (классический опрос агентов).

---

## Практическая часть (TICK-стек)

Для запуска стека я использовал `docker-compose`.

### Скриншот 1: Веб-интерфейс Chronograf
[https://github.com/user-attachments/assets/e77faf2a-6c16-45da-a0c9-f8cc612a223c]

### Скриншот 2: График утилизации CPU
[https://github.com/user-attachments/assets/eb34202c-f8f8-487c-ae9d-e5920bff79c9]

### Скриншот 3: Метрики Docker в Telegraf
[https://github.com/user-attachments/assets/48043e76-d8e3-452e-80cc-fb3f619278b4]

---

## Дополнительное задание (со звездочкой*) - Python-скрипт

**а) Код Python 3 скрипта (`awesome_monitoring.py`):**
Скрипт собирает 4 метрики из `/proc` (CPU Load, Memory Free, Uptime, Processes count) и пишет их в JSON.

```python
#!/usr/bin/env python3
import json
import time
import datetime
import os

def get_metrics():
    # 1. Load Average (из /proc/loadavg)
    with open('/proc/loadavg', 'r') as f:
        load_avg = float(f.read().split()[0])
    
    # 2. Свободная память в КБ (из /proc/meminfo)
    with open('/proc/meminfo', 'r') as f:
        mem_free = 0
        for line in f:
            if line.startswith('MemFree'):
                mem_free = int(line.split()[1])
                break
                
    # 3. Uptime в секундах (из /proc/uptime)
    with open('/proc/uptime', 'r') as f:
        uptime = float(f.read().split()[0])
        
    # 4. Количество запущенных процессов (считаем директории в /proc, названия которых - цифры)
    proc_count = sum(1 for d in os.listdir('/proc') if d.isdigit())

    return {
        "timestamp": int(time.time()),
        "cpu_load_1m": load_avg,
        "mem_free_kb": mem_free,
        "uptime_sec": uptime,
        "processes_count": proc_count
    }

if __name__ == "__main__":
    metrics = get_metrics()
    date_str = datetime.datetime.now().strftime("%y-%m-%d")
    log_filename = f"/var/log/{date_str}-awesome-monitoring.log"
    
    with open(log_filename, 'a') as f:
        f.write(json.dumps(metrics) + '\n')

б) Конфигурация cron-расписания:
Чтобы скрипт запускался каждую минуту, добавьте в crontab -e следующую строку:

* * * * * /usr/bin/python3 /path/to/awesome_monitoring.py

в) Пример верно сформированного лога YY-MM-DD-awesome-monitoring.log (5 записей):

{"timestamp": 1698835200, "cpu_load_1m": 0.12, "mem_free_kb": 845123, "uptime_sec": 14500.5, "processes_count": 142}
{"timestamp": 1698835260, "cpu_load_1m": 0.15, "mem_free_kb": 842011, "uptime_sec": 14560.5, "processes_count": 143}
{"timestamp": 1698835320, "cpu_load_1m": 0.10, "mem_free_kb": 850300, "uptime_sec": 14620.5, "processes_count": 142}
{"timestamp": 1698835380, "cpu_load_1m": 0.18, "mem_free_kb": 830123, "uptime_sec": 14680.5, "processes_count": 145}
{"timestamp": 1698835440, "cpu_load_1m": 0.14, "mem_free_kb": 838900, "uptime_sec": 14740.5, "processes_count": 142}
