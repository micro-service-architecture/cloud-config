# cloud-config
- 분산 시스템에서 서버, 클라이언트 구성에 필요한 설정 정보(`application.yml`) 를 외부 시스템에서 관리한다.
- 하나의 중앙화 된 저장소에서 구성요소 관리가 가능하다.
- 각 서비스를 다시 빌드하지 않고 바로 적용 가능하다.
- 애플리케이션 배포 파이프라인을 통해 DEV-QA-PROD 환경에 맞는 구성 정보 사용이 가능하다.

![image](https://user-images.githubusercontent.com/31242766/195135318-62b48222-d165-4419-8bfa-1b77e044fa06.png)

## 변화된 Conifg 설정 정보 값 가져오는 방법
- 서버 재기동
- Actuator refresh
- Spring Cloud Bus 사용

### Actuator
- Application 상태, 모니터링할 수 있는 방법을 제공한다. Metric 수집을 위한 Http End point 를 제공한다. 
- 실행 중인 Application 의 내부를 볼 수 있게 하고 어느 정도까지는 Application 의 작동 방법을 제어할 수 있게 한다. 예를 들면, 다음과 같다.
  - 애플리케이션 환경에서 사용할 수 있는 구성 속성들
  - 애플리케이션에 포함된 다양한 패키지의 로깅 레벨(logging level)
  - 애플리케이션이 사용 중인 메모리
  - 지정된 엔드포인트가 받은 요청 횟수
  - 애플리케이션의 건강 상태 정보      
  
|HTTP 메서드|경로|설명|디폴트 활성화|
|----|------------------------|-------------------------|-----------|
|POST|/refresh|변경된 설정 정보를 반영한다.|NO|
|GET|/auditevents|호출된 감사(audit) 이벤트 리포트를 생성한다.|NO|
|GET|/beans|스프링 애플리케이션 컨텍스트의 모든 빈을 알려준다.|NO|
|GET|/conditions|성공 또는 실패했던 자동-구성 조건의 내역을 생성한다.|NO|
|GET|/configprop|모든 구성 속성들을 현재 값과 같이 알려준다.|NO|
|GET,POST,DELETE|/env|스프링 애플리케이션에 사용할 수 있는 모든 속성 근원과 이 근원들의 속성을 알려준다.|NO|
|GET|/env/{toMatch}|특정 환경 속성의 값을 알려준다.|NO|
|GET|/health|애플리케이션의 건강 상태 정보를 반환한다.|Yes|
|GET|/heapdump|힙(heap) 덤프를 다운로드한다.|NO|
|GET|/httptrace|가장 최근의 100개 요청에 대한 추적 기록을 생성한다.|NO|
|GET|/info|개발자가 정의한 애플리케이션에 관란 정보를 반환한다.|Yes|
|GET|/loggers|애플리케이션의 패키지 리스트(각 패키지의 로깅 레벨이 포함된)를 생성한다.|NO|
|GET|/loggers/{name}|지정된 로거의 로깅 레벨(구성된 로깅 레벨과 유효 로깅 레벨 모두)을 반환한다. 유효 로깅 레벨은 HTTP POST 요청으로 설정될 수 있다.|NO|
|GET|/mappings|모든 HTTP 매핑과 이 매핑들을 처리하는 핸들러 메서드들의 내역을 제공한다.|NO|
|GET|/metrics|모든 메트릭 리스트를 반환한다.|NO|
|GET|/metrics/{name}|지정된 메트릭의 값을 반환한다.|NO|
|GET|/scheduledtasks|스케줄링된 모든 태스크의 내역을 제공한다.|NO|
|GET|/threaddump|모든 애플리케이션 스레드의 내역을 반환한다.|NO|

### Actuator refresh
1. `spring-cloud-starter-config` 로 외부에 설정 정보를 저장해두었다.
```yml
server:
  port: 8888

spring:
  application:
    name: config-service
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations: file:C:/git-local-repo
```
#### git-local-repo 폴더
![image](https://user-images.githubusercontent.com/31242766/195380855-4f71c816-e8b0-447e-bd9b-00a316d44fc9.png)

```yml
token:
  expiration_time: 86400000 # 60 * 60 * 24 * 1000 (하루)
  secret: comCloudServiceBootUserServiceSecretKeyAuthorizationJwtManageToken

gateway:
  ip: 59.29.153.172
```
2. 사용하고자 하는 [마이크로서비스](https://github.com/multi-module-project/cloud-service/tree/master/boot-user-service)에서 `spring-cloud-starter-bootstrap` 을 이용(`bootstrap.yml`)하여 미리 설정 정보를 가져온다.
```yml
spring:
  cloud:
    config:
      uri: http://127.0.0.1:8888
      name: ecommerce
```
![image](https://user-images.githubusercontent.com/31242766/195382288-3f247548-7520-4db2-acc2-1d8a5b3eff92.png)

```java
@GetMapping("/health_check")
public String status() {
    return String.format("It's Working in User Service"
            + ", port(local.server.port)=" + env.getProperty("local.server.port")
            + ", port(server.port)=" + env.getProperty("server.port")
            + ", token secret=" + env.getProperty("token.secret")
            + ", token expiration time=" + env.getProperty("token.expiration_time"));
}
```
```console
It's Working in User Service, port(local.server.port)=60058, port(server.port)=0, 
token secret=comCloudServiceBootUserServiceSecretKeyAuthorizationJwtManageToken, token expiration time=86400000
```
3. 사용하고자 하는 [마이크로서비스](https://github.com/multi-module-project/cloud-service/tree/master/boot-user-service)에서 actuator endpoints 작성 (`application.yml`)
```yml
...
management:
  endpoints:
    web:
      exposure:
        include: refresh, health, beans
...
```
4. 변경된 설정 정보 반영 (`/actuator/refresh`)
```yml
token:
  expiration_time: 86400000 # 60 * 60 * 24 * 1000 (하루)
  secret: user_token

gateway:
  ip: 59.29.153.172
```
![image](https://user-images.githubusercontent.com/31242766/195383759-c335ab25-5ed5-43bc-a6f1-656cda6aeb32.png)

![image](https://user-images.githubusercontent.com/31242766/195384213-3777f1ec-1cbb-4439-8171-3901737c0865.png)

### Multiple environments
DEV-QA-PROD 환경에 맞는 구성 정보 사용이 가능하다.

![image](https://user-images.githubusercontent.com/31242766/195982432-4272b1c3-45f4-4f90-87f1-cea5c800d3d9.png)

git-local-repo 폴더에 파일을 추가하고 각 JWT 토큰을 다르게 수정해보자.

![image](https://user-images.githubusercontent.com/31242766/195982669-1b16b9eb-02f2-4caf-a93d-40c29bd7e750.png)

`gateway` 와 `마이크로서비스` 에 `bootstrap.yml` 에 profiles 를 추가해보자.
```yml
spring:
  ...
  profiles:
    active: dev
```
![image](https://user-images.githubusercontent.com/31242766/195983083-a25d1b76-2641-4f31-a6ab-abf4853faab8.png)

```yml
spring:
  ...
  profiles:
    active: prod
```
![image](https://user-images.githubusercontent.com/31242766/195983194-74e13903-2a03-4f6f-98d2-13784b611c05.png)

### Git Remote Repository
원격 저장소에 있는 정보를 가져온다.
```yml
spring:
  application:
    name: config-service
  cloud:
    config:
      server:
        git:
          uri: https://github.com/multi-module-project/cloud-config-repo
```

### Native File Repository
로컬 경로에 있는 정보를 가져온다. (`prfiles.active: native`) 로 설정한다.
```yml
spring:
  application:
    name: config-service
  profiles:
    active: native
  cloud:
    config:
      server:
        git:
          uri: https://github.com/multi-module-project/cloud-config-repo
        native:
          search-locations: file:C:/git-local-repo
```

### Spring Cloud Bus
JWT 토큰 정보를 변경하고나서 `localhost:8000/user-service/actuator/refresh` 를 통해 `user-serivce` 에 변경된 정보를 반영해보자. 해당 `user-service` 에 변경된 토큰 정보가 반영된다. 그런데 `gateway` 는 refresh 하지 않았다면? `gateway` 에 변경된 토큰 정보가 반영되지 않는다. 그래서 `localhost:8000/actuator/refresh` 를 통해 `gateway` 또한 변경된 정보를 반영해야한다. 여기서, 비효율적인 문제가 발생한다. config 정보를 변경할 때마다 `마이크로서비스` 및 `gateway` 에 `/actuator/refresh` 를 통해 변경된 정보를 반영해주어야 하기 때문이다. 이러한 비효율적인 문제를 해소하고자 `Spring Cloud Bus` 가 등장했다.

#### Actuator bus-refresh Endpoint
- 분산 시스템의 마이크로서비스를 경량 메시지 브로커([RabbitMQ](https://github.com/haeyonghahn/TIL/tree/master/RabbitMQ)) 와 연결      
Spring Cloud Bus 에 연결되어 있는 각각의 마이크로서비스에 변경된 정보를 알려줄 때 `AMQP(Advanced Message Queuing Protocol)` 방식으로 전달해준다.

![image](https://user-images.githubusercontent.com/31242766/196180785-c3a45495-f7d7-4fe8-a72b-789fb7e78e11.png)

- 상태 및 구성에 대한 변경 사항을 연결된 마이크로서비스에게 전달

![tempsnip](https://user-images.githubusercontent.com/31242766/196176144-c4d0a877-86c1-44f9-a4f5-30e02afb148d.png)

연결되어있는 각각의 마이크로서비스(`USER-SERVICE`, `ORDER-SERVICE`, `CATALOG-SERVICE`)는 외부에서 `/busrefresh` 를 호출하게되면 다른 마이크로서비스에게 변경된 내용을 전달하게 된다.

* 참고 : `/busrefresh` 를 호출할 때 Spring Cloud Bus 에 연결되어 있는 마이크로서비스를 호출하면 된다. Cloud Bus 는 변경 사항을 감지하게 되고 Cloud Bus 와 연결되어 있는 다른 마이크로서비스에 변경 사항을 전달하게 된다. `Config Server`, `Gateway` 어디에서든지 `/busrefresh` 를 호출하게 되면 Cloud Bus 와 연결되어 있는 곳에 변경 사항을 전달하게 된다.

## 참고
http://forward.nhnent.com/hands-on-labs/java.spring-boot-actuator/04-endpoint.html      
https://junjangsee.tistory.com/entry/springboot-actuator
