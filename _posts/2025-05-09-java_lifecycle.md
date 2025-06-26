---
title: "스프링 라이프사이클"
date: 2024-05-09
categories: [CS, Java, Spring, SpringBoot, Backend]
tags: [
  Spring, SpringBoot, ApplicationLifecycle, SpringContainer, IoC, DI,
  JVM, ClassLoader, JIT, GarbageCollection,
  Bean, BeanFactory, ApplicationContext, BeanDefinition, BeanPostProcessor,
  AutoConfiguration, Conditional, Starter,
  AOP, Proxy, CGLIB, JDKDynamicProxy,
  Tomcat, Netty, Servlet, WebFlux, DispatcherServlet,
  Transaction, Caching, Scheduling,
  ExecutableJAR, JarLauncher, LaunchedURLClassLoader,
  CircularDependencies, FailureAnalyzer, ApplicationEvent
]
toc: true
comments: true
---

## 서문

이 문서는 Spring Boot 애플리케이션이 `java -jar` 명령어로 실행되는 순간부터 JVM이 종료되기까지의 실행 과정에 대한 간략정리입니다

### 목차

1.  **프로젝트 준비 단계**
-   1.1 JVM 환경 설정 상세
-   1.2 클래스패스 구성 상세
-   1.3 빌드 도구 설정 및 의존성 관리
    -   1.3.1 스프링 부트 스타터의 역할
2.  **컴파일 단계**
-   2.1 소스 코드 컴파일 상세
-   2.2 어노테이션 프로세싱 상세
-   2.3 AOP 설정과 AspectJ 통합
-   2.4 정적 분석 및 테스트 실행
3.  **클래스 로딩 단계**
-   3.0 Spring Boot Executable JAR 로딩 메커니즘
-   3.1 클래스로더 계층 구성 상세
-   3.2 클래스 로딩 순서 및 처리 상세
-   3.3 동적 클래스 로딩 및 재로딩
4.  **애플리케이션 시작 준비**
-   4.1 SpringApplication 인스턴스 생성 및 초기화 상세
-   4.2 애플리케이션 실행 환경 준비 상세
    -   4.2.1 시작 실패 분석 (FailureAnalyzers)
-   4.3 로깅 시스템 초기화
5.  **스프링 컨테이너 초기화**
-   5.1 ApplicationContext 초기화 상세
-   5.2 빈 메타정보 로딩 상세 (by `ConfigurationClassPostProcessor`)
-   5.3 빈 팩토리 후처리기 실행 상세
-   5.4 조건부 구성 상세
-   5.5 BeanPostProcessor 등록 상세
-   5.6 SpEL (Spring Expression Language)
-   5.7 리소스 처리
-   5.8 메시지 소스 상세
6.  **빈 생성 및 초기화**
-   6.1 빈 인스턴스화 우선순위 및 절차
-   6.2 의존성 주입 상세 절차
    -   6.2.1 순환 참조 해결 메커니즘
-   6.3 스프링 타입 변환 시스템
-   6.4 유효성 검사 처리 상세
-   6.5 초기화 콜백 상세 절차
-   6.6 빈 스코프 관리
-   6.7 스프링 이벤트 처리 상세
-   6.8 AOP 프록시 생성 상세 절차
-   6.9 AOP 추가 설정
-   6.10 컨테이너 준비 완료 및 이벤트 발행
7.  **웹 애플리케이션 초기화**
-   7.1 내장 웹 서버 준비 상세
-   7.2 서블릿 스택 초기화 (Tomcat, DispatcherServlet 등)
    -   7.2.1 톰캣과의 상호작용 상세
    -   7.2.2 서블릿 컨텍스트 초기화 상세
    -   7.2.3 DispatcherServlet 설정 상세
-   7.3 리액티브 스택(WebFlux) 초기화 상세
    -   7.3.1 내장 리액티브 서버 준비 (Netty 등)
    -   7.3.2 DispatcherHandler 설정 상세
    -   7.3.3 WebFilter 체인 구성
-   7.4 보안 설정 상세
-   7.5 웹소켓 및 실시간 통신 설정
8.  **런타임 동작**
-   8.1 요청 처리 상세
-   8.2 트랜잭션 처리 상세
-   8.3 데이터 액세스 추가 설정
-   8.4 캐시 처리 상세
-   8.5 비동기 처리 상세
-   8.6 배치 처리 상세
-   8.7 모니터링 및 로깅
9.  **호출 시점 런타임 처리**
-   9.1 JIT 컴파일 및 최적화
-   9.2 가비지 컬렉션 상세
-   9.3 성능 최적화 및 튜닝
10. **종료 프로세스**
-   10.1 컨테이너 종료 시작 상세
-   10.2 리소스 정리 상세
-   10.3 빈 소멸 상세
-   10.4 최종 정리 상세

---

## 1\. 프로젝트 준비 단계

### 1.1 JVM 환경 설정 상세

