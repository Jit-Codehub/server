# 🚀 Django Celery Integration Guide




## 🛠️ Installation

```bash
pip install Django
pip install celery
pip install redis
```

---

## 📦 Basic Setup

1. **Create Django Project and Apps**

2. **Create `celery.py` inside the inner project folder**  
   Add your basic Celery config code.

3. **Configure Django Celery in `settings.py`**

4. **Create `tasks.py` inside your app**  
   Write your Celery tasks here.

5. **Trigger Celery tasks**  
   Call them from `views.py` or wherever needed.

6. **Start Celery Worker**

```bash
celery -A your_project_name worker -l info
```

---

## 🔧 Key Celery Functions

### `apply_async()`

Used to enqueue tasks for asynchronous execution.  
Returns an `AsyncResult` object to track task status and result.

#### 🧾 Syntax

```python
my_task.apply_async(
    args=["arg1", "arg2"],
    kwargs={"keyword_arg": "value"},
    countdown=10,
    expires=60
)
```

- `args`: Positional arguments
- `kwargs`: Keyword arguments
- `countdown`: Delay in seconds
- `expires`: Expiry time in seconds

---

### ✅ Example: Using `apply_async`

**`tasks.py`**

```python
from time import sleep
from celery import shared_task

@shared_task
def add(x, y, author=None):
    sleep(10)
    return x + y
```

**`views.py`**

```python
from .tasks import add
from django.shortcuts import render

def my_view(request):
    response = add.apply_async(args=[10, 20], kwargs={"author": "Geeky"})
    return render(request, "home.html", {"result": response.id})
```

---

### ⚡ `delay()`

A shorthand for `apply_async()` – convenient for simple use cases.

#### 🧾 Syntax

```python
my_task.delay(arg1, arg2, keyword_arg="value")
```

---

### ✅ Example: Using `delay()`

**`tasks.py`**

```python
@shared_task
def add(x, y, author=None):
    sleep(10)
    return x + y
```

**`views.py`**

```python
def my_view(request):
    response = add.delay(10, 20, author="Geeky")
```

---

## ⚖️ `apply_async()` vs `delay()`

| Feature         | `apply_async()`                          | `delay()`                               |
|----------------|------------------------------------------|------------------------------------------|
| Control        | Full (countdown, expires, etc.)           | Minimal                                  |
| Use Case       | Complex or delayed tasks                  | Simple enqueuing                         |
| Example        | `add.apply_async(..., countdown=10)`      | `add.delay(10, 20)`                      |

---

## 🔍 `AsyncResult` Attributes

- `result`: Final return value (if completed)
- `state`: Current state (e.g., `PENDING`, `SUCCESS`, etc.)
- `status`: Alias for state
- `task_id` / `id`: Unique identifier for task
- `expires`: Expiration timestamp (UTC)

---

## 🧪 `AsyncResult` Methods

- `ready()`: ✅ True if task is done
- `successful()`: ✅ True if completed without error
- `failed()`: ❌ True if task raised an exception
- `get(timeout=n)`: Blocks and returns result or raises exception

---

## 📊 Checking Task Status

```python
response = add.apply_async(args=[10, 20])

response.id              # Task ID
response.state           # Status: PENDING, STARTED, SUCCESS...
response.ready()         # True if finished
response.successful()    # True if no exception
response.get(timeout=20) # Blocks until result is ready
```

---

## ✅ Example: Task Status View

**`views.py`**

```python
from django.http import JsonResponse
from .tasks import add

def task_status(request, task_id):
    result = add.AsyncResult(task_id)
    return JsonResponse({
        "state": result.state,
        "result": result.result if result.ready() else None,
    })
```

---

Happy Asynchronous Coding! ⚙️✨
