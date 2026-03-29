## ⚙️ 중앙 설정 관리 서버 (Config Server)

Spring Cloud Config를 사용하여 우리 MSA 프로젝트의 모든 마이크로서비스 설정을 중앙에서 관리합니다. 각 서비스의 application.yml을 직접 수정하는 대신, 이 서버를 통해 설정값을 주입받습니다.

## 📍 인프라 정보

표준 포트: 8888

Service Discovery: Eureka Server (http://localhost:8761)에 자동으로 등록됩니다.

설정 저장소: [project-configs](https://github.com/FiveHandSixHand/project-configs)

### 설정 파일 규칙 (project-configs 레포지토리)

project-configs 내에 아래 구조로 파일을 배치해야 합니다.

```
project-configs
├── common/
│   └── application.yml: 모든 서비스 공통 설정 (DB, Eureka 등) (로컬/default)
│   ├── application-dev.yml         # 개발 서버 공통 설정
│   └── application-prod.yml        # 운영 서버 공통 설정
│
└── {service-name}/
    └── {service-name}.yml: 특정 서비스 기본 설정
    └── {service-name}-{profile}.yml: 특정 서비스의 특정 환경(dev, prod) 전용 설정

```

## 🚀 클라이언트 서비스 설정 가이드

본인의 서비스에서 Config Server를 연결하는 상세 가이드입니다.

### 1. 의존성 추가 (build.gradle)

```java
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-config'
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```

### 2. 프로젝트 설정 (application.yml) 예시

각 서비스는 "자신의 이름"과 "설정 서버 위치"만 알면 됩니다. 아래 내용을 복사하여 서비스명을 수정하세요.

```yml
spring:
  application:
    name: { 서비스명 } # 예: user-service
  profiles:
    active: ${ACTIVE_PROFILE:default} # 환경별 설정 분리용
  config:
    # 1순위: 직접 주소로 Config Server 접속 시도 (optional이므로 실패해도 다음 단계 진행)
    import: "optional:configserver:http://localhost:8888"
  cloud:
    config:
      discovery:
        # 2순위: 직접 접속 실패 시 Eureka를 통해 'config-server'를 찾음
        enabled: true
        service-id: config-server

eureka:
  instance:
    # 도커 환경 등을 고려하여 호스트네임 유연하게 설정
    hostname: ${HOSTNAME:localhost}
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: ${EUREKA_SERVER_URL:http://localhost:8761/eureka/}
```

### 3. 실시간 설정 갱신 (Refresh)

project-configs의 설정을 수정한 후, 서비스를 재시작하지 않고 반영하는 방법입니다.

1. Git Push: project-configs 저장소에 변경 사항을 Push합니다.

2. 설정을 사용하는 클래스 상단에 @RefreshScope를 추가합니다.

```java
// 예시
@Component
@RefreshScope
public class SomeConfig {

    @Value("${my.custom.property}")
    private String property;

    public String getProperty() {
        return property;
    }
}
```

- 위 코드에서 @RefreshScope 어노테이션을 적용한 SomeConfig 빈은, my.custom.property 값이 변경될 때마다 /actuator/refresh 엔드포인트를 호출하면 해당 값이 갱신됩니다.

3. 설정을 수정한 후 아래 API를 호출합니다

```
curl -X POST http://{service-host}:{port}/actuator/refresh

// 로컬환경
curl -X POST http://localhost:{서비스포트}/actuator/refresh
```

## 🛠️ 로컬 테스트 및 디버깅

설정 조회 확인 (API)
브라우저에서 직접 접속하여 설정이 잘 배달되는지 테스트하세요.

http://localhost:8888/{service-name}/{profile}

예: http://localhost:8888/user-service/default

예: http://localhost:8888/order-service/dev

## 📂 폴더 구조 및 데이터 보관

- postgres/init/: 컨테이너 최초 실행 시 실행될 초기 SQL 스크립트(init.sql 등)를 넣는 곳입니다.
- Data Persistence: DB 데이터는 Docker의 Named Volume(postgres_data)에 저장되므로, 컨테이너를 삭제해도 데이터는 유지됩니다.

## ⚠️ 주의사항

### 1. 실행 순서

1. Infra: infra-repo의 Docker 컨테이너(DB 등) 실행

2. Discovery: eureka-server 실행

3. Config: config-server 실행 (본 프로젝트)

4. Business: 각 마이크로서비스 실행

### 2. 우선 순위

로컬에 있는 application.yml 보다 Config Server에서 가져온 설정의 우선순위가 더 높습니다. 로컬 설정이 필요하면, config-server를 종료 후 사용하세요.

### 3. 보안

데이터베이스 비밀번호나 API 키와 같은 민감 정보는 절대 평문으로 Push하지 마세요.