```
1. JVM 메모리 영역 설정
   ├── Heap 영역 설정
       ├── -Xms: 초기 힙 크기 설정
       ├── -Xmx: 최대 힙 크기 설정
       ├── -XX:NewSize: Young 영역 크기 설정
       ├── -XX:MaxNewSize: 최대 Young 영역 크기 설정
       └── -XX:SurvivorRatio: Eden과 Survivor 영역 비율 설정
   ├── Non-Heap 영역 설정
       ├── -XX:MetaspaceSize: 초기 Metaspace 크기 설정
       ├── -XX:MaxMetaspaceSize: 최대 Metaspace 크기 설정
       └── -XX:CompressedClassSpaceSize: 압축된 클래스 공간 크기 설정
   └── Stack 영역 설정
       ├── -Xss: 스레드 스택 크기 설정

2. GC 설정 상세
   ├── GC 알고리즘 선택
       ├── -XX:+UseSerialGC: Serial GC 사용
       ├── -XX:+UseParallelGC: Parallel GC 사용
       ├── -XX:+UseG1GC: G1 GC 사용 (Java 9부터 기본)
       ├── -XX:+UseZGC: ZGC 사용 (Java 11 이상)
       └── -XX:+UseShenandoahGC: Shenandoah GC 사용 (Java 12 이상)
   ├── GC 튜닝 파라미터
       ├── -XX:ParallelGCThreads: 병렬 GC 스레드 수 설정
       ├── -XX:MaxGCPauseMillis: 최대 GC 일시 중지 시간 목표 설정
   └── GC 로깅 설정
       ├── -Xlog:gc*: GC 로그 상세 설정 (Java 9 이상)
       └── -Xloggc:<파일경로>: GC 로그 파일 경로 지정 (Java 8 이하)

3. 시스템 프로퍼티 설정 상세
   ├── -D<key>=<value> 형식으로 설정
       ├── java.security.egd: 난수 생성기 설정 (예: file:/dev/./urandom)
       ├── file.encoding: 파일 인코딩 설정 (예: UTF-8)
       └── user.timezone: 타임존 설정 (예: Asia/Seoul)

4. JVM 실행 옵션
   ├── 디버깅 옵션
       └── -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
   ├── 모니터링 옵션
       ├── -Dcom.sun.management.jmxremote: JMX 활성화
       └── -XX:+HeapDumpOnOutOfMemoryError: OOM 발생 시 힙덤프 생성
   └── 성능 옵션
       ├── -XX:+TieredCompilation: 계층적 컴파일 사용 (기본값)
       └── -XX:+UseStringDeduplication: G1 GC 사용 시 중복 문자열 제거
```

### 1.2 클래스패스 구성 상세

```
1. 의존성 관리 상세
   ├── 직접 의존성 검사: 필수 라이브러리 존재 확인, 버전 호환성 검사
   ├── 전이 의존성 분석: 의존성 트리 구성, 충돌 검사 및 버전 조정
   ├── 스프링 프레임워크 핵심 라이브러리 확인: spring-core, spring-beans, spring-context 등
   └── 외부 라이브러리 검사: 로깅(SLF4J, Logback), DB 드라이버, 커넥션 풀(HikariCP) 등

2. 리소스 파일 구성 상세
   ├── 설정 파일 위치 확인 (우선순위 순)
       ├── file:./config/
       ├── file:./
       ├── classpath:/config/
       └── classpath:/
   ├── 로깅 설정 파일: logback-spring.xml, log4j2-spring.xml 등
   └── 스프링 메타데이터: META-INF/spring.factories

3. 클래스패스 검증
   ├── 클래스패스 중복 검사: 동일 클래스나 리소스 파일이 여러 JAR에 포함되었는지 확인
   └── 클래스로더 구조 검증: 부모-자식 관계 및 로딩 위임 규칙 확인
```

### 1.3 빌드 도구 설정 및 의존성 관리

```
1. 빌드 도구 선택
   ├── Maven: pom.xml 기반, 플러그인 설정
   └── Gradle: build.gradle(.kts) 기반, 유연한 스크립트 작성

2. 의존성 관리
   ├── 의존성 버전 관리: spring-boot-dependencies BOM(Bill of Materials)을 통한 버전 관리
   ├── 프로파일별 의존성 설정: 개발, 테스트, 프로덕션 환경에 따라 다른 의존성 구성
   └── 저장소 설정: 사설 저장소(Nexus, Artifactory) 또는 미러 설정

3. 빌드 플러그인 설정
   ├── 컴파일러 플러그인: Java 버전, 인코딩 설정
   ├── 테스트 플러그인: JUnit, Mockito 등 설정, 테스트 커버리지 도구(JaCoCo) 연동
   └── 패키징 플러그인: spring-boot-maven-plugin/spring-boot-gradle-plugin을 사용하여 실행 가능 JAR/WAR 생성
```

#### 1.3.1 스프링 부트 스타터의 역할

```
1. 의존성 번들 관리
   ├── spring-boot-starter-web, spring-boot-starter-data-jpa 등 특정 기능에 필요한 라이브러리 모음 제공
   └── 전이 의존성(Transitive Dependencies) 자동 관리: 호환되는 버전의 라이브러리들을 자동으로 포함시켜 버전 충돌 문제를 최소화

2. 자동 구성(Auto-Configuration)의 트리거
   ├── 스타터 추가 시 클래스패스에 특정 클래스가 포함됨 (예: tomcat-embed-core.jar)
   ├── 스프링 부트는 이 클래스의 존재 여부를 감지 (@ConditionalOnClass)
   └── 해당 조건이 충족되면 관련된 자동 구성 클래스(예: ServletWebServerFactoryAutoConfiguration)가 활성화되어 필요한 빈들을 자동으로 등록
```

