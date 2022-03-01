### 실험 준비

- **`사용자가 앱에 접속한 경우`**, 이를 트리거로 FE에서 BE로 GPS 로그가 넘어오게 된다. (http POST 통신)

- 사용자가 

  ```
  앱에 접속하는 주기
  ```

  에 따라 다양한 시나리오가 나올 수 있다.

  - 이를테면, 24시간 중 사용자가 앱을 3번 접속하였다면, 3번의 POST 통신에 나누어 GPS 로그가 BE로 전달될 것이다.
  - 24시간 중 사용자가 앱을 한번도 접속하지 않았다면, 언젠가 앱에 들어올 때 local에 기록된 GPS 로그가 단 1번의 POST 통신으로 BE에 전달될 것이다.

### 실험 목적

- **GPS 로그가 BE로 전달되는 방식에 상관없이, 동일한 interval들을 가진 파이차트가 생성되어야 한다.**

### 시나리오

- 시나리오 1
  - **사용자가 3일 정도 시간이 지난 뒤에 앱을 켠 경우**
  - 24시간의 모든 GPS 로그가 단 한번의 POST 통신으로 넘어오게 되는 경우
- 시나리오 2
  - **사용자가 앱에 들어오는 주기가 잦은 경우**
  - 24시간의 GPS 로그 하나하나를 POST 통신으로 넘긴다는 극단적인 가정으로 test 진행
- 시나리오 3
  - **사용자가 앱을 하루에 두 세번 정도 켜게되는 비교적 일반적인 경우**
  - 24시간의 모든 GPS 로그가 POST 통신 2 ~ 3회로 나누어 들어오게 된다.

### Test 코드

```python
from django.contrib.auth.models import User
from django.urls import reverse

from rest_framework.test import APITestCase
from rest_framework import status
from intervalapp.models import IntervalStay, IntervalMove

import json
import random
from dailypathapp.tests.utils import *

"""
3 시나리오

1. GPSLog 데이터를 뭉텅이로 넘기는 경우
2. GPSLog 데이터를 매번 POST 통신으로 넘기는 경우
3. 1과 2가 뒤섞인 경우
"""

class TestClass(APITestCase):
    timeSequence = list()

    @classmethod
    def setUpTestData(cls):
        fp = open("./dailypathapp/tests/dummydata", 'r')
        cls.timeSequence = mk_timeSequence_from_txt_file(fp)
        cls.timeSequence.sort(key=lambda x: x["time"])
        fp.close()

    def test_모든_GPS로그가_POST_통신_한방에_뭉텅이로_오는_시나리오(self):
        data_to_send = {"user": "TESTBOY", "timeSequence": self.timeSequence, "fcmToken": ""}
        url = reverse(viewname="dailypathapp:pathdaily_request")
        json_data = json.dumps(data_to_send)
        response = self.client.post(
            path=url,
            data=json_data,
            content_type="application/json"
        )

        print("\\n********** (뭉텅이) RESULT ************")
        print(f"IntervalStay: {len(IntervalStay.objects.all())} 개")
        print(f"IntervalMove: {len(IntervalMove.objects.all())} 개")
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        print("******************************")

    def test_모든_GPS로그_각각이_POST_통신으로_쪼개져서_오는_시나리오(self):
        for timestamp in self.timeSequence:
            data_to_send = {"user": "TESTBOY", "timeSequence": [timestamp], "fcmToken": ""}
            url = reverse(viewname="dailypathapp:pathdaily_request")
            json_data = json.dumps(data_to_send)
            response = self.client.post(
                path=url,
                data=json_data,
                content_type="application/json"
            )

        print("\\n********** (모든로그가POST) RESULT ************")
        print(f"IntervalStay: {len(IntervalStay.objects.all())} 개")
        print(f"IntervalMove: {len(IntervalMove.objects.all())} 개")
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        print("******************************")

    def test_GPS로그가_드문드문_POST통신으로_넘어오는_시나리오(self):
        chk = [False for _ in range(len(self.timeSequence))]
        for _ in range(10):
            chk[random.randint(1, len(self.timeSequence) - 1)] = True

        l = 0
        for r, timestamp in enumerate(self.timeSequence):
            if chk[r]:
                data_to_send = {"user": "TESTBOY", "timeSequence": self.timeSequence[l:r], "fcmToken": ""}
                url = reverse(viewname="dailypathapp:pathdaily_request")
                json_data = json.dumps(data_to_send)
                response = self.client.post(
                    path=url,
                    data=json_data,
                    content_type="application/json"
                )
                l = r

        print("\\n********** (이따금씩POST) RESULT ************")
        print(f"IntervalStay: {len(IntervalStay.objects.all())} 개")
        print(f"IntervalMove: {len(IntervalMove.objects.all())} 개")
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        print("******************************")
```

### 실험 결과

- 동일한 Stay Point가 2개의 interval로 쪼개지는 버그 발견

```bash
********** (이따금씩POST) RESULT ************
IntervalStay: 4 개
IntervalMove: 4 개
******************************
.
********** (모든로그가POST) RESULT ************
IntervalStay: 6 개
IntervalMove: 5 개
******************************
.
********** (뭉텅이) RESULT ************
IntervalStay: 6 개
IntervalMove: 4 개
******************************
```

- 버그 fix 중..