# AI Professor Lite Bridge API 규격 v0.1

**상태:** Draft  
**기준일:** 2026-10-07  
**영문 규격:** [BRIDGE_SPEC_EN.md](BRIDGE_SPEC_EN.md)

## 1. 범위

이 규격은 AI Professor Lite가 Moodle 이외 플랫폼에도 동일한 배포 흐름을 사용할 수 있도록 브리지가 제공해야 하는 최소 호환 계약을 정의합니다.

브리지는 Moodle 전체를 구현하지 않습니다. AI Professor Lite의 현재 게시 흐름에 필요한 endpoint와 function만 구현합니다.

## 2. Base URL과 endpoint

AI Professor Lite에는 Bridge Base URL과 API token을 등록합니다.

Base URL 아래에는 다음 상대 경로가 있어야 합니다.

~~~text
/webservice/rest/server.php
/webservice/upload.php
~~~

REST endpoint는 POST를 처리해야 합니다. GET 요청에 대해 오류 JSON을 반환할 수 있으나, 브리지 존재 확인이 가능하도록 404가 아닌 응답을 권장합니다.

## 3. 인증

REST 호출:

~~~text
wstoken=<token>
wsfunction=<function>
moodlewsrestformat=json
~~~

업로드 호출:

~~~text
token=<token>
file_1=<multipart file>
~~~

토큰 비교는 timing-safe 방식 사용을 권장합니다. 토큰을 로그, HTML, 오류 메시지에 노출해서는 안 됩니다.

## 4. 필수 function

브리지는 최소 다음 function을 제공해야 합니다.

- core_webservice_get_site_info
- core_enrol_get_users_courses
- core_course_get_contents
- mod_simplevideotracker_create_activity
- mod_simplevideotracker_set_video

이전 VideoTracker 호환성을 위해 아래 alias를 선택적으로 제공할 수 있습니다.

- mod_videotracker_create_activity
- mod_videotracker_set_video

AI Professor Lite는 site info에 노출된 function 목록을 확인하여 videotracker 또는 simplevideotracker component를 선택할 수 있습니다.

## 5. create_activity

입력:

| 필드 | 필수 | 의미 | 기본값 |
|---|---:|---|---:|
| courseid | Y | 플랫폼에서 course로 매핑되는 대상 | - |
| sectionnum | Y | section 또는 논리적 게시 위치 | - |
| name | Y | 강의 제목 | - |
| intro | N | 강의 소개 HTML/text | 빈 문자열 |
| completionpercent | N | 완료 기준 | 80 |
| preventseeking | N | 미검증 구간 앞으로 이동 제한 | true |
| seektolerance | N | 허용 오차(초) | 3 |
| heartbeat | N | heartbeat 간격(초) | 5 |

현재 AI Professor Lite 게시기는 courseid, sectionnum, name, intro를 사용합니다. 나머지 값은 플랫폼 구현에서 기본값을 사용할 수 있습니다.

성공 응답은 최소 다음 정보를 제공해야 합니다.

~~~json
{
  "success": true,
  "cmid": 123
}
~~~

권장 추가 필드: courseid, sectionnum, instanceid, name, url.

cmid는 브리지 내부에서 강의를 다시 찾을 수 있는 안정적인 activity identifier여야 합니다.

## 6. 파일 업로드

/webservice/upload.php는 multipart 업로드를 받고 JSON 배열을 반환합니다.

AI Professor Lite는 첫 항목에서 itemid 또는 draftitemid를 읽습니다.

최소 호환 예:

~~~json
[
  {
    "itemid": 456,
    "filename": "lecture.mp4"
  }
]
~~~

draft ID는 set_video가 완료될 때까지 해당 token과 연결되어야 합니다. 임시 업로드는 만료 정책을 가져야 합니다.

## 7. set_video

입력:

| 필드 | 필수 | 의미 |
|---|---:|---|
| cmid | Y | 대상 activity identifier |
| draftitemid | Y | upload endpoint가 발급한 임시 파일 ID |
| duration | N | 신뢰 가능한 영상 길이(초) |

duration이 주어지면 유한한 양수인지 검증해야 합니다.

성공 응답은 최소 다음 형태를 권장합니다.

~~~json
{
  "success": true,
  "cmid": 123
}
~~~

플랫폼별 구현은 저장 filename, URL, duration, 변경 여부 등의 메타데이터를 추가할 수 있습니다.

## 8. 플랫폼 매핑

브리지는 공통 계약을 각 플랫폼 자원에 매핑합니다.

| 공통 개념 | Moodle | Rhymix | WordPress | GnuBoard5 |
|---|---|---|---|---|
| Course | course | 지정 게시판/module | 논리적 publish target | bo_table |
| Activity/cmid | course module id | document_srl 또는 bridge id | post_id 또는 bridge id | wr_id 또는 bridge id |
| User | user.id | member_srl | WP user ID | mb_id |
| Video | Moodle file area | Rhymix attachment | Media Library | board file |
| Progress | plugin table | bridge progress table | bridge progress table | bridge progress table |

정확한 내부 ID 형식은 구현별로 다를 수 있지만 외부 contract는 안정적으로 유지해야 합니다.

## 9. 오류

Moodle 스타일 오류 JSON 사용을 권장합니다.

~~~json
{
  "exception": "moodle_exception",
  "errorcode": "invalidparameter",
  "message": "..."
}
~~~

HTTP 200 안에 오류 JSON을 넣는 Moodle 호환 동작을 사용할 수 있지만, 브리지 내부 오류와 인증 오류를 로그에서 구분할 수 있어야 합니다.

## 10. Idempotency

create_activity는 본질적으로 중복 생성 위험이 있습니다. 브리지는 가능하면 external_id 또는 자체 idempotency key를 지원해야 합니다.

AI Professor Lite는 생성 결과가 불확실한 경우 자동 재생성보다 사용자 확인을 우선합니다.

set_video는 같은 activity에 같은 파일을 다시 연결해도 데이터가 손상되지 않도록 구현하는 것을 권장합니다.

## 11. 보안

- HTTPS 권장
- 장기 난수 token 사용
- 업로드 확장자와 MIME 검증
- 경로 조작 방지
- 임시 파일 만료 및 정리
- 사용자 입력 HTML sanitization
- 서버 측 진도 검증
- 비밀정보 로그 금지