---

## 2\. 컴파일 단계

### 2.1 소스 코드 컴파일 상세

```
1. 컴파일 준비
   ├── 소스 파일 탐색: 소스 디렉토리 스캔, 파일 인코딩 확인
   ├── 컴파일러 설정: 자바 버전, 인코딩, 디버그 정보 포함 여부
   └── 컴파일 경로 설정: 소스 경로, 클래스 출력 경로, 리소스 출력 경로 지정

2. 컴파일 실행
   ├── 구문 분석(Parsing) -> 의미 분석(Semantic Analysis) -> 중간 코드 생성 -> 바이트코드 최적화
   └── 최종 결과물: .java 파일이 .class 파일(바이트코드)로 변환됨

3. 리소스 처리
   ├── 리소스 파일 복사: src/main/resources 디렉토리의 파일들을 빌드 출력 경로로 복사
   ├── 메타 정보 파일 복사: MANIFEST.MF, META-INF/spring.factories 등
   └── 패키징 준비: 디렉토리 구조 생성
```

### 2.2 어노테이션 프로세싱 상세

> **참고:** 이 단계는 `@Component` 등 스프링 핵심 어노테이션이 아닌, **컴파일 시점**에 실제 코드나 메타데이터 파일을 생성하는 어노테이션 프로세서의 동작을 설명합니다.

```
1. 어노테이션 프로세서 초기화
   ├── 프로세서 검색: META-INF/services/javax.annotation.processing.Processor 파일 스캔
   └── 프로세서 인스턴스 생성 및 초기화

2. 어노테이션 기반 코드 및 메타데이터 생성
   ├── @ConfigurationProperties 처리 (spring-boot-configuration-processor): META-INF/spring-configuration-metadata.json 파일 생성 (IDE 자동완성용)
   ├── Lombok 어노테이션 처리: @Getter, @Data 등을 기반으로 바이트코드에 실제 메서드 생성
   ├── QueryDSL Q클래스 생성: @Entity를 분석하여 타입-세이프 쿼리용 Q-Type 클래스(.java 파일) 생성
   ├── JPA 메타모델 생성: @Entity를 기반으로 Criteria API용 메타모델 클래스 생성
   └── MapStruct 어노테이션 처리: @Mapper 인터페이스의 구현체 클래스를 자동 생성

3. 생성된 메타데이터/소스 통합
   ├── 생성된 소스 코드(.java) 컴파일
   └── 생성된 메타데이터 파일을 빌드 출력 경로에 포함
```

### 2.3 AOP 설정과 AspectJ 통합

```
1. AOP 방식 이해
   ├── 스프링 AOP: 런타임에 프록시(Proxy)를 생성하여 부가 기능 주입 (기본 방식)
       ├── 인터페이스 기반: JDK 동적 프록시
       └── 클래스 기반: CGLIB
   ├── AspectJ 통합: 컴파일 또는 로드 타임에 바이트코드를 직접 조작 (위빙, Weaving)
       ├── 컴파일 타임 위빙 (CTW): AspectJ 컴파일러(ajc) 사용
       └── 로드 타임 위빙 (LTW): -javaagent 옵션으로 위빙 에이전트 지정
```

### 2.4 정적 분석 및 테스트 실행

```
1. 정적 코드 분석
   ├── 코드 스타일 검사: Checkstyle, PMD
   ├── 잠재적 버그 분석: SpotBugs
   └── 보안 취약점 검사: OWASP Dependency-Check

2. 테스트 실행
   ├── 단위 테스트: JUnit, Mockito를 사용하여 독립적인 단위 로직 검증
   └── 통합 테스트: Spring TestContext Framework를 사용하여 컨테이너를 띄우고 전체적인 통합 동작 검증
```

---

## 3\. 클래스 로딩 단계

### 3.0 Spring Boot Executable JAR 로딩 메커니즘

```
1. 실행 시작점 (Launcher)
   ├── java -jar my-app.jar 실행 시, MANIFEST.MF 파일에 정의된 JarLauncher가 가장 먼저 실행됨
   └── JarLauncher는 애플리케이션의 실제 main 메서드를 호출하기 전, 클래스 로딩 환경을 설정

2. 커스텀 클래스로더 생성 (LaunchedURLClassLoader)
   ├── JarLauncher는 LaunchedURLClassLoader라는 커스텀 클래스로더를 생성
   └── 이 클래스로더는 JAR 파일 내에 중첩된 JAR 파일을 로딩할 수 있음

3. 클래스패스 구성
   ├── BOOT-INF/classes/ 디렉토리를 클래스패스에 추가
   ├── BOOT-INF/lib/ 디렉토리의 모든 의존성 JAR 파일들의 경로를 클래스패스에 추가
   └── 이를 통해 애플리케이션 코드와 모든 라이브러리 클래스를 로딩할 수 있는 환경이 구성됨

4. 애플리케이션 메인 메서드 호출
   ├── 클래스 로딩 환경 설정이 완료되면, JarLauncher는 Start-Class에 지정된 애플리케이션의 main 메서드를 찾아 실행
   └── 이 시점부터 4. 애플리케이션 시작 준비 단계가 진행됨
```

