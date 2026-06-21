
---

# Kafka 중복·유실·지연 설계와 메시지 전송 보장 방식 적용 검토

## 1. 메시지 전송 보장 방식 정리

### 1-1. At most once (중복 X, acks = 0 - 응답을 기다리지 않음)
메시지를 최대 한 번만 처리하는 방식입니다.

Consumer가 메시지를 읽고 offset을 먼저 commit한 뒤 실제 DB 반영 전에 애플리케이션이 종료되면, Kafka 입장에서는 이미 처리된 메시지로 간주하기 때문에 재처리되지 않습니다.

중복 가능성은 낮지만 장애 상황에서 유실이 발생할 수 있습니다. 로그 수집, 모니터링 이벤트처럼 일부 데이터가 누락되어도 전체 서비스의 흐름이 깨지지 않는 곳에는 사용할 수 있지만, 결제 상태 변경처럼 반드시 처리되어야 하는 업무에는 적합하지 않습니다.

### 1-2. At least once (중복 허용, retry 수행, (acks 1, all))
메시지를 적어도 한 번 이상 처리하는 방식입니다.

Consumer가 DB 반영은 성공했지만 offset commit 전에 장애가 발생하면, 같은 메시지를 다시 읽어 중복 처리할 수 있습니다. 유실 가능성은 줄어드는 대신 중복 처리를 전제로 하기 때문에, Consumer 로직에 멱등성 설계가 반드시 필요합니다.

### 1-3. Exactly once (중복 X, retry를 수행하나 중복이 제거됨)
이름만 보면 메시지가 물리적으로 정확히 한 번만 전달되는 것처럼 보이지만, 실무적으로는 **최종 비즈니스 결과가 한 번만 반영되도록 만드는 것**에 가깝다고 이해해야 합니다. 기본적으로 멱등성이 보장되어야 합니다.

Kafka의 idempotent producer나 transaction은 특정 구간의 보장을 강화할 수 있지만, Consumer가 DB를 업데이트하거나 메일/알림을 발송하는 과정까지 자동으로 보장해주지는 않습니다. 따라서 exactly once를 구현할 때는 어떤 경계 안에서의 보장인지를 명확히 해야 합니다.

- **Kafka 내부 (Producer → Broker) 보장:** 3.0 버전 이후부터는 `enable.idempotence = true`가 기본값입니다. 프로듀서는 메시지를 보낼 때 Producer ID와 sequence number를 헤더에 담습니다. 브로커는 수신한 메시지의 sequence가 기존 시퀀스보다 정확히 1만큼 큰 경우에만 저장하며, 이미 존재하는 시퀀스인 경우 로그를 저장하지 않고 ack만 보냅니다.
- **DB 반영까지 포함하는지 여부**
- **알림/메일 같은 외부 side effect까지 포함하는지 여부**

---

## 2. 내 서비스에 적용할 흐름과 판단

### At most once (중복 X, 유실 가능)

- **이메일 회사:** firebase 모바일 알림 (알림 유실 가능)
- **보안 회사:** 
    - (after commit 후 저장) 진단 종료 후 자동 조치 요청을 만드는 후처리
    - (after commit 후 저장) 감사 로그 기록

### At least once (중복이 허용되는 경우)

- **이메일 회사:** 
    - SMTP 메일 발송 큐: 전송 처리 결과가 `temporary_failure`이고 재시도가 가능한 경우 `result_error`를 반환합니다. 큐 프로세서는 retry 횟수와 다음 retry delay를 기록한 후 메시지를 재처리합니다. 영구 실패하거나 재시도 한도를 초과한 경우 실패 안내 메일을 보내거나 failed 디렉터리(Kafka의 DLQ 기능과 유사)로 이동합니다.
- **보안 회사:** 
    - 결제 후 알림 발송: 중복이 허용되는 상대측 메일 서버 응답이 늦게 와서 timeout이 발생하고 이를 실패로 인지하는 경우입니다. 재시도 로직이 실행되어 동일 알림이 5번 이상 가는 현상이 발생할 수 있으며, 이는 timeout 설정을 늘려서 해결합니다.

### Exactly once

- **이메일 회사:** 
    - 검색 인덱스: Insert/Update/Delete 작업이 mailUid 기반으로 처리되어 멱등합니다. 멱등성은 보장되지만 이를 기술적인 exactly once 사례로 볼 수 있는지는 추가 검토가 필요합니다.

---

## 3. Retry / DLQ / Idempotency 기준

### 3-1. Retry 대상
일시적인 오류는 Retry 대상으로 분류합니다.
- DB 커넥션 일시 오류
- 네트워크 timeout
- 외부 API 일시 장애
- 일시적인 lock 경합

무한 재시도를 방지하기 위해 재시도 횟수를 제한하고, exponential backoff + jitter를 적용해 재시도 폭풍을 방지해야 합니다. 
(상황에 따라 이진 백오프를 적용할지, 일정 시간 이후 재시도를 적용할지에 대한 기준 필요)

### 3-2. DLQ 대상
계속 재시도해도 성공 가능성이 낮은 메시지는 DLQ로 분리합니다.
- 필수 값 누락
- 이미 삭제된 대상에 대한 이벤트
- 비즈니스 상태상 처리 불가능한 이벤트

DLQ로 분리 후 운영자가 원인을 확인하고 데이터 보정 또는 수동 재처리를 해야 합니다. 실패 이벤트에는 추적이 가능한 정보가 남아야 합니다.

---

## 4. Producer 의 재전송 동작 관련 주요 파라미터

- **delivery.timeout.ms >= linger.ms + request.timeout.ms**

- **max.block.ms**
    - `send()` 호출 시 Record Accumulator에 입력하지 못하고 block 되는 최대 초과 시간입니다. 초과 시 Timeout Exception이 발생하며, 샌더 스레드까지 도달하지 못합니다.

- **linger.ms**
    - Sender Thread가 Record Accumulator에서 배치별로 메시지를 가져가기 위한 최대 대기 시간입니다.

- **request.timeout.ms**
    - 전송에 걸리는 최대 시간입니다. (전송 재시도 대기 시간인 `linger.ms`, `retry.backoff.ms`는 제외)
    - 초과 시 retry를 수행하거나 Timeout Exception이 발생합니다.
    - 샌더 스레드는 브로커의 응답이 없을 경우 이 시간만큼 기다리며, 이 시간 안에 응답이 와야 메시지 재전송을 하지 않습니다.

- **retry.backoff.ms**
    - 전송 재시도를 위한 대기 시간입니다.

- **delivery.timeout.ms**
    - Producer 메시지(배치) 전송에 허용된 최대 시간입니다.
    - 설정된 retry 횟수만큼 재전송을 시도하다가 이 시간이 초과되면 재전송을 중지하고 Timeout Exception을 발생시킵니다.
