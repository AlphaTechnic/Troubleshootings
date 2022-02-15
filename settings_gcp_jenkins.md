# 환경설정



## GCP

- 인스턴스 생성
  - 리전
    - 서울 (차선책 오사카)
  - 삭제보호
    - 사용 설정
  - 부팅디스크
    - 운영체제 : ubuntu 설정
  - 방화벽
    - HTTP 트래픽 허용
- VPC네트워크 -> 외부IP주소 ->고정 IP
- 방화벽 설정 (port open)
- https 인증



## Portainer

- [도커 설치](https://shanepark.tistory.com/237)
- 포테이너 설치

```shell
sudo docker run -d \
-p 9000:9000 \
--name=portainer \
--restart=unless-stopped \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /data/portainer/data:/data \
portainer/portainer
```

- Docker swarm

``` shell
sudo docker swarm init --advertise-addr [IP 주소]
sudo docker swarm join-token manager
```

- secrets 등록





## Jenkins

### 서비스의 구조

![Architect_true](./imgs/Architect_true.jpg)



### 미리 읽는 Troubleshooting Point**

- Docker가 sudo로 실행되어서 jenkins가 github에 등록한 키 인식을 못하는 문제 
  - ⇒ jenkins를 설치한 주체가 root가 아니라 alpha_technic이어야 한다.
  - ⇒ jenkins를 도커 container가 아닌 **hostPC에 직접 설치**하는 방향으로 해결
- 이따금씩 서버가 다운되고 다시 뜨고 하는데, 그렇게 되면 컨테이너가 죽게될 지도 모름. 
  - ⇒ 도커스웜의 오토힐링에 의존하기 보다, GCP를 믿자.
  - ⇒ 리눅스 `서비스`에 등록을 한다. (`jenkins.service` 파일 생성)
- 계정이 alpha.technic이면 parsing에 문제가 있는 경우가 있기도 하다. 
  - ⇒ alpha_technic으로 바꿔서 해결
- jenkins가 실행할 script에 sudo 명령 입력이 안됨.. 
  - ⇒ alpha_technic 계정도 sudo 명령 없이 docker를 실행할 수 있도록 설정
  - ⇒ `/etc/sudoers`를 수정한다.

### Big Concept

- 리눅스 `서비스`에 등록하고자 함
  - why?
    1. jenkins를 도커 컨테이너로 만들면 VM, Docker, 컨테이너 3가지가 모두 성공적으로 오토힐링 되어야 함
    2. Github 레포 Webhooks에 jenkins 등록할 때 User 문제(docker를 sudo로 띄워서 문제)가 생김

- GCP는 vpn 외부에 있고, 카카오브레인 Github는 vpn 내부에 있어서 환경 셋팅이 곤란
  - => Private Github Organization

- Dockerfile 있는 상황

```dockerfile
FROM python:3.9.0

# 도커가 (혹은 portainer가) build 시 속도를 위해 caching을 해서,
# 장고에서 내용을 수정 후 다시 build할 때 마저도 cache하는 불상사가 없도록
# 아래의 명령어를 바꿔가면서 (예: testing!으로 했다가 test!!로 했다가 등등) build
RUN echo "ssddddeSting!!!"

RUN mkdir /root/.ssh/

ADD ./.ssh/id_rsa /root/.ssh/id_rsa

RUN chmod 600 /root/.ssh/id_rsa

RUN touch /root/.ssh/known_hosts

RUN ssh-keyscan github.com >> /root/.ssh/known_hosts


WORKDIR /home/alpha.technic/

# 한국 시간으로 설정
ENV TZ=Asia/Seoul
RUN apt-get install -y tzdata
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

RUN git clone -b dev --single-branch git@github.com:neo-wolley/wolley-deploy.git

WORKDIR /home/alpha.technic/wolley-deploy/

RUN pip install -r requirements.txt

RUN pip install gunicorn

RUN pip install mysqlclient

EXPOSE 8000

CMD ["bash", "-c", "python manage.py collectstatic --settings=myapi.settings.deploy --no-input && python manage.py migrate --settings=myapi.settings.deploy && gunicorn myapi.wsgi --env DJANGO_SETTINGS_MODULE=myapi.settings.deploy --bind 0.0.0.0:8000"]
```

- docker-compose.yml 있는 상황

```yaml
version: "3.7"
services:
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
    container_name: nginx_container

  django_container_gunicorn:
    image: django_image:1
    networks:
      - network
    volumes:
      - static-volume:/home/alpha_technic/wolley-deploy/staticfiles
    secrets:
      - MYSQL_PASSWORD
      - DJANGO_SECRET_KEY
      - MY_REST_API_KEY
    depends_on:
      - mariadb
    container_name: django_container_gunicorn

  mariadb:
    image: mariadb:10.5
    networks:
      - network
    volumes:
      - /home/alpha_technic/mariadb_setting/my.cnf:/etc/mysql/my.cnf
      - maria-database:/var/lib/mysql
    secrets:
      - MYSQL_PASSWORD
      - MYSQL_ROOT_PASSWORD
    command: --lower_case_table_names=0
    environment:
      MYSQL_DATABASE: wolleydb
      MYSQL_USER: root
      MYSQL_PASSWORD_FILE: /run/secrets/MYSQL_PASSWORD
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/MYSQL_ROOT_PASSWORD
    container_name: mariadb_container

networks:
  network:

volumes:
  maria-database:
  static-volume:

secrets:
  DJANGO_SECRET_KEY:
    external: true
  MYSQL_PASSWORD:
    external: true
  MYSQL_ROOT_PASSWORD:
    external: true
  MY_REST_API_KEY:
    external: true
```



### java 설치

```shell
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install openjdk-11-jdk

java -version
```

```shell
# 환경 설정
vim ~/.bashrc

# ~/.bashrc
export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which java))))
export PATH=$PATH:$JAVA_HOME/bin

# 적용, 확인
source ~/.bashrc
echo $JAVA_HOME
```



### Jenkins 설치

- jenkins 설치

```shell
mkdir jenkins
cd jenkins

wget https://get.jenkins.io/war/2.309/jenkins.war
```



- service 등록
  - **!주의!** 아래 스크립트에서 ExecStart에 경로를 각자의 환경에 맞게 바꿀것.

```
sudo vi /etc/systemd/system/jenkins.service

# script 입력
[Unit]
Description=jenkins
After=network.target
Requires=network.target

[Service]
Type=simple
ExecStart=/usr/bin/java -jar /home/alpha_technic/jenkins/jenkins.war --httpPort=8080
Restart=always
User=alpha_technic
RestartSec=20

[Install]
WantedBy=multi-user.target

```



- jenkins 설치 확인

```shell
# service restart, start jenkins, 상태 확인
sudo systemctl daemon-reload
sudo systemctl kill jenkins.service
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins

```



### Jenkins Github 설정

[<캡쳐본과 함께 안내된 링크>](https://www.dongyeon1201.kr/9026133b-31be-4b58-bcc7-49abbe893044#fd6c4c7c-4c5b-43ee-a4f0-4100fba870e0)

- Jenkins에 private key 등록
- Github 레포 settings에 public key 등록
- Github 레포 Webhooks에 jenkins 등록
- 더불어 Github → Discord 알림 설정



### Jenkins Job 설정

- jenkins에게 sudo 권한 주기

```shell
# /etc/sudoers 수정 (ctrl+X - y - enter)
sudo visudo

# 아래 내용 추가 (!주의! alpha_technic은 사용환경에 맞게 수정)
alpha_technic ALL=(ALL) NOPASSWD: ALL
jenkins ALL=(ALL) NOPASSWD: ALL
```



- [ 빌드 유발 ] 탭 선택
  - `GitHub hook trigger for GITScm polling` 체크

- [ Build ] 탭 선택 → `Execute shell` 탭 선택
  - 빌드 & 테스트를 위한 코드 작성

```shell
whoami
pwd
cd
cd /home/alpha_technic

docker stack rm dj_stack
sleep 10
docker rmi django_image:1

docker image build -t django_image:1 .
docker stack deploy -c docker-compose.yml dj_stack
sleep 20
docker service ls
```


