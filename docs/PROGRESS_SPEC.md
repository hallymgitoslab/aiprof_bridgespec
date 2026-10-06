# 영상 학습 진도 규격 v0.1

**상태:** Draft  
**기준일:** 2026-10-07  
**영문 규격:** [PROGRESS_SPEC_EN.md](PROGRESS_SPEC_EN.md)

## 1. 문서 목적

본 문서는 Moodle, Rhymix, WordPress, GnuBoard5에서 동일한 영상을 시청했을 때 가능한 한 동일한 학습 진도 결과를 얻을 수 있도록 공통 판정 기준을 설명합니다.

각 플랫폼의 데이터베이스 구조나 API를 동일하게 만드는 것이 목적은 아닙니다. 플랫폼마다 저장 방식은 다르게 구현할 수 있으며, **시청 구간을 어떤 방식으로 검증하고 진도율을 계산할 것인지에 대한 의미를 통일하는 것**이 목적입니다.

## 2. 기본 설정

현재 기준값은 다음과 같습니다.

- heartbeat: 5초
- seek grace/tolerance: 3초
- completion threshold: 80%
- 개인별 진도는 기본적으로 로그인한 사용자에 대해서만 저장합니다.

각 플랫폼에서 설정값을 변경할 수는 있으나, 공통 테스트를 수행할 때에는 어떤 값을 사용했는지 명확하게 기록하는 것이 좋습니다.

## 3. 서버 검증을 기준으로 하는 이유

영상 플레이어의 current_time 값은 브라우저에서 전달되는 값이기 때문에 그 자체를 누적 시청시간으로 신뢰해서는 안 됩니다.

예를 들어 사용자가 개발자 도구를 사용하거나 API를 직접 호출할 경우 실제로 영상을 시청하지 않고도 큰 current_time 값을 전달할 수 있습니다.

따라서 서버는 최소한 다음과 같은 상태를 기준으로 진도를 계산하는 것이 좋습니다.

- watched_ranges
- watched_seconds
- frontier
- last_position
- last_heartbeat
- completed

최종 학습 진도는 클라이언트가 주장하는 값이 아니라 서버가 검증한 결과를 기준으로 처리합니다.

## 4. Heartbeat 검증

새로운 사용자의 진도 레코드가 아직 없는 경우에는 첫 요청에서 기준 상태만 생성하고, 해당 요청만으로 검증된 시청시간을 증가시키지 않는 방식을 기본 동작으로 사용합니다.

기존 레코드가 있고 playing=true인 경우에는 다음 계산을 기준으로 허용 가능한 위치를 구합니다.

~~~text
max_forward = last_position + max(1, elapsed_seconds) + seek_grace
allowed_position = min(client_current_time, max_forward)
~~~

allowed_position이 last_position보다 큰 경우에만 해당 구간을 watched_ranges에 추가합니다.

이 방식은 heartbeat 사이에 실제로 경과한 시간보다 지나치게 먼 위치가 전달되는 경우, 서버가 허용 범위까지만 인정하도록 하기 위한 것입니다.

### 일시정지 상태

playing=false인 요청만으로 검증된 시청시간이 증가해서는 안 됩니다.

기본적으로 다음 범위까지만 위치를 허용합니다.

~~~text
allowed_position = min(client_current_time, frontier + seek_grace)
~~~

일시정지 상태에서는 저장 위치를 이미 검증된 구간 안으로 조정할 수 있지만, 새로운 watched range를 추가하지 않습니다.

## 5. 시청 구간 병합

watched_ranges에 저장되는 각 구간은 영상 duration 범위 안으로 제한합니다.

다음과 같은 구간은 유효하지 않은 값으로 처리합니다.

- 형식이 잘못된 구간
- end가 start보다 작거나 같은 구간
- duration 범위를 벗어난 값

정상 구간은 시작 위치를 기준으로 정렬한 뒤, 서로 겹치거나 인접한 구간을 하나로 병합합니다.

현재 기준 구현에서는 다음 조건을 사용합니다.

~~~text
next.start <= current.end + 1
~~~

watched_seconds는 병합이 완료된 구간들의 실제 길이를 합산하여 계산합니다.

따라서 같은 부분을 여러 번 반복해서 시청하더라도 해당 구간이 중복하여 진도에 포함되지는 않습니다.

## 6. Frontier

frontier는 **영상의 0초부터 끊기지 않고 연속적으로 검증된 가장 먼 위치**를 의미합니다.

예를 들어 다음과 같은 시청 구간이 있다고 가정할 수 있습니다.

~~~text
[0, 30]
[50, 70]
~~~

이 경우 전체 watched_seconds에는 두 구간이 포함될 수 있지만, 30초와 50초 사이가 비어 있기 때문에 frontier는 30초 근처에서 멈춥니다.

