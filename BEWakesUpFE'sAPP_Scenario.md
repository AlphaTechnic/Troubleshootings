## 환경

![https://user-images.githubusercontent.com/58129950/155041694-57af051c-bd65-4afc-b232-b930fa7039f7.png](https://user-images.githubusercontent.com/58129950/155041694-57af051c-bd65-4afc-b232-b930fa7039f7.png)

- GCP setting
  - region : 오사카
  - 삭제 보호 : 설정됨
  - 부팅디스크
    - 운영체제 : ubuntu
  - 방화벽 : HTTP 트래픽
  - 고정 IP
  - port opened
    - 22, 80, 443, 8000, 8080, 8888, 9000
  - https 인증
- 도커 version 3.7
  - django-gunicorn 컨테이너 (custom image 기반)
  - nginx 1.19.5 컨테이너
  - mariadb 10.5 컨테이너





### FCM 을 활용하는 목적

- **BE에서 주는 액션으로 FE에서 GPS 로그를 로컬 DB에 저장하기**
    - FCM 활용 전에는 FE측에서 일정 범위를 벗어나는 액션을 trigger로 GPS 로그를 찍는다.
    - 따라서, 정작 Stay Point인 위치에 GPS 로그를 찍지 못하는 문제가 있다.
    - 일정한 시간 주기(이를테면 30분)를 trigger로 BE 측에서 앱을 깨울 수 있고, 이를 trigger로 GPS 로그를 찍을 수 있다면, 더 정확하게 interval을 만들 수 있다.
    - 특히 오래(30분 이상) 머문(Stay) 공간의 경우, 그 장소를 정밀하게 detection 할 수 있을 것으로 기대할 수 있다.
- **통계 집계 전에 GPS 로그들 FE → BE로 넘겨주기**
    - BE에서 배지 지급을 위해 일주일에 한번 사용자들의 일상에 대한 통계를 낸다.
    - 그런데, 사용자가 앱에 접속하지 않으면 BE에서 사용자들의 일주일간 쌓인 일상 정보(어떤 활동에 어느 정도의 시간을 썼는지)를 알 수가 없다.
    - FE → BE로 일주일간의 정보를 넘겨주면, 이에 대한 집계를 할 수가 있다.
    - 이를테면, 일요일 24시10분에 정보를 넘기고 24시30분에 집계를 하는 방식



- **⇒ FCM은 BE에서 FE의 앱을 깨우기 위한 시도이다.**



- `FCM.py`

```python

import datetime

from firebase_admin import messaging

def send_to_firebase_cloud_messaging(registration_token, is_silent, msg_type, msg_title, msg_body):
    # See documentation on defining a message payload.
    apns = messaging.APNSConfig(
        payload=messaging.APNSPayload(
            aps=messaging.Aps(content_available=True, thread_id=msg_type)
        )
    )

    if is_silent:
        message = messaging.Message(
            apns=apns,
            token=registration_token,
        )
    else:
        message = messaging.Message(
            notification=messaging.Notification(
                title=msg_title,
                body=msg_body,
            ),
            apns=apns,
            token=registration_token,
        )
    response = messaging.send(message)
    # Response is a message ID string.
    print('Successfully sent message:', response)

def send_to_firebase_cloud_group_messaging(registration_tokens, is_silent, msg_type, msg_title, msg_body):
    # See documentation on defining a message payload.

    apns = messaging.APNSConfig(
        payload=messaging.APNSPayload(
            aps=messaging.Aps(content_available=True, thread_id=msg_type)
        )
    )

    if is_silent:
        message = messaging.MulticastMessage(
            apns=apns,
            tokens=registration_tokens,
        )
    else:
        message = messaging.MulticastMessage(
            notification=messaging.Notification(
                title=msg_title,
                body=msg_body,
            ),
            apns=apns,
            tokens=registration_tokens,
        )
    response = messaging.send_multicast(message)
    print(f"{datetime.datetime.now()} 에 메세지 전송 시도!! 결과는..?")
    print(f"{response.success_count} messages were sent successfully.")

def func_to_schedule(appuser_tokens, is_silent, msg_type, msg_title, msg_body):
    try:
        send_to_firebase_cloud_group_messaging(appuser_tokens, is_silent, msg_type, msg_title, msg_body)
    except:
        print("There are no tokens")
```



### 1. saveLocation

- thread_id
    - saveLocation
- 목적
    - 앱이 깨워져 로컬 디바이스에 GPSLog 데이터가 찍혀 생성되도록 하기 위함
- 방법
    - `silent push`
- noti 시각
    - 매일 30분에 한 번씩
- 메세지 내용
    - 없음
    
- `saveLocationNoti.py`

```python
from notificationapp.FCM import *

import datetime

# from apscheduler.schedulers.blocking import BlockingScheduler
from apscheduler.schedulers.background import BackgroundScheduler

from accountapp.models import AppUser

def get_nearest_half_hour():
    now_minute = datetime.datetime.now().minute
    now_second = datetime.datetime.now().second
    delta = (30 - now_minute) % 30
    return datetime.datetime.now() + datetime.timedelta(minutes=delta) - datetime.timedelta(seconds=now_second)

def start_saveLocation():
    app_users = AppUser.objects.exclude(fcmToken="abc").exclude(fcmToken="")
    appuser_tokens = [app_user.fcmToken for app_user in app_users]

    scheduler = BackgroundScheduler(timezone="Asia/Seoul", job_defaults={'max_instances': 1})

    scheduler.add_job(func_to_schedule, 'cron', minute="*/30",
                      args=[appuser_tokens, True, "saveLocation", "saveLocation 통신", "saveLocation 통신"])

    scheduler.start()
    print("after scheduler.start() !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!")
```



### 2. pathDaily

- thread_id
    - pathDaily
- 목적
    - 사용자에게 일주일 동안의 기록 확인을 유도
    - `path/daily` POST 통신을 하여, FE의 모든 GPS 로그 기록을 BE로 넘겨준다.
- 방법
    - `remote push`
- noti 시각
    - 일->월 넘어가는 00:10:00
- 메세지 내용
    - 일주일 동안의 기록을 확인해보세요!

- `pathDailyNoti.py`

```python
from notificationapp.FCM import *

# from apscheduler.schedulers.blocking import BlockingScheduler
from apscheduler.schedulers.background import BackgroundScheduler

from accountapp.models import AppUser

def start_path_daily_noti():
    app_users = AppUser.objects.exclude(fcmToken="abc").exclude(fcmToken="")
    appuser_tokens = [app_user.fcmToken for app_user in app_users]

    scheduler = BackgroundScheduler(timezone="Asia/Seoul", job_defaults={"max_instance": 1})
    scheduler.add_job(func_to_schedule, 'cron', day_of_week='mon', hour=0, minute=10,
                      args=[appuser_tokens, , "pathDaily", "From Wolley 🗓", "일주일 동안의 기록을 확인해보세요!"])

    scheduler.start()
```