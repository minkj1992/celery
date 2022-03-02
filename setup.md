# Setup

`Celery`를 django에서 사용하기 위해 꼭 필요한 작업들을 정리합니다.

### 1. 의존성

```
pip install -U Celery
```

만약 celery proj에 포함된 기타 의존성들을 가져오고 싶다면 아래 명령어를 사용하면 됩니다. ([Celery](https://github.com/celery) 프로젝트 리스트)

```
pip install "celery[gevent,librabbitmq, redis, auth]"
```

### 2. django setup
> [docs](https://docs.celeryproject.org/en/stable/django/first-steps-with-django.html) 참조

```
├── [프로젝트-루트]
│   ├── Dockerfile
│   ├── apps
│   │   ├── __init__.py
│   │   ├── core
│   │   └── fantem
│   ├── [프로젝트앱]  --> setting을 포함한 디렉토리 명을 의미합니다.
│   │   ├── __init__.py  --> [2]
│   │   ├── __pycache__
│   │   ├── celery.py  -> [1]
│   │   ├── settings  -> [3]
│   │   ├── urls.py
│   │   ├── vault.py
│   │   └── wsgi.py
```

#### 2.1. [프로젝트명]/celery.py

```python
# http://docs.celeryproject.org/en/latest/django/first-steps-with-django.html
from __future__ import absolute_import, unicode_literals
import os
from celery import Celery
import logging

from django.conf import settings

logger = logging.getLogger(__name__)

# 장고에 root app name으로 셀러리 앱 등록
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'con_item.settings.base')

app = Celery(프로젝트앱)

# - namespace='CELERY' means all celery-related configuration keys
#   should have a `CELERY_` prefix.
app.config_from_object('django.conf:settings', namespace='CELERY')

# Load task modules from all registered Django app configs.
app.autodiscover_tasks(lambda: settings.INSTALLED_APPS)


"""
# 셀러리 정상 작동 확인용 코드 (celery-beat)
app.conf.beat_schedule = {
    'display_time-100-seconds': {
        'task': 'fantem.tasks.display_time',
        'schedule': 100.0
    },
    'sentry_health_check': {
        'task': 'fantem.tasks.sentry_health_check',
        'schedule': 100.0
    },
}

@app.task(bind=True)
def debug_task(self):
    logger.info('Request: {0!r}'.format(self.request))
"""
```

- `os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'con_item.settings.base')`에서 `con_item.settings.base`는 아래의 base.py 세팅 파일을 의미합니다.

```
├── settings
│   └── base.py
```

#### 2.2. [프로젝트앱명]/__init__.py

장고 실행 시점에, 앞에서 생성해준 셀러리 앱을 등록해줍니다.

```python
from __future__ import absolute_import, unicode_literals

# 장고가 시작 시점에 Celery 앱이 자동 로드
from .celery import app as celery_app

__all__ = ['celery_app']
```

#### 2.3. settings.py

1. 로컬 환경

```python
celery_broker= {'host': 'localhost', 'password': 'admin', 'port': '5672', 'user': 'admin'}
BROKER_URL = 'memory://'
BROKER_BACKEND = 'memory'
CELERY_TASK_ALWAYS_EAGER = True
CELERY_TASK_EAGER_PROPAGATES = True

RABBITMQ_URL = "amqp://{user}:{password}@{host}:{port}".format(
    user=celery_broker['user'],
    password=celery_broker['password'],
    host=celery_broker['host'],
    port=celery_broker['port'],
)
```

2. 스테이징 환경

vault 세팅이 되었다고 가정합니다.

```python
VAULT_KMS_TOKEN = get_vault_token(VAULT_KMS)
VAULT = read_vault_secrets(VAULT_KMS, VAULT_KMS_TOKEN)

## CELERY
# https://docs.celeryproject.org/en/stable/userguide/configuration.html
BROKER_HEARTBEAT = os.getenv('BROKER_HEARTBEAT', 120) # health 체크 용도
CELERY_TASK_DEFAULT_QUEUE = os.getenv('CELERY_TASK_DEFAULT_QUEUE', 'con_item_local') # 큐 이름
# CELERY_DEFAULT_QUEUE (x), https://stackoverflow.com/a/55362566

CELERY_BROKER_URL = RABBITMQ_URL
CELERY_RESULT_BACKEND = 'rpc://'
CELERY_RESULT_SERIALIZER = 'json' # type 지정
CELERY_ACCEPT_CONTENT = ['application/json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_TIMEZONE = TIME_ZONE # TIME_ZONE = 'Asia/Seoul'

## FOR HEALTH CHECK
BROKER_URL = CELERY_BROKER_URL # 만약 헬스체크를 사용한다면, 추가 url
```


만약 sentry 로깅에 추가하고 싶다면 아래를 추가 해주시면 됩니다.


- settings.py
```python
import sentry_sdk
from sentry_sdk.integrations.celery import CeleryIntegration

...


    sentry_sdk.init(
        dsn=os.getenv('SENTRY_DSN'),
        integrations=[DjangoIntegration(), CeleryIntegration()],
        traces_sample_rate=1.0,  # We recommend lowering this in production
        send_default_pii=True,
        before_send=before_send,
    )
```

### 3. task 생성

장고 앱 부분에 셀러리 용 task 파일을 생성해줍니다.


- task 생성
```python
@shared_task(
    base=ExceptionHandlerTask,
    autoretry_for=(Exception,),
    retry_kwargs={'max_retries': 3},
    retry_backoff=True,
    default_retry_delay=3 * 60)
def example_task(data):
    response = api요청()

    result = {
        'kakao_account_id': data['card_data'].get('owner_kakao_account_id'),
        'status_code': response.status_code,
        'data': response.json(),
    }

    return {'vertical_image_url': data['vertical_image_url'], 'response_data': result}
```

- task 호출
  - 용도에 맞게, chain, chord, single task call 해주시면 됩니다.

```python
data = example_task.delay(something)
```



### 4. 실행

```
$ celery -A con_item worker -l info -c 2 --max-tasks-per-child=4096
```

- A: app
- l: log level
- -c: concurrency

