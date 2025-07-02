# 그레이스풀 셧다운(Graceful Shutdown)
- 서버나 애플리케이션이 즉시 종료되지 않고, 다음과 같은 절차를 거쳐 안전하게 종료되는 과정.

1. 신규 요청 수신 중단
2. 진행중인 작업 완료 및 대기
3. 리로스 정리(커녁센반환, 스레드 종료 등)
4. 종료

# for 
1. 데이터 무결성
2. 이중 처리 방지
3. 리소스 안전 해제


# Graceful Shutdown이 없을 때 발생하는 문제
## 요약 : Graceful shutdown 미구현 시, 중간 요청 중단으로 데이터 정합성 훼손, 리소스 누수, 클라이언트 장애 등이 발생


1. 진행 중인 요청(트랜잭션) 중단
- 클라이언트는 응답을 받지 못하거나 5xx 에러를 받음.
- 요청 재시도화된 클라이언트의 경우 계속해서 재처리 요청, 중복 처리 위험 존재.
- DB 트랜잭션이나 외부 API 호출이 중간에 롤백·타임아웃 되어 데이터 정합성 깨짐.

2. 데이터 일관성 훼손
- 트랜잭션이 정상적으로 커밋, 롤백되지 않은채 종료되면 데이터 정합성에 이슈 발생
- 메시지 큐나 이벤트 발행 단계에서 중단되면, downstream 시스템과의 데이터 동기화에도 이슈 발생

4. 리소스 누수 및 데드락, 커넥션 오프라인
- 사용중이던 커넥션 풀, 스레드 풀, 파일 핸들, DB 커넥션 등이 정상 해제되지 않음.
- 장시간 락이 걸린 채로 방치되어 다음 스타트 시에도 데드락 혹은 커넥션 부족 현상 발생
- 비정상 해제되어 커넥션 리소스가 누수되거나 타임아웃이 발생하게 되고 이후 재기동 시 커넥션풀에 잔류 세션이 있어 애플리케이션 성능 저하와 장애 유발 가능.

4. 서비스 디스커버리(로드밸런서) 측 불가피한 에러 (로드밸런서의 health체크 실패)
- 클라이언트 혹은 LB는 아직 서버가 살아있다고 판단해 요청을 보내지만, 실제 처리 불가
- 장애 감지 지연 → 트래픽 폭주로 주변 서비스까지 연쇄 장애 유발
- 트래픽 분산 과정에서 비정상 요청이 새로 할당될 가능성 존재

5. 메시지 큐·배치 작업 불완전 처리 (데이터 일관성, 정합성 훼손과 동일 context)
- Kafka, RabbitMQ Consumer가 처리 도중 중단되면 offset 커밋 누락등이 발생, (물론 HA 내구성있게 설계된경우 중단지점부터 재처리 가능, 데드레터큐등을 두어 지수텀으로 백오프를 두고 처리하는 등)
- 동일 메시지 재처리 혹은 처리 누락 발생


# 구현 방법
## 요약: “새 요청 중단 -> 기존 요청 정상 처리 -> 리소스 해제 -> 안전 종료”의 완전한 그레이스풀 셧다운을 보장

## 스프링 부트 내장 기능 사용
1. 스프링 부트 내장 기능 사용
  -   application 프로퍼티 설정파일에 HTTP커넥터 graceful shutdown 활성화
  -   server.shutdown=graceful 
2. shutdown 단계별 최대 대기 시간 설정
-   spring.lifecycle.timeout-per-shutdown-phase=30s
3. 동작 원리
동작 원리
- SIGTERM 수신 시, 먼저 새로운 커넥션 수락 중단
- 기존 요청이 최대 30초(위 설정)까지 정상 완료될 때까지 대기
- 완료되면 스프링 컨텍스트 종료 및 애플리케이션 종료

## 직접 톰캣 커넥터, 스레드풀 제어 (TomcatConnectorCustomizer, ApplicationListener을 구현) / 커스터마이징하여 종료전 대기 로직 구현
1. colose event를 받으면 새 커넥션 수락을 중단하고 (톰캣 connector.pasuse)
2. 기존 요청 처리를 대기 한다. 그리고 ThreadPoolExcecutor 을 이용, 스레드풀을 shutdown시켜 이후의 요청을 받지 않는다. (threadpool.shutdown)
3. 지정 시간동안 이전 요청에 대해 정상처리가 가능하도록 대기하고(threadPool.awaitTermination(30, TimeUnit.SECONDS)) || executor.setAwaitTerminationSeconds(30)
4. 종료전 로깅처리 등
- 스프링 컨텍스트가 닫힐 때(ContextClosedEvent) 톰캣 커넥터를 멈추고, 남은 요청이 끝날 때까지 기다린 뒤 애플리케이션을 종료한다.
-  executor.setWaitForTasksToCompleteOnShutdown(true); 백그라운드 작업이 마무리 되도록 대기. 

## 쿠버네티스 preStop + readinessProbe + terminationGracePeriodSeconds 설정
- 컨테이너 환경(Kubernetes)에서는 preStop 훅에서 sleep을 걸어 Draining 시간을 벌고, readiness probe를 failure로 만든 뒤 프로세스를 종료

## 로드밸런서, API gateway등을 활용, draining 모드 활용 
- 인스턴스 종료 신호 (SIGHUP, SIGTERM) 이 발생하면 health체크를 false로 표시하고 로드밸런서에 신규 요청, 신규 트래픽을 차단한다. 
- AWS ALB, Nginx, Istio 등에서 지원하는 Draining/DrainTimeout 설정을 사용

## 데이터베이스에서 트랜잭션 타임아웃을 명시적으로 설정하여 무한 대기로 인한 지연 방지한다.


# 정리
- “Graceful Shutdown은 ‘갑작스러운 서비스 중단 시 인-플라이트 요청을 마무리하고, 리소스를 정리하여 데이터 일관성과 운영 안정성을 확보하는’ 필수 메커니즘.
- 이를 위해 로드밸런서 Draining, Spring Boot 내장 graceful 모드, JVM Shutdown Hook, 트랜잭션 타임아웃, 메시지 브로커 정리, Kubernetes preStop 훅 등을 유기적으로 조합하여 구현해서 사용 가능.