### 3.1 클래스로더 계층 구성 상세

```
1. Bootstrap ClassLoader: 최상위 로더, $JAVA_HOME/lib의 핵심 JDK 클래스 로딩
2. Extension(Platform) ClassLoader: $JAVA_HOME/lib/ext의 확장 라이브러리 로딩 (Java 9+부터 Platform으로 변경)
3. Application(System) ClassLoader: -classpath 옵션이나 CLASSPATH 환경변수의 클래스 로딩
4. LaunchedURLClassLoader (Spring Boot): Application ClassLoader의 자식으로, 중첩된 JAR 내부 클래스 로딩
```

### 3.2 클래스 로딩 순서 및 처리 상세

```
1. 로딩(Loading): 클래스로더가 .class 파일을 찾아 읽고, 그 내용을 JVM 메모리(메서드 영역)에 저장
2. 링크(Linking)
   ├── 검증(Verification): 바이트코드의 유효성과 보안 검사
   ├── 준비(Preparation): 클래스의 정적(static) 변수를 위한 메모리 할당 및 기본값 초기화
   └── 해석(Resolution): 심볼릭 레퍼런스(이름)를 실제 메모리 주소로 변환
3. 초기화(Initialization): 정적 변수에 할당된 값을 대입하고, static 블록 실행
```

### 3.3 동적 클래스 로딩 및 재로딩

```
1. 동적 클래스 로딩: 리플렉션(Class.forName()), 프록시(JDK, CGLIB) 생성 시 런타임에 클래스 로딩
2. 클래스 재로딩 (개발 환경)
   ├── Spring Boot DevTools: Restart ClassLoader가 변경된 클래스만 다시 로딩하여 빠른 재시작 지원
   └── LiveReload: 브라우저와 연동하여 정적 리소스, 템플릿 변경 시 자동 새로고침
```

---

## 4\. 애플리케이션 시작 준비

### 4.1 SpringApplication 인스턴스 생성 및 초기화 상세

```
1. SpringApplication 생성: new SpringApplication(PrimarySource.class)
   ├── 웹 애플리케이션 타입 감지: 클래스패스에 Servlet/Reactive 관련 클래스가 있는지 확인하여 타입 결정
   ├── 초기화자(Initializer) 검색: META-INF/spring.factories에서 ApplicationContextInitializer 구현체 로딩
   └── 리스너(Listener) 검색: META-INF/spring.factories에서 SpringApplicationRunListener, ApplicationListener 구현체 로딩

2. SpringApplication.run() 호출: 애플리케이션 시작 프로세스 개시
```

### 4.2 애플리케이션 실행 환경 준비 상세

```
1. 리스너 통지 및 환경(Environment) 준비
   ├── starting() 호출 및 ApplicationStartingEvent 발행
   ├── Environment 객체 생성 및 시스템 환경 변수, JVM 프로퍼티, 커맨드라인 인자 로드
   ├── application.properties/yml 등 설정 파일 로딩 및 Environment에 추가
   ├── environmentPrepared() 호출 및 ApplicationEnvironmentPreparedEvent 발행

2. 배너 출력: classpath:banner.txt 또는 설정에 따라 배너를 콘솔에 출력

3. ApplicationContext 생성 및 준비
   ├── 웹 애플리케이션 타입에 맞는 ApplicationContext 구현체 선택 및 생성
   ├── contextPrepared() 호출 및 ApplicationContextInitializedEvent 발행

4. 로깅 시스템 초기화 (상세 내용은 4.3 참조)
```

#### 4.2.1 시작 실패 분석 (FailureAnalyzers)

```
1. 시작 실패 예외 발생: SpringApplication.run() 실행 중 예외 발생(포트 충돌, 빈 생성 실패 등)
2. FailureAnalyzer 실행: META-INF/spring.factories에 등록된 FailureAnalyzer 구현체들이 예외를 분석
3. 분석 리포트 출력: 사람이 이해하기 쉬운 실패 원인과 해결 방안을 로그에 출력 (예: "Port 8080 was already in use.")
```

### 4.3 로깅 시스템 초기화

```
1. 로깅 구성 로딩: 클래스패스에서 logback-spring.xml 등 로깅 설정 파일 탐색
2. 로깅 프레임워크 설정: Logback, Log4j2 등 설정 적용 (패턴, 레벨, Appender)
3. 로깅 출력 설정: 콘솔, 파일 출력 및 롤링 정책 등 구성
```

---

## 5\. 스프링 컨테이너 초기화

### 5.1 ApplicationContext 초기화 상세

> `SpringApplication.run()` 내부에서 `refreshContext()`가 호출되면서 시작. 이 과정의 핵심은 `ApplicationContext`의 `refresh()` 메서드.

