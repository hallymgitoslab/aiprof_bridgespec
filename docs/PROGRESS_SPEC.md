# 영상 학습 진도 규격 v0.1

**상태:** Draft  
**기준일:** 2026-10-07  
**영문 규격:** [PROGRESS_SPEC_EN.md](PROGRESS_SPEC_EN.md)

## 1. 목표

Moodle, Rhymix, WordPress, GnuBoard5에서 동일한 시청 입력에 대해 가능한 한 동일한 검증 진도 결과를 만들기 위한 공통 baseline입니다.

이 규격은 각 플랫폼의 저장 API를 통일하지 않습니다. 판정 의미만 통일합니다.

## 2. 기본 값

- heartbeat: 5초
- seek grace/tolerance: 3초
- completion threshold: 80%
- 로그인 사용자만 개인 진도를 저장하는 것을 기본으로 함

플랫폼 설정으로 변경할 수 있지만 테스트 시 사용한 값은 명시해야 합니다.

## 3. 서버 권위

클라이언트가 보낸 current_time을 그대로 누적 시청시간으로 신뢰해서는 안 됩니다.

서버는 최소 다음 값을 유지합니다.

- watched_ranges
- watched_seconds
- frontier
- last_position
- last_heartbeat
- completed

## 4. Heartbeat 검증

기존 진도 레코드가 없으면 첫 요청은 기준점만 만들고 검증 시청시간을 증가시키지 않는 것을 baseline으로 합니다.

기존 레코드가 있고 playing=true이면:

~~~text
max_forward = last_position + max(1, elapsed_seconds) + seek_grace
allowed_position = min(client_current_time, max_forward)
~~~

allowed_position이 last_position보다 클 때만 해당 구간을 watched_ranges에 추가합니다.

playing=false이면 paused heartbeat만으로 검증 진도가 증가하면 안 됩니다.

~~~text
allowed_position = min(client_current_time, frontier + seek_grace)
~~~

paused 요청은 위치를 검증 범위 안으로 되돌릴 수 있지만 watched range를 새로 추가하지 않습니다.

## 5. 구간 병합

각 구간은 duration 범위로 clamp합니다.

잘못된 구간이나 end <= start인 구간은 버립니다.

정렬 후 겹치거나 인접한 구간을 병합합니다. baseline 구현은 다음 조건을 사용합니다.

~~~text
next.start <= current.end + 1
~~~

watched_seconds는 병합 완료된 구간 길이 합입니다. 따라서 반복 시청은 중복 집계하지 않습니다.

## 6. Frontier

frontier는 영상 0초부터 끊김 없이 검증된 가장 먼 위치입니다.

다음 구간 시작점이 현재 frontier + 1보다 크면 연속성이 끊긴 것으로 봅니다.

클라이언트는 frontier를 이용해 과도한 앞으로 이동을 즉시 되돌릴 수 있지만 최종 권위는 서버 응답입니다.

## 7. 완료 판정

~~~text
percentage = watched_seconds / duration * 100
completed = percentage >= completion_threshold
~~~

duration이 0 또는 유효하지 않으면 완료로 판정하지 않습니다.

## 8. Seek 처리

클라이언트는 사용자가 frontier + tolerance보다 앞으로 이동하려 하면 frontier 근처로 복귀시키는 것을 권장합니다.

하지만 클라이언트 seek 제한만으로 보안을 보장해서는 안 됩니다. 개발자 도구나 직접 API 호출을 가정하고 서버에서 동일한 이동량 검증을 해야 합니다.

## 9. 재생/일시정지/종료

- play: 즉시 상태 동기화 후 heartbeat 시작
- pause: heartbeat 중지 후 상태 동기화
- ended: heartbeat 중지 후 최종 동기화
- pagehide/beforeunload: 가능한 범위에서 keepalive 또는 최종 전송

클라이언트 이벤트 손실이 발생해도 서버가 과도한 진도 증가를 허용하지 않는 것이 중요합니다.

## 10. 미디어 변경

같은 activity의 영상 내용이 변경되면 기존 진도가 새 영상에 잘못 적용되지 않도록 reset 또는 명확한 migration 정책이 필요합니다.

Moodle Simple Video Tracker 기준 구현은 영상 content 변경 시 진도 reset을 수행하고, 같은 내용에서 duration만 변경된 경우 재계산할 수 있습니다.

Bridge 구현도 이 의미를 따르는 것을 권장합니다.

## 11. 동시성

동일 사용자/동일 강의에 여러 heartbeat가 겹칠 수 있습니다.

구현은 lost update가 발생하지 않도록 DB transaction, row lock, application lock 또는 동등한 직렬화 수단을 사용해야 합니다.

Moodle 기준 구현처럼 미디어 교체와 heartbeat가 동시에 실행되는 경우도 고려해야 합니다.

## 12. 공통 테스트

[test-vectors/progress-v0.1.json](../test-vectors/progress-v0.1.json)의 시나리오를 각 브리지 테스트에 재사용합니다.

플랫폼별 저장 형식은 달라도 expected watched_seconds, frontier, position, completed, percentage 의미는 같아야 합니다.
