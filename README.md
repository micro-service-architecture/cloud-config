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
### config-service 정보 가져오기
마이크로서비스 `bootstrap.yml` 파일에서 `profiles.active` 설정없이 `spring.cloud.config.name` 을 `config-server` 의 `spring.application.name` 으로 설정하면 `config-server` 의 `application.yml` 파일의 정보를 가져온다.
```yml
spring:
  cloud:
    config:
      uri: http://127.0.0.1:8888
#      name: ecommerce
      name: config-service
#  profiles:
#    active: prod
```

![image](https://user-images.githubusercontent.com/31242766/196247674-b2328d66-39cb-49de-acff-4db39761d89a.png)


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

#### Spring Cloud Bus 테스트
구동중인 서버는 아래와 같다.
- [config-server](https://github.com/multi-module-project/cloud-config)
- [discovery-server](https://github.com/multi-module-project/cloud-system/tree/master/discovery)
- [gateway-server](https://github.com/multi-module-project/cloud-system/tree/master/gateway)
- [user-server](https://github.com/multi-module-project/cloud-service/tree/master/boot-user-service)

`user-server` 를 통해 회원가입 및 로그인 절차를 통해서 `/health_check` 한 결과 정보이다.
```console
It's Working in User Service, port(local.server.port)=52031, port(server.port)=0, token secret=comCloudServiceBootUserServiceSecretKeyAuthorizationJwtManageTokenNative, token expiration time=86400000
```

해당 상태에서 `config-server` 의 설정 정보를 수정 후 확인해보자. 현재 `config-server` 의 `application.yml` 기본 정보이며 로컬에서 설정 정보를 가져올 것이다.
```yml
server:
  port: 8888
spring:
  application:
    name: config-service
    ...
  profiles:
    active: native
```
```yml
// http://localhost:8888/config-server/default

{
  "name": "config-server",
  "profiles": [
    "default"
  ],
  "label": null,
  "version": null,
  "state": null,
  "propertySources": [
    {
      "name": "file:C:\\git-local-repo\\application.yml",
      "source": {
        "token.expiration_time": 86400000,
        "token.secret": "comCloudServiceBootUserServiceSecretKeyAuthorizationJwtManageTokenNativeChanged",
        "gateway.ip": "59.29.153.172"
      }
    }
  ]
}
```
`token.secret` comCloudServiceBootUserServiceSecretKeyAuthorizationJwtManageTokenNative -> comCloudServiceBootUserServiceSecretKeyAuthorizationJwtManageTokenNativeChanged    
로 수정한 상태이다.

이 상태로 `user-server` 의 상태만 갱신해보자. 갱신 후에는 `user-server` 의 정보가 갱신된 것을 알 수 있다.

![image](https://user-images.githubusercontent.com/31242766/196255139-4e2c8bd3-6a73-4511-a4ea-1629357eceed.png)

![image](https://user-images.githubusercontent.com/31242766/196255350-9e5b5043-817d-44da-bc56-14c758d47ff3.png)

또한, `bus-refresh` 를 통해서 연결되어 있는 `gateway-server` 의 정보도 갱신된 것을 알 수 있다.

![image](https://user-images.githubusercontent.com/31242766/196255774-97e3a8bf-ee77-4476-a3e8-1d6066f5dc0c.png)

## 설정 정보의 암호화 처리
config 설정 정보(`application.yml`, `ootstrap.yml`) 에 노출되면 위험한 정보들이 존재한다. 이러한 정보들을 암호화 처리하여 노출되지 않도록 처리한다.
- 대칭 키 암호화 방식(Symmetric Encryption)      
암호화와 복호화할 때 같은 키 값을 사용하는 방식이다.
- 비대칭 키 암호화 방식(Asymmetric Encryption)     
암호화할 때 사용하는 키와 복호화할 때 사용하는 키를 달리 해서 사용하는 방식이다.

![image](https://user-images.githubusercontent.com/31242766/196425328-1aa18760-d541-461a-bfae-4b40b061c9bf.png)

### 대칭 키 암호화 방식
- [config-server](https://github.com/multi-module-project/cloud-config)
- [user-server](https://github.com/multi-module-project/cloud-service/tree/master/boot-user-service)

해당 서버를 사용하여 진행할 것이다. 먼저, `config-server` 에 `bootstrap.yml` 파일을 추가하고 암호 키를 추가한다.
```yml
encrypt:
  key: abcdefghijklmnopqrstuvwxyz0123456789
```
`config-server` 를 실행시킨 후에 encrypt 및 decrypt 되는 것을 확인할 수 있다. 

![image](https://user-images.githubusercontent.com/31242766/196434116-576daf28-3668-4b7f-8ef6-cf657b5d8063.png)

![image](https://user-images.githubusercontent.com/31242766/196434172-6008a71d-9b3b-4d0f-a518-f24381b1052d.png)

다음으로, `user-server` 에 설정되어 있던 datasource 정보를 `config-server` 로 옮기고 `user-server` 의 `bootstrap.yml` 으로 설정된 암호화 정보를 복호화해서 가져올 것이다.
```yml
# application.yml
spring:
  ...
  h2:
    console:
      enabled: true
      settings:
        web-allow-others: true
      path: /h2-console
# datasource:
#   driver-class-name: org.h2.Driver
#   url: jdbc:h2:mem:testdb
#   username: sa
#   password: 1234
...
```
```yml
# bootstrap.yml
spring:
  cloud:
    config:
      uri: http://127.0.0.1:8888
      name: user-service
```

![image](https://user-images.githubusercontent.com/31242766/196435593-bd14f48a-76af-40be-9e8c-5e087391db12.png)

```yml
# user-service.yml
spring:
  datasource:
    driver-class-name: org.h2.Driver
    url: jdbc:h2:mem:testdb
    username: sa
    password: 1234 <- 해당 정보를 암호화
    ...
```

![image](https://user-images.githubusercontent.com/31242766/196437355-af581fa2-8968-4fd5-861d-eb9dcb14a7df.png)

```yml
# user-service.yml
spring:
  datasource:
    driver-class-name: org.h2.Driver
    url: jdbc:h2:mem:testdb
    username: sa
    password: '{cipher}631d72a0b0a8ef85b69b7f4657c454b7385d0c8824959eef9e5caa90f3657e93'
    ...
```

`config-server` 를 실행 후 결과를 확인해보면, 데이터소스 비밀번호가 복호화되서 나오는 것을 확인할 수 있다.
```json
// 20221019073955
// http://localhost:8888/user-service/default

{
  "name": "user-service",
  "profiles": [
    "default"
  ],
  "label": null,
  "version": null,
  "state": null,
  "propertySources": [
    {
      "name": "file:C:\\git-local-repo\\user-service.yml",
      "source": {
        "spring.datasource.driver-class-name": "org.h2.Driver",
        "spring.datasource.url": "jdbc:h2:mem:testdb",
        "spring.datasource.username": "sa",
        "token.expiration_time": 86400000,
        "token.secret": "comCloudServiceBootUserServiceSecretKeyAuthorizationJwtManageTokenNativeChanged",
        "gateway.ip": "59.29.153.172",
        "spring.datasource.password": "1234"
      }
    },
    {
      "name": "file:C:\\git-local-repo\\application.yml",
      "source": {
        "token.expiration_time": 86400000,
        "token.secret": "comCloudServiceBootUserServiceSecretKeyAuthorizationJwtManageTokenNativeChanged",
        "gateway.ip": "59.29.153.172"
      }
    }
  ]
}
```

그렇다면, `user-server` 를 실행시킨 후에 `h2-console` 로그인이 가능한 것을 확인해보자. 비밀번호 `1234` 를 입력 후에 `Test Connection` 을 해보면 성공했다는 것을 알 수 있다.

![image](https://user-images.githubusercontent.com/31242766/196558950-d4f9ec12-b7be-4241-b984-27eb7939aa98.png)

### 비대칭 키 암호화 방식
`keystore` 라는 폴더를 만들었고 해당 폴더에 `java 라이브러리`에서 제공해주는 `keytool` 를 이용하여 키를 생성한다.
```console
C:\keystore>keytool -genkeypair <- 키 생성 명령 
                    -alias apiEncryptionKey <- 키의 별칭 
                    -keyalg RSA <- 키에서 사용하는 알고리즘
                    -dname "CN=Kenneth Lee, OU=API Development, O=joneconsulting.co.kr, L=Seoul, C=KR" <- 키 정보
                    -keypass "1q2w3e4r" <- key 비밀번호
                    -keystore apiEncryptionKey.jks <- 생성할 파일 이름
                    -storepass "1q2w3e4r" <- store 비밀번호
```
![image](https://user-images.githubusercontent.com/31242766/196693109-b2a73726-8b3b-4750-b9a7-ab20f596fe04.png)

#### 개인 키 정보 확인
```console
C:\keystore>keytool -list -keystore apiEncryptionKey.jks -v
```
![image](https://user-images.githubusercontent.com/31242766/196694340-26a50f1c-0acf-4c1c-85de-9181d2bda3b3.png)

#### 공개 키 생성하기
```console
C:\keystore>keytool -export 
                    -alias apiEncryptionKey <- 키 별칭
                    -keystore apiEncryptionKey.jks <- 실제 파일 이름
                    -rfc <- Request for Comments 의 약자로서 인터넷에서 사용하는 표준 양식
                    -file trustServer.cer <- 생성할 파일 이름
```
![image](https://user-images.githubusercontent.com/31242766/196696275-77b298db-02fc-40ec-b0f4-37d2ecfc3895.png)

`config-server` 에 `bootstrap.yml` 파일에 개인 키를 추가한다.
```yml
encrypt:
#  key: abcdefghijklmnopqrstuvwxyz0123456789 <- 대칭키
  keystore:
    location: file:C:/keystore/apiEncryptionKey.jks
    password: 1q2w3e4r
    alias: apiEncryptionKey
```
`config-server` 를 실행시킨 후에 encrypt 및 decrypt 되는 것을 확인할 수 있다.

![image](https://user-images.githubusercontent.com/31242766/196697450-1eaacc19-d366-421d-85b1-73c470e056ad.png)

![image](https://user-images.githubusercontent.com/31242766/196697618-a41c2c03-1ab4-4c8c-9c07-b20dfd46b588.png)

`user-server` 에 설정되어 있던 datasource 정보를 암호화해보자.
```yml
# user-service.yml
spring:
  datasource:
    driver-class-name: org.h2.Driver
    url: jdbc:h2:mem:testdb
    username: sa
    password: 1234 <- 해당 정보를 암호화
    ...
```

![image](https://user-images.githubusercontent.com/31242766/196698622-13fde244-8505-42ea-a322-1b3b6e6e1236.png)

```yml
# user-service.yml
spring:
  datasource:
    driver-class-name: org.h2.Driver
    url: jdbc:h2:mem:testdb
    username: sa
    # password: '{cipher}631d72a0b0a8ef85b69b7f4657c454b7385d0c8824959eef9e5caa90f3657e93'
    password: '{cipher}AQA4mhTmwhe3U85rn2Aeovb7RZ9z9fwm9C7+QY9Lajya/QBG25RM6Jru/3YnT6c/
    rP1i5gdCbwJOF9HB3nfZvT8JWUD8PB/YnxrwIaoQE93+KHwGyUoZC7LVNwvvw+8OC2rfuRnxuMSM
    tXC+NZoOUYIPgxbweTp0btR4KI6P4V9tlBvssK86wdRTmyKUu9vjUf9tZbRhROhI67sumujh5xmqUxga
    Ay2f8qDLHJ9nsdEInVzHLE3Tl/sIfBWmm3CP6b3JQUXVoprfAttnUkhEu+sHHc8iqY4lKwebuRqsVM3F
    7+A9b/l9UcNWgEfTxlyHqPDPMXEt+J2QzPQlHmppIccN4LCnXGScAwHVbcaS6HH8JygCtTxjsDUhlbuKeSFmYOw='
    ...
```
`config-server` 를 실행 후 결과를 확인해보면, 데이터소스 비밀번호가 복호화되서 나오는 것을 확인할 수 있다.
```json
// 20221019221043
// http://localhost:8888/user-service/default

{
  "name": "user-service",
  "profiles": [
    "default"
  ],
  "label": null,
  "version": null,
  "state": null,
  "propertySources": [
    {
      "name": "file:C:\\git-local-repo\\user-service.yml",
      "source": {
        "spring.datasource.driver-class-name": "org.h2.Driver",
        "spring.datasource.url": "jdbc:h2:mem:testdb",
        "spring.datasource.username": "sa",
        "token.expiration_time": 86400000,
        "token.secret": "comCloudServiceBootUserServiceSecretKeyAuthorizationJwtManageTokenNativeChanged",
        "gateway.ip": "59.29.153.172",
        "spring.datasource.password": "1234"
      }
    },
    {
      "name": "file:C:\\git-local-repo\\application.yml",
      "source": {
        "token.expiration_time": 86400000,
        "token.secret": "comCloudServiceBootUserServiceSecretKeyAuthorizationJwtManageTokenNativeChanged",
        "gateway.ip": "59.29.153.172"
      }
    }
  ]
}
```

## 참고
http://forward.nhnent.com/hands-on-labs/java.spring-boot-actuator/04-endpoint.html      
https://junjangsee.tistory.com/entry/springboot-actuator