```
1. 컨텍스트 준비 (prepareRefresh): 시작 시간 기록, 활성화 플래그 설정
2. BeanFactory 준비 (obtainFreshBeanFactory): BeanFactory 생성 및 설정(직렬화 등)
3. BeanFactory 후처리기 실행 전 준비 (prepareBeanFactory): 기본적인 BeanPostProcessor, 내장 빈(environment, systemProperties) 등 등록
4. BeanFactoryPostProcessor 실행 (invokeBeanFactoryPostProcessors)
    ├── 이 단계에서 ConfigurationClassPostProcessor가 실행되어 다음 5.2 단계의 작업을 수행
```

### 5.2 빈 메타정보 로딩 상세 (by `ConfigurationClassPostProcessor`)

```
1. 구성 클래스 처리 (@Configuration): @ComponentScan, @Import, @Bean 등의 어노테이션을 파싱
2. @ComponentScan 처리: 지정된 base-package를 스캔하여 @Component, @Service, @Repository, @Controller 등을 찾아 BeanDefinition으로 변환
3. @Import 처리: ImportSelector, ImportBeanDefinitionRegistrar 등을 통해 추가적인 구성 클래스나 BeanDefinition을 로딩
4. 자동 구성 클래스 처리 (@EnableAutoConfiguration): spring.factories에 정의된 자동 구성 클래스들을 로딩하고, 조건부(@Conditional) 평가를 통해 필요한 빈들을 등록
5. 최종 결과: 모든 빈에 대한 설계도인 BeanDefinition이 BeanFactory에 등록됨
```

### 5.3 빈 팩토리 후처리기 실행 상세

```
1. BeanDefinitionRegistryPostProcessor 실행: 모든 BeanDefinition이 로드된 후, 프로그래밍 방식으로 새로운 BeanDefinition을 등록/수정할 기회 제공
2. PropertySourcesPlaceholderConfigurer 처리: `${...}` 형태의 플레이스홀더를 Environment에 등록된 프로퍼티 값으로 치환
```

### 5.4 조건부 구성 상세

```
1. @Conditional 어노테이션 처리: 자동 구성 클래스나 @Bean 메서드에 붙은 조건들을 평가
2. 조건 평가: @ConditionalOnClass(클래스 존재 여부), @ConditionalOnBean(빈 존재 여부), @ConditionalOnProperty(프로퍼티 값) 등 다양한 조건을 검사
3. 조건부 빈 등록 결정: 조건이 참일 경우에만 해당 BeanDefinition을 활성화
```

### 5.5 BeanPostProcessor 등록 상세

```
1. BeanPostProcessor 등록 (registerBeanPostProcessors): BeanFactory에 정의된 모든 BeanPostProcessor 빈들을 찾아 컨테이너에 등록
2. 등록되는 핵심 후처리기들:
   ├── AutowiredAnnotationBeanPostProcessor: @Autowired, @Value 어노테이션 처리
   ├── CommonAnnotationBeanPostProcessor: @PostConstruct, @PreDestroy 어노테이션 처리
   ├── AnnotationAwareAspectJAutoProxyCreator: AOP를 위한 프록시 생성 처리
```

### 5.6 SpEL (Spring Expression Language)

```
1. 표현식 사용: @Value("#{...}"), @ConditionalOnExpression("'${my.prop}' == 'value'") 등에서 사용
2. EvaluationContext 설정: 빈 참조(예: @beanName), 시스템 프로퍼티 등에 접근할 수 있는 컨텍스트 내에서 표현식 평가
```

### 5.7 리소스 처리

```
1. ResourceLoader: ApplicationContext는 ResourceLoader를 구현하여 다양한 리소스(classpath:, file:, http:)에 일관된 방식으로 접근
```

### 5.8 메시지 소스 상세

```
1. MessageSource 초기화 (initMessageSource): 국제화(i18n)를 지원하는 MessageSource 빈 초기화
2. 메시지 파일 로딩: messages.properties, messages_ko.properties 등 로케일에 맞는 메시지 파일 로딩
```

---

## 6\. 빈 생성 및 초기화

### 6.1 빈 인스턴스화 우선순위 및 절차

```
1. 빈 인스턴스화 시작: 컨테이너 초기화의 마지막 단계(finishBeanFactoryInitialization)에서 싱글톤 빈들의 인스턴스화를 시작
2. 인스턴스 생성 전략 결정: 생성자, 팩토리 메서드(@Bean), Supplier 등을 통해 인스턴스 생성
3. 스마트 인스턴스화: SmartInstantiationAwareBeanPostProcessor가 특정 빈의 생성자 등을 미리 결정하여 최적화
```

### 6.2 의존성 주입 상세 절차

```
1. 의존성 해결 준비: @Autowired, @Inject, @Value 어노테이션이 붙은 필드나 메서드를 스캔하여 주입할 의존성 정보 수집
2. 의존성 주입 실행 (populateBean):
   ├── 생성자 주입: 인스턴스화 단계에서 이미 완료됨
   ├── 필드/세터 주입: 리플렉션을 사용하여 의존하는 빈을 찾아 주입
3. 순환 참조 처리: 아래 6.2.1 참조
```

#### 6.2.1 순환 참조 해결 메커니즘

