# 🚀 Celery: Distributed Task Queue System

## 📌 Overview

**Celery** is a task queue system that allows you to execute work outside the Python web application’s HTTP request-response cycle.

---

## 🌟 Key Features

* 🗓 **Task Scheduling:** Schedule tasks to run periodically (daily reports, backups, etc.).
* ⚡ **Concurrency:** Execute multiple tasks simultaneously with multiprocessing, threading, or greenlets.
* 📈 **Scalability:** Easily adjust the number of worker processes according to your load.
* 🔗 **Robust Integration:** Compatible with popular message brokers (RabbitMQ, Redis, Amazon SQS) and result storage backends.
* ⏰ **Celery Beat:** An optional scheduler that periodically sends tasks to the Celery worker queue.

---

## 🔧 Core Components

### 📦 Tasks

Define Python functions to execute asynchronously:

```python
from celery import shared_task

@shared_task
def add(x, y):
    return x + y
```

Queue tasks with:

```python
add.delay(2, 3)
```

### 📡 Message Broker

It acts as an intermediary between the celery client (the application that sends tasks) and the celery workers (the processes that execute tasks). The primary function of a message broker in Celery are Message Queuing, Task Distribution.

**Common brokers:**

* 🐰 **RabbitMQ:** Robust but slightly complex to configure.
* 🚀 **Redis:** Easy to set up and manage.
* ☁️ **Amazon SQS:** Fully managed AWS solution.

### 🛠 Workers

Worker processes or threads execute queued tasks:

Start a worker:

```bash
celery -A your_project.celery worker -l INFO
```

---

## 🏗 Execution Pools

Choose the right execution model for your tasks:

* 🚦 **Prefork (default):** Ideal for CPU-intensive tasks, leverages multiprocessing.
* 🎯 **Solo:** Single-threaded execution, useful for debugging and limited environments (e.g., Windows).
* 🧵 **Threads:** Suitable for I/O-bound tasks but limited by Python's GIL.
* 🍀 **Eventlet/Gevent:** Greenlets-based, highly efficient for concurrent I/O-bound tasks.

Examples:

**Prefork:**

```bash
celery -A your_project.celery worker --pool=prefork --concurrency=4 -l INFO
```

**Solo:**

```bash
celery -A your_project.celery worker --pool=solo -l INFO
```

**Threads:**

```bash
celery -A your_project.celery worker --pool=threads --concurrency=10 -l INFO
```

**Gevent:**

```bash
pip install gevent
celery -A your_project.celery worker --pool=gevent --concurrency=100 -l INFO
```

---

## ⏲ Celery Beat and Django-Celery-Results

### Celery Beat

Periodic task scheduler:

```bash
celery -A your_project.celery beat -l INFO
```

Define scheduled tasks:

```python
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    'daily-report': {
        'task': 'app.tasks.daily_report',
        'schedule': crontab(hour=7, minute=0),
    },
}
```

### Django-Celery-Results

Persist task results in Django's database:

```bash
pip install django-celery-results
```

Configure in `settings.py`:

```python
INSTALLED_APPS += ['django_celery_results']
CELERY_RESULT_BACKEND = 'django-db'
```

Run migrations:

```bash
python manage.py migrate django_celery_results
```

---

## ✅ Selecting the Right Pool

* 🔥 **CPU-intensive:** Use `prefork`.
* 🌊 **I/O-intensive:** Use `gevent` or `eventlet`.
* 🐞 **Debugging or Windows:** Use `solo`.

---

## 🛠 Step-by-Step Example Setup

1. **Install Redis (as a broker):**

```bash
sudo apt-get install redis-server
```

2. **Install Celery and Django integration:**

```bash
pip install celery django-celery-results
```

3. **Create `celery.py` in your Django project:**

```python
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'your_project.settings')
app = Celery('your_project')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
```

4. **Define a Celery task (`tasks.py`):**

```python
from celery import shared_task

@shared_task
def send_email(user_id):
    # email sending logic here
    pass
```

5. **Start Celery worker:**

```bash
celery -A your_project.celery worker -l INFO
```

6. **Enqueue tasks from Django views:**

```python
from app.tasks import send_email

def register(request):
    send_email.delay(user_id)
    return HttpResponse("Task queued!")
```

---

## 🏅 Recommended Best Practices

* 📊 **Monitoring:** Use Flower (`pip install flower`) for real-time monitoring.
* ⚠️ **Task failure handling:** Implement retries with exponential backoff.
* 📫 **Dedicated queues:** Route tasks efficiently based on type.
* 🔐 **Security:** Secure brokers and backends with proper credentials.

---

## 📖 Summary

Celery significantly enhances your Django or Python application by handling background tasks asynchronously, improving responsiveness, scalability, and maintainability.