기본적으로 다음 구간의 시작 위치가 현재 frontier + 1보다 크면 연속성이 끊긴 것으로 판단합니다.

클라이언트는 이 frontier 값을 사용하여 사용자가 지나치게 앞으로 이동했을 때 영상 위치를 다시 되돌릴 수 있습니다. 다만 최종 판단은 항상 서버 응답을 기준으로 합니다.

## 7. 진도율과 완료 판정

진도율은 다음 방식으로 계산합니다.

~~~text
percentage = watched_seconds / duration * 100
~~~

완료 여부는 다음 조건으로 판정합니다.

~~~text
completed = percentage >= completion_threshold
~~~

예를 들어 completion threshold가 80%라면 전체 영상 길이 중 검증된 시청시간이 80% 이상인 경우 완료 상태가 됩니다.

duration 값이 0이거나 유효하지 않은 경우에는 완료 상태로 처리하지 않습니다.

## 8. 앞으로 건너뛰기 처리

클라이언트에서는 사용자가 frontier + tolerance보다 먼 위치로 이동하려 할 때 기존 검증 위치로 되돌리는 방식을 권장합니다.

다만 브라우저의 JavaScript만으로는 보안을 보장할 수 없습니다.

따라서 서버에서도 동일하게 heartbeat 경과시간과 이전 검증 위치를 기준으로 허용 가능한 최대 이동 범위를 계산해야 합니다.

이 구조를 사용하면 JavaScript를 수정하거나 API를 직접 호출하더라도 서버가 비정상적인 진도 증가를 제한할 수 있습니다.

## 9. 재생 이벤트 처리

클라이언트는 일반적으로 다음 흐름으로 동작하는 것을 권장합니다.

- **play**: 현재 상태를 한 번 동기화한 뒤 heartbeat를 시작합니다.
- **pause**: heartbeat를 중지하고 현재 상태를 동기화합니다.
- **ended**: heartbeat를 중지하고 마지막 상태를 전송합니다.
- **pagehide / beforeunload**: 가능한 환경에서는 keepalive 또는 마지막 전송을 시도합니다.

네트워크 상태나 브라우저 종료로 인해 일부 이벤트가 서버에 전달되지 않을 수도 있습니다.

따라서 일부 heartbeat가 누락되더라도 서버가 과도한 진도 증가를 허용하지 않도록 설계하는 것이 중요합니다.

## 10. 영상 변경 시 진도 처리

같은 activity에 연결된 영상이 다른 영상으로 교체되는 경우, 기존 학습 진도를 그대로 유지하면 새로운 영상에 이전 진도가 잘못 적용될 수 있습니다.

따라서 영상 내용이 실제로 변경되었을 경우에는 기존 진도를 초기화하거나, 명확한 migration 정책을 적용해야 합니다.

Moodle Simple Video Tracker 기준 구현에서는 영상 content가 변경된 경우 기존 진도를 reset합니다.

반면 동일한 영상에서 duration 값만 변경된 경우에는 기존 watched range를 기준으로 진도율을 다시 계산할 수 있습니다.

다른 플랫폼의 Bridge에서도 가능한 한 같은 의미를 유지하는 것을 권장합니다.

## 11. 동시 요청 처리

동일한 사용자와 동일한 강의에 대해 여러 heartbeat가 거의 동시에 서버에 도착할 수 있습니다.

이 경우 단순히 기존 값을 읽은 뒤 다시 저장하면 한 요청의 결과가 다른 요청에 의해 덮어써지는 lost update가 발생할 수 있습니다.

따라서 각 구현에서는 환경에 따라 다음과 같은 방법을 사용하는 것이 좋습니다.

- DB transaction
- row lock
- application lock
- 동일 사용자·강의 단위의 직렬화 처리

또한 관리자가 영상을 교체하는 순간 학습자의 heartbeat가 동시에 도착하는 상황도 고려해야 합니다.

Moodle Simple Video Tracker 기준 구현에서는 미디어 변경 작업과 heartbeat가 서로 충돌하지 않도록 별도의 잠금과 transaction을 사용합니다.

## 12. 공통 테스트

각 브리지에서는 [progress-v0.1.json](../test-vectors/progress-v0.1.json)에 정의된 테스트 시나리오를 재사용하는 것을 권장합니다.

플랫폼마다 데이터 저장 방식은 달라도 다음 값의 의미는 동일해야 합니다.

- watched_seconds
- frontier
- position
- completed
- percentage

공통 테스트를 유지하면 한 플랫폼에서 진도 알고리즘을 수정했을 때 다른 플랫폼에서도 동일한 결과가 유지되는지 비교하기 쉬워집니다.