```
1. 3단계 캐시 활용 (싱글톤 빈 & 세터 주입의 경우):
   ├── singletonFactories (3단계): 빈을 생성하는 팩토리를 저장
   ├── earlySingletonObjects (2단계): 생성은 되었으나 초기화는 안된 빈(또는 그 프록시)을 저장
   ├── singletonObjects (1단계): 완전히 생성 및 초기화된 빈을 저장
2. 동작 과정:
   ├── 빈 A 생성 시작 -> A의 팩토리를 3단계 캐시에 저장
   ├── A가 의존하는 B 생성 시작
   ├── B가 다시 A를 의존 -> 1단계 캐시에 A가 없으면, 3단계 캐시에서 A의 팩토리를 찾아 인스턴스(또는 프록시)를 생성하고 2단계 캐시에 저장 후 B에 주입
   ├── B 생성 완료 -> 1단계 캐시에 저장
   ├── 다시 A 생성 과정으로 돌아와 B를 주입받고, A 생성 완료 -> 1단계 캐시에 저장
```

### 6.3 스프링 타입 변환 시스템

```
1. ConversionService 활용: @Value로 프로퍼티 주입 시, 문자열을 숫자, Boolean 등 다른 타입으로 변환할 때 사용
2. Converter/Formatter 등록: 사용자가 커스텀 변환 로직을 추가할 수 있음
```

### 6.4 유효성 검사 처리 상세

```
1. Bean Validation (JSR-380): @Validated, @Valid 어노테이션 사용 시 Validator가 빈의 제약 조건(@NotNull, @Size 등)을 검증
```

### 6.5 초기화 콜백 상세 절차

> 빈 인스턴스가 생성되고 의존성 주입이 완료된 후, 초기화 로직이 순서대로 실행됨.

```
1. Aware 인터페이스 처리: BeanNameAware, BeanFactoryAware, ApplicationContextAware 순서로 관련 객체 주입
2. BeanPostProcessor의 postProcessBeforeInitialization 호출
3. 초기화 메서드 실행: @PostConstruct 어노테이션이 붙은 메서드 실행 -> InitializingBean의 afterPropertiesSet() 실행 -> @Bean(initMethod="...")에 지정된 메서드 실행
4. BeanPostProcessor의 postProcessAfterInitialization 호출
```

### 6.6 빈 스코프 관리

```
1. 스코프 결정: @Scope 어노테이션에 따라 singleton(기본값), prototype, request, session 등 스코프 결정
2. 스코프 프록시: 싱글톤 빈이 request나 session 스코프 빈을 주입받을 때, 실제 객체 대신 프록시 객체를 주입하여 런타임에 실제 빈을 찾도록 함
```

### 6.7 스프링 이벤트 처리 상세

```
1. ApplicationEventMulticaster 초기화: 이벤트 발행 및 리스너 전파를 담당
2. 이벤트 발행: ApplicationEventPublisher를 통해 커스텀 이벤트 발행
3. 이벤트 리스너: @EventListener 어노테이션이 붙은 메서드가 특정 이벤트 발생 시 호출됨
```

### 6.8 AOP 프록시 생성 상세 절차

```
1. 프록시 생성 시점: 초기화 콜백의 마지막 단계인 `postProcessAfterInitialization`에서 `AnnotationAwareAspectJAutoProxyCreator`가 동작
2. 프록시 생성 결정: 빈에 @Transactional과 같은 AOP 관련 어노테이션이 있거나, 포인트컷 표현식과 일치하는 경우 프록시 생성
3. 프록시 타입 결정: 대상 클래스가 인터페이스를 구현하면 JDK 동적 프록시, 그렇지 않으면 CGLIB 프록시 사용
4. 프록시 반환: 원본 빈 대신 프록시 객체가 컨테이너에 최종적으로 등록됨
```

### 6.9 AOP 추가 설정

```
1. 포인트컷 표현식 최적화: execution, @within 등 효율적인 표현식 사용
2. 어드바이스 순서: 여러 Aspect가 적용될 때 @Order 어노테이션으로 순서 제어
```

### 6.10 컨테이너 준비 완료 및 이벤트 발행

```
1. LifecycleProcessor 실행 (finishRefresh): ApplicationContext의 refresh() 마지막 단계. Lifecycle 인터페이스를 구현한 빈(예: 스케줄러)들의 start() 메서드를 호출하여 백그라운드 프로세스 시작
2. ApplicationReadyEvent 발행: 컨테이너와 모든 빈이 완전히 준비되었음을 알리는 이벤트 발행
3. Runner 실행: ApplicationRunner, CommandLineRunner 구현체들이 이 시점에 실행되어 애플리케이션 시작 시 특정 로직을 수행
```

---

## 7\. 웹 애플리케이션 초기화

### 7.1 내장 웹 서버 준비 상세

```
1. 서버 팩토리 초기화: ServletWebServerFactory(서블릿 스택) 또는 ReactiveWebServerFactory(리액티브 스택) 빈 생성
2. 서버 커스터마이징: WebServerFactoryCustomizer 빈을 통해 포트, SSL, 컨텍스트 경로, 압축 등 설정
3. 웹 서버 시작: 팩토리가 내장 웹 서버(Tomcat, Netty 등) 인스턴스를 생성하고 시작
```

### 7.2 서블릿 스택 초기화 (Tomcat, DispatcherServlet 등)

