## Load Test

- 일부러 시스템에 부하를 발생시킴. 문제 발생시,
  - => 코드 튜닝을 진행
  - => 하드웨어 증설 혹은 아키텍쳐 변경을 고려
- 상위 5%의 화면이 95% 사용자의 요청을 받는다고 가정하고 튜닝의 대상을 선정하는게 일반

- **To Check**
  - **Time**
    - 서비스가 얼마나 빠른가
  - **TPS**
    - 일정 시간동안 얼마나 많이 처리할 수 있는가
  - **Users**
    - 얼마나 많은 사람들이 동시에 사용할 수 있는가



## nGrinder

- 특징
  - 오픈소스 부하테스트 도구인 grinder를 기반으로 작성
  - 웹 기반으로 테스트 진행
  - python 스크립트를 작성해서 테스트 시나리오를 만들 수 있다.
- 선택 이유
  - Jthon 사용가능
    - python 문법으로 JAVA virtual 엔진에서 동작
    - JAVA의 api 사용가능
  - Jmeter는 JAVA 100% - spring과 더 잘맞아 보였음
  - UI good



## controller

- 웹 기반의 GUI 시스템
- 부하테스트 실시 & 모니터링
- stress 시나리오를 저장하고 재활용 가능



## agent

- 부하를 발생시키는 주체
- 복수의 머신에 설치해서 controller의 신호에 따라서 일시에 부하를 발생시킴



## 설치

> ~~직접 설치~~ 보다는 도커 이미지를 활용하는 방식이 간편

### 0. Prerequisite

- java JDK 1.6 이상 (JRE는 안된다)
- tomcat 6.x 이상

###1. nGrinder-controller 이미지 pull

```shell
docker run -d --name ngrinder_controller -v ~/ngrinder-controller:/opt/ngrinder-controller -p 80:80 -p 16001:16001 -p 12000-12009:12000-12009 ngrinder/controller:3.4
```

- 접속 포트
  - 80번 포트
- Agent가 12000 이상의 port를 이용하기 때문에 열어 놓는다.



### 2. nGrinder-agent 이미지를 pull

- 공식문서 said,
  - controller 컨테이너가 동작 중인 머신과 agent 컨테이너를 같은 인스턴스에서 구동하지 말 것을 강하게 권고
  - 여러 Agent들이 동작하다 보면, 부하를 발생시키는 머신의 자원을 모두 소모할 수 있음.

```shell
docker run -v ~/ngrinder-agent:/opt/ngrinder-agent -d --name ngrinder_agent ngrinder/agent:3.4 [controller_ip:controller_web_port]
```

```shell
docker run -v ~/ngrinder-agent:/opt/ngrinder-agent -d --name ngrinder_agent ngrinder/agent:3.4 34.97.222.236:80
```



### cf. 아래의 ngrinder-docker-compose.yml 파일을 활용해도 된다.

```shell
# docker-compose 설치
sudo curl -L "https://github.com/docker/compose/releases/download/1.28.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
docker-compose --version

# docker-compose 명령
docker compose -f ngrinder-docker-compose.yml up
```

```
version: '3.7'

services:
  controller:
    container_name: ngrinder-controller
    image: ngrinder/controller:latest
    environment:
      - TZ=Asia/Seoul
    ports:
      - "8880:80"
      - "16001:16001"
      - "12000-12009:12000-12009"
    volumes:
      - /tmp/ngrinder-controller:/opt/ngrinder-controller
    sysctls:
      - net.core.somaxconn=65000

  agent-1: 
    container_name: ngrinder-agent-1
    image: ngrinder/agent:latest
    links:
      - controller
    environment:
      - TZ=Asia/Seoul
    sysctls:
      - net.core.somaxconn=65000
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nproc:
        soft: 1024000
        hard: 1024000
      nofile:
        soft: 1024000
        hard: 1024000

  agent-2:
    container_name: ngrinder-agent-2
    image: ngrinder/agent:latest
    links:
      - controller
    environment:
      - TZ=Asia/Seoul
    sysctls:
      - net.core.somaxconn=65000
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nproc:
        soft: 1024000
        hard: 1024000
      nofile:
        soft: 1024000
        hard: 1024000
```

- `somaxconn(Socket Max Connection)`
  - 네트워크 연결 최대 개수 (윈도우의 경우 default=1000)
- `--ulimit memlock`
  - 메모리 주소 공간 최대 size (default 64kb)
  - -1 : swap 메모리 사용하지 않음
- `soft`
  - 새로운 프로그램을 생성하면 기본적으로 적용되는 한도
