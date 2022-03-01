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



## 문제 상황

- nginx에 던져지는 쿼리가 `access.log`, `error.log`에 기록이 되어야하는데, empty인 문제가 생겼음

## 문제 원인

- log 파일의 ownership이 바뀌어서 nginx가 이 파일에 쓰기 권한이 없는 문제

## 문제 해결

- 일단 nginx.conf 파일에 해당 경로에 로그파일을 쓰겠다는 설정을 해둠 (default인것 같긴 함. 생략해도 될듯)

```bash
http {
   ##
   # Logging Settings
   ##

   access_log /var/log/nginx/access.log;
   error_log /var/log/nginx/error.log;
}
```

- 파일을 지움
- nginx를 reload함

```bash
rm -f /var/log/nginx/*
nginx -s reload
```