#### 7.2.1 톰캣과의 상호작용 상세

```
1. 내장 톰캣 초기화: TomcatServletWebServerFactory가 톰캣 서버 인스턴스 생성, 컨텍스트 및 커넥터 구성
2. 서블릿 컨테이너 통합: ServletContextInitializer를 통해 DispatcherServlet, 필터, 리스너 등을 프로그래밍 방식으로 톰캣에 등록
```

#### 7.2.2 서블릿 컨텍스트 초기화 상세

```
1. 필터 체인 구성: CharacterEncodingFilter, HiddenHttpMethodFilter 등 내장 필터와 Spring Security의 DelegatingFilterProxy 등 보안 필터 등록
2. 서블릿 등록: DispatcherServlet을 "/" 패턴으로 등록하여 모든 요청을 받도록 설정
```

#### 7.2.3 DispatcherServlet 설정 상세

```
1. 초기화: 웹 서버가 시작되면서 DispatcherServlet이 초기화되고, 자신만의 WebApplicationContext를 구성
2. 핵심 컴포넌트 로딩:
   ├── HandlerMapping: @RequestMapping 등을 분석하여 요청 URL과 처리할 컨트롤러 메서드를 매핑
   ├── HandlerAdapter: 매핑된 컨트롤러 메서드를 실제 호출하고, 인자 처리 및 반환값 처리 수행
   ├── ViewResolver: 컨트롤러가 반환한 뷰 이름을 실제 뷰(Thymeleaf, JSP) 객체로 변환
   ├── HttpMessageConverter: @RequestBody, @ResponseBody 처리 시 JSON, XML 등 변환
```

### 7.3 리액티브 스택(WebFlux) 초기화 상세

#### 7.3.1 내장 리액티브 서버 준비 (Netty 등)

```
1. 서버 팩토리 초기화: NettyReactiveWebServerFactory (기본값) 생성
2. 서버 커스터마이징: 포트, SSL, 스레드 모델(이벤트 루프 그룹) 등 설정
```

#### 7.3.2 DispatcherHandler 설정 상세

```
1. 중앙 처리 허브: 서블릿 스택의 DispatcherServlet에 해당하는 DispatcherHandler 빈 생성
2. 핵심 컴포넌트 구성: HandlerMapping, HandlerAdapter, HandlerResultHandler 등을 찾아 등록하여 리액티브 타입(Mono, Flux) 처리
```

#### 7.3.3 WebFilter 체인 구성

```
1. 요청/응답 필터: 서블릿 필터에 해당하는 WebFilter 인터페이스 구현체들을 등록
2. 순서 지정: @Order 어노테이션으로 필터 실행 순서 제어 (보안, 로깅 등)
```

### 7.4 보안 설정 상세

```
1. SecurityFilterChain 구성: HttpSecurity를 통해 URL 패턴별 접근 제어, 로그인/로그아웃 처리, CORS/CSRF 설정 등 구성
2. 인증 매니저 구성: AuthenticationProvider가 UserDetailsService를 통해 사용자 정보를 로딩하고, PasswordEncoder로 비밀번호를 검증
```

### 7.5 웹소켓 및 실시간 통신 설정

```
1. WebSocket 설정: @EnableWebSocketMessageBroker로 STOMP 프로토콜 기반 메시징 활성화, 엔드포인트 및 메시지 브로커 설정
2. Server-Sent Events (SSE) 설정: 컨트롤러에서 SseEmitter를 반환하여 서버-클라이언트 간 단방향 실시간 통신 구현
```

---

## 8\. 런타임 동작

### 8.1 요청 처리 상세

```
1. HTTP 요청 수신: 내장 웹 서버의 커넥터가 요청을 받아 작업 스레드에 할당
2. 필터 체인 실행: 등록된 필터(보안, 로깅 등)들이 순서대로 실행
3. DispatcherServlet(또는 DispatcherHandler) 처리:
   ├── HandlerMapping이 요청을 처리할 컨트롤러 메서드를 찾음
   ├── HandlerAdapter가 HandlerMethodArgumentResolver를 사용해 메서드 인자( @RequestBody, @PathVariable 등)를 준비
   ├── HandlerAdapter가 컨트롤러 메서드를 호출
   ├── HandlerAdapter가 HandlerMethodReturnValueHandler를 사용해 반환값( @ResponseBody, View 이름 등)을 처리
4. 뷰 렌더링 또는 응답 데이터 전송: 뷰 리졸버를 통해 HTML을 생성하거나, HttpMessageConverter를 통해 JSON/XML 응답을 생성
5. 예외 처리: @ControllerAdvice와 @ExceptionHandler를 통해 발생한 예외를 공통으로 처리
```

### 8.2 트랜잭션 처리 상세

```
1. 트랜잭션 시작: @Transactional 메서드 호출 시 AOP 프록시가 동작하여 PlatformTransactionManager를 통해 트랜잭션 시작
2. 비즈니스 로직 실행: 데이터베이스 작업 수행
3. 트랜잭션 완료: 메서드가 정상 종료되면 커밋, 예외가 발생하면 롤백
```

### 8.3 데이터 액세스 추가 설정

