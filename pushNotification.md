```python
#%% md
# 패키지 설치 및 serviceAccountKey.json 파일 업로드
#%%
!pip install firebase-admin
!pip install APscheduler
#%%
from google.colab import files

print("⬇ serviceAccountKey.json 파일을 업로드 해주세요 ⬇\n")
uploaded = files.upload()

print("\n업로드 되었습니다!!")
#%%
import os

print("왼쪽 구석에 폴더모양 아이콘을 클릭하면, 작업 중인 디렉토리에 접근할 수 있습니다.\n")
print(f"현재 작업 중인 위치입니다 : {os.getcwd()}")
print(f"현재 작업 중인 경로의 파일들 입니다 : {os.listdir()}")
#%%
# firebase 통신 환경 구축

import firebase_admin
from firebase_admin import credentials


def init_app():
    # firebase 관련
    cred_path = os.path.join(f"{os.getcwd()}", "serviceAccountKey.json")
    cred = credentials.Certificate(cred_path)
    firebase_admin.initialize_app(cred)

init_app()
#%% md
# 함수 정의
#%%
# push notification 한 번 보내기 - 함수 정의

import firebase_admin
from firebase_admin import credentials, messaging
  

def send_to_firebase_cloud_messaging(registration_token, is_silent):
    # See documentation on defining a message payload.
    apns = messaging.APNSConfig(
        payload=messaging.APNSPayload(
            aps=messaging.Aps(content_available=True)  # individual needs in the background notification part
        )
    )

    if is_silent:
      message = messaging.Message(
          # notification=messaging.Notification(
          #     title='(test) title 입니다.',
          #     body='(test) u r so pretty girl~',
          # ),
          apns=apns,
          token=registration_token,
      )
    else:
      message = messaging.Message(
          notification=messaging.Notification(
              title='(test) title 입니다.',
              body='(test) u r so pretty girl~',
          ),
          apns=apns,
          token=registration_token,
      )
    response = messaging.send(message)

    # Response is a message ID string.
    print('Successfully sent message:', response)

  
def send_to_firebase_cloud_group_messaging(registration_tokens, is_silent):
    # See documentation on defining a message payload.
    apns = messaging.APNSConfig(
        payload=messaging.APNSPayload(
            aps=messaging.Aps(content_available=True)  # individual needs in the background notification part
        )
    )

    if is_silent:
      message = messaging.MulticastMessage(
          # notification=messaging.Notification(
          #     title='(test) title 입니다.',
          #     body='(test) u r so pretty girl~',
          # ),
          apns=apns,
          tokens=registration_tokens,
      )
    else:
      message = messaging.MulticastMessage(
          notification=messaging.Notification(
              title='(test) title 입니다.',
              body='(test) u r so pretty girl~',
          ),
          apns=apns,
          tokens=registration_tokens,
      )
    response = messaging.send_multicast(message)
    print(f"{response.success_count} messages were sent successfully.")

#%% md
# 아래의 셀을 실행하면서 test하면 됩니다.
#%%
import datetime

token_ella = "cSH6WSu28kO3pz9x8HT7Cy:APA91bFctXDl90lqMFom8LOf3zu660ZRFGc9_CvhxN1Yx6RYX92aNaZQ6cj0JRrO-EYpBhridcrNIcgGY1e8mLuohagwH4dYZfbmtxM2BkVxN93HOBCo1PuPzY2URgTTncIJ4GbuP15u"
#%% md
### push notificaton 한번만 보내기
#%%
if __name__ == "__main__":
    print(datetime.datetime.today())
    tokens = [token_ella]

    # for tok in tokens:
    #     send_to_firebase_cloud_messaging(tok, True)
    send_to_firebase_cloud_group_messaging(tokens, True)

#%% md
### push notification 일정한 시간 간격으로 보내기
#%%
from apscheduler.schedulers.blocking import BlockingScheduler

num = 1
def schedule_func():
    global num
    print(num)
    num += 1

    print(datetime.datetime.today())
    tokens = [token_ella]
    # for tok in tokens:
    #     pushFCMNotification.send_to_firebase_cloud_messaging(tok)
    send_to_firebase_cloud_group_messaging(tokens, True)


if __name__ == "__main__":
    scheduler = BlockingScheduler(timezone='Asia/Seoul', job_defaults={'max_instances': 1})
    
    ########## 아래 파라미터를 조정하세요.  ex ) seconds=10, minutes=5 ###########
    scheduler.add_job(schedule_func, 'interval', seconds=5)

    # starting now
    for j in scheduler.get_jobs():
        j.modify(next_run_time=datetime.datetime.now())

    scheduler.start()
#%%
```