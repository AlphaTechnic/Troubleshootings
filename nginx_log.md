- 문제 상황

  - nginx에 던져지는 쿼리가 access.log, error.log에 기록이 되어야하는데, empty인 문제가 생겼음

- 문제 원인

  - log 파일의 ownership이 바뀌어서 nginx가 이 파일에 쓰기 권한이 없는 문제

- 문제 해결

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
  sudo rm -f /var/log/nginx/*
  sudo nginx -s reload
  ```