```
1. JPA 엔티티 리스너: @EntityListeners를 통해 엔티티의 영속성 이벤트(@PrePersist 등)를 감지하여 추가 로직 수행
2. 2차 캐시 설정: Hibernate 2차 캐시(Ehcache, Hazelcast 등)를 설정하여 엔티티 캐싱으로 DB 부하 감소
```

### 8.4 캐시 처리 상세

```
1. 캐시 추상화: @EnableCaching 활성화
2. 캐시 조작: @Cacheable(조회 및 캐싱), @CachePut(업데이트), @CacheEvict(삭제) 어노테이션을 통해 선언적으로 캐시 관리
3. 캐시 제공자: ConcurrentMap(기본값), EhCache, Redis, Caffeine 등 다양한 캐시 구현체와 연동
```

### 8.5 비동기 처리 상세

```
1. @Async 메서드 처리: @EnableAsync 활성화 후 @Async 어노테이션을 붙이면, 별도의 스레드 풀에서 메서드가 비동기적으로 실행됨
2. 스레드 풀 관리: TaskExecutor 빈을 커스터마이징하여 스레드 풀의 크기, 큐 용량 등 설정
```

### 8.6 배치 처리 상세

```
1. 스케줄링 관리: @EnableScheduling 활성화 후 @Scheduled 어노테이션으로 특정 시간에 작업 스케줄링
2. Spring Batch: 대용량 데이터 처리를 위한 Job, Step, Reader, Processor, Writer 구성. JobLauncher를 통해 배치 작업 실행
```

### 8.7 모니터링 및 로깅

```
1. 애플리케이션 모니터링 (Actuator): /actuator/health, /actuator/metrics 등 엔드포인트를 통해 애플리케이션 상태 모니터링
2. 메트릭 수집 (Micrometer): JVM, CPU, 톰캣 스레드 등 다양한 메트릭을 수집하여 Prometheus, Grafana 등 외부 시스템과 연계
3. 분산 추적 (Spring Cloud Sleuth/Zipkin): 마이크로서비스 환경에서 요청의 흐름을 추적
```

---

## 9\. 호출 시점 런타임 처리

### 9.1 JIT 컴파일 및 최적화

```
1. JIT 컴파일 시작: JVM이 자주 호출되는 "핫스팟" 코드를 인터프리터 모드에서 네이티브 코드로 컴파일
2. 코드 최적화: 인라이닝(메서드 호출 제거), 탈출 분석(객체 스택 할당) 등 고급 최적화 수행
3. 네이티브 코드 실행: 최적화된 네이티브 코드로 대체 실행하여 성능 향상
4. 참고: AOT(Ahead-of-Time) 컴파일
   - GraalVM Native Image를 사용하면, JIT과 달리 빌드 시점에 네이티브 실행 파일로 컴파일
   - 시작 속도를 크게 향상시키고 메모리 사용량을 줄임 (Spring Native / Boot 3+ 지원)
```

### 9.2 가비지 컬렉션 상세

```
1. GC 알고리즘: G1 GC(Java 9+ 기본) 등 서버 환경에 맞는 GC 알고리즘 선택
2. Young/Old Generation GC: Minor GC(Young 영역)와 Major GC(Old 영역)가 발생하며 더 이상 참조되지 않는 객체 메모리 회수
```

### 9.3 성능 최적화 및 튜닝

```
1. JVM 옵션 튜닝: 힙 크기, GC 옵션 등을 애플리케이션 특성에 맞게 조정
2. 애플리케이션 레벨 최적화: 불필요한 객체 생성 최소화, 적절한 자료구조 선택, 알고리즘 개선
3. 모니터링 및 프로파일링: APM 도구나 프로파일러(JProfiler, VisualVM)를 사용하여 병목 지점 분석
```

---

## 10\. 종료 프로세스

### 10.1 컨테이너 종료 시작 상세

```
1. 종료 시그널 처리: SIGTERM, SIGINT 등 시스템 종료 시그널 수신 시 JVM의 Shutdown Hook이 동작
2. 그레이스풀 셧다운 (Graceful Shutdown):
   ├── 신규 요청 수락 중단
   └── 진행 중인 요청이 완료될 때까지 대기 (타임아웃 설정 가능)
3. 이벤트 발행: ContextClosedEvent가 발행되어 종료 프로세스가 시작됨을 알림
```

### 10.2 리소스 정리 상세

```
1. 데이터 액세스 리소스: 커넥션 풀, 메시지 브로커 연결 등 외부 리소스 연결 종료
2. 백그라운드 스레드: 스케줄러, 비동기 작업 스레드 풀 등 종료
```

### 10.3 빈 소멸 상세

```
1. 빈 소멸 준비: 의존성의 역순으로 빈 소멸 순서 결정
2. 소멸 메서드 실행: @PreDestroy 어노테이션이 붙은 메서드 실행 -> DisposableBean의 destroy() 실행 -> @Bean(destroyMethod="...")에 지정된 메서드 실행
```

### 10.4 최종 정리 상세

```
1. 로깅 시스템 종료: 로그 버퍼를 모두 비우고(flush), 로그 파일 핸들러 종료
2. JVM 종료: 모든 정리 작업이 완료된 후 JVM 프로세스 종료
```