- `hard`
  - 소프트 한도에서 최대로 늘릴 수 있는 한도
- `nproc`
  - 해당 도메인(사용자, 그룹)의 최대 프로세스 개수
  - 한 사용자에게 허용 가능한 프로세스(user process)의 최대 개수 제한
- `nofile`
  - 해당 도메인(사용자, 그룹)이 오픈할 수 있는 최대 파일 개수
  - 오픈할 수 있는 파일기술자(`fd` : file descriptor)의 최대 개수 제한



### 3. 부하 test

- `Vuser`
  - virtual user로 동시에 접속하는 유저의 수를 의미
  - 사용할 수 있는 최대 Vuser의 총합 개수
    - Agent 개수 * process 개수 * thread 개수
- 특정 시간에 예약을 걸어둘 수 있음
- `Ramp-up`
  - 순차적으로 사용자 수 늘려가는게 가능

![image-20220213133849667](/Users/mac/Library/Application Support/typora-user-images/image-20220213133849667.png)

- **TPS**
  - 초당 트랜잭션의 수 = 초당 처리 수
- **트랜잭션**
  - HTTP Request가 성공할 때마다, 트랜잭션의 수가 1씩 증가
- **최고 TPS**
  - 초당 처리 수의 최대치
- **평균 테스트 시간**
  - 사용자가 request한 시점에서 시스템이 response할 때까지 걸린 시간
- **총 실행 테스트**
  - 테스트 시간동안 실행한 테스트의 수



### 4. Groovy Scripts 작성 

- quickStart의 문제점
  - `저장 후 시작`을 하더라도 이후 메인 페이지에서 동일 url에 대해 테스트 시작 버튼을 누르면 overwrite되어 사라짐
- 해결
  - `Scripts`를 이용

```java
import static net.grinder.script.Grinder.grinder
import static org.junit.Assert.*
import static org.hamcrest.Matchers.*
import net.grinder.plugin.http.HTTPRequest
import net.grinder.plugin.http.HTTPPluginControl
import net.grinder.script.GTest
import net.grinder.script.Grinder
import net.grinder.scriptengine.groovy.junit.GrinderRunner
import net.grinder.scriptengine.groovy.junit.annotation.BeforeProcess
import net.grinder.scriptengine.groovy.junit.annotation.BeforeThread
// import static net.grinder.util.GrinderUtils.* // You can use this if you're using nGrinder after 3.2.3
import org.junit.Before
import org.junit.BeforeClass
import org.junit.Test
import org.junit.runner.RunWith

import java.util.Date
import java.util.List
import java.util.ArrayList

import HTTPClient.Cookie
import HTTPClient.CookieModule
import HTTPClient.HTTPResponse
import HTTPClient.NVPair

/**
 * A simple example using the HTTP plugin that shows the retrieval of a
 * single page via HTTP. 
 * 
 * This script is automatically generated by ngrinder.
 * 
 * @author admin
 */
@RunWith(GrinderRunner)
class TestRunner {

	public static GTest test
	public static HTTPRequest request
	public static NVPair[] headers = []
	public static NVPair[] params = []
	public static Cookie[] cookies = []

	@BeforeProcess
	public static void beforeProcess() {
		HTTPPluginControl.getConnectionDefaults().timeout = 6000
		test = new GTest(1, "ec2-11.11.2.3.ap-northeast-2.compute.amazonaws.com")
        //test = new GTest(1, "{도메인주소orIP주소}")
		request = new HTTPRequest()
		grinder.logger.info("before process.");
	}

	@BeforeThread 
	public void beforeThread() {
		test.record(this, "test")
		grinder.statistics.delayReports=true;
		grinder.logger.info("before thread.");
	}
	
	@Before
	public void before() {
		request.setHeaders(headers)
		cookies.each { CookieModule.addCookie(it, HTTPPluginControl.getThreadHTTPClientContext()) }
		grinder.logger.info("before thread. init headers and cookies");
	}

  // 해당 부분을 커스텀!!!!!!!!
	@Test
	public void test(){
		HTTPResponse result = request.GET("http://ec2-11.11.2.3.ap-northeast-2.compute.amazonaws.com/api/member", params)
	//	HTTPResponse result = request.GET("{도메인주소orIP주소}", params)

		if (result.statusCode == 301 || result.statusCode == 302) {
			grinder.logger.warn("Warning. The response may not be correct. The response code was {}.", result.statusCode); 
		} else {
			assertThat(result.statusCode, is(200));
		}
	}
```

