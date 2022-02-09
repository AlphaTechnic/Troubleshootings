- 문제 상황 

  - nginx 컨테이너의 터미널에서 date를 찍어봤을 때, 미국 timezone으로 나온다.
  - config 파일을 뒤져서 설정을 바꿔주면 되는데, 안내된 내용은 
    - nginx timezone 설정을 localtime으로 바꿔라
    - 그러면, nginx가 timezone을 도커의 timezone으로 보게 된다.
    - 도커의 timezone을 바꿔라

- 문제 원인 추정 

  - nginx가 timezone을 도커의 timezone을 보질 못한다.

- 문제 원인 

  - nginx가 뜰 때, environment 설정

- 해결 방안 

  - docker-compose.yml 

    - environment 옵션 활용

      ```yaml
      ngnix:
          image: nginx:1.19.5
          environment:
            - TZ=Asia/Seoul
          volumes:
            - /home/alpha_technic/nginx_setting/nginx.conf:/etc/nginx/nginx.conf
            - static-volume:/data/static
          networks:
            - network
          ports:
            - 80:80
      ```