# AI Professor Lite Bridge API 규격 v0.1

**상태:** Draft  
**기준일:** 2026-10-07  
**영문 규격:** [BRIDGE_SPEC_EN.md](BRIDGE_SPEC_EN.md)

## 1. 문서 목적

본 문서는 AI Professor Lite가 Moodle뿐만 아니라 WordPress, Rhymix, GnuBoard와 같은 다른 플랫폼에도 동일한 방식으로 강의를 배포할 수 있도록, 각 브리지가 제공해야 하는 최소 호환 규격을 설명합니다.

여기서 말하는 브리지는 Moodle 전체 기능을 다시 구현하는 프로그램이 아닙니다. AI Professor Lite의 현재 강의 배포 과정에서 실제로 사용하는 Web Service 기능만 선택적으로 구현합니다.

이 방식을 사용하면 AI Professor Lite 본체는 기존 Moodle 배포 구조를 그대로 유지하면서도 여러 플랫폼과 연동할 수 있습니다.

## 2. Base URL과 endpoint

AI Professor Lite에는 각 브리지에서 제공하는 **Base URL**과 **API Token**을 등록하여 사용합니다.

Base URL 아래에는 다음 두 endpoint가 제공되어야 합니다.

~~~text
/webservice/rest/server.php
/webservice/upload.php
~~~

REST endpoint는 POST 요청을 처리해야 합니다.

브라우저에서 endpoint의 존재 여부를 확인할 수 있도록 GET 요청에 대해서도 404 대신 JSON 형태의 안내 또는 오류 응답을 반환하는 방식을 권장합니다.

예를 들어 다음과 같은 응답을 사용할 수 있습니다.

~~~json
{
  "exception": "moodle_exception",
  "errorcode": "invalidparameter",
  "message": "POST required."
}
~~~

## 3. 인증 방식

REST 요청은 다음 값을 사용합니다.

~~~text
wstoken=<token>
wsfunction=<function>
moodlewsrestformat=json
~~~

파일 업로드 요청은 다음 형식을 사용합니다.

~~~text
token=<token>
file_1=<multipart file>
~~~

API Token은 충분히 긴 난수 문자열을 사용하는 것을 권장합니다.

또한 토큰 비교 시에는 가능한 경우 timing-safe 비교 함수를 사용하고, 토큰 값이 로그나 HTML, 오류 메시지에 그대로 노출되지 않도록 주의해야 합니다.

## 4. 필수 Web Service function

현재 AI Professor Lite 배포 흐름을 지원하려면 최소한 다음 function이 필요합니다.

- core_webservice_get_site_info
- core_enrol_get_users_courses
- core_course_get_contents
- mod_simplevideotracker_create_activity
- mod_simplevideotracker_set_video

기존 VideoTracker와의 호환성을 유지할 필요가 있는 경우 다음 alias도 함께 제공할 수 있습니다.

- mod_videotracker_create_activity
- mod_videotracker_set_video

AI Professor Lite는 core_webservice_get_site_info 응답에 포함된 function 목록을 확인한 뒤, 사용 가능한 component를 선택할 수 있습니다.

따라서 각 브리지는 실제로 지원하는 function만 정확하게 노출하는 것이 좋습니다.

## 5. create_activity

create_activity는 새로운 강의 또는 영상 학습 activity를 생성하는 기능입니다.

### 입력 값

| 필드 | 필수 | 설명 | 기본값 |
|---|---:|---|---:|
| courseid | Y | 플랫폼에서 course에 해당하는 대상 | - |
| sectionnum | Y | section 또는 논리적인 게시 위치 | - |
| name | Y | 강의 제목 | - |
| intro | N | 강의 소개 HTML 또는 text | 빈 문자열 |
| completionpercent | N | 수료 또는 완료 기준 비율 | 80 |
| preventseeking | N | 미검증 위치로의 앞쪽 이동 제한 여부 | true |
| seektolerance | N | seek 허용 오차(초) | 3 |
| heartbeat | N | heartbeat 간격(초) | 5 |

현재 AI Professor Lite 게시기는 주로 다음 값을 전달합니다.

- courseid
- sectionnum
- name
- intro

나머지 설정은 브리지 또는 플랫폼의 기본값을 사용할 수 있습니다.

### 성공 응답

성공 응답에는 최소한 다음 정보가 포함되어야 합니다.

~~~json
{
  "success": true,
  "cmid": 123
}
~~~

가능한 경우 다음 정보도 함께 제공하는 것을 권장합니다.

- courseid
- sectionnum
- instanceid
- name
- url

cmid는 이후 set_video 호출에서 동일한 강의를 다시 찾을 수 있도록 안정적으로 유지되는 식별자여야 합니다.

플랫폼 내부에서 사용하는 실제 ID의 종류는 서로 달라도 괜찮습니다. 다만 외부에서 보이는 cmid의 의미는 일관되게 유지하는 것이 중요합니다.

## 6. 파일 업로드

/webservice/upload.php는 multipart 방식으로 영상 파일을 전달받고 JSON 배열을 반환합니다.

AI Professor Lite는 첫 번째 항목에서 itemid 또는 draftitemid 값을 읽어 이후 set_video 호출에 사용합니다.

최소 호환 응답 예시는 다음과 같습니다.

~~~json
[
  {
    "itemid": 456,
    "filename": "lecture.mp4"
  }
]
~~~

브리지 내부에서는 이 ID를 실제 임시 파일과 연결하여 관리할 수 있습니다.

임시 업로드 파일은 다음 조건을 갖는 것을 권장합니다.

- 업로드를 수행한 API Token 또는 세션과 연결합니다.
- 일정 시간 이후 자동으로 만료되도록 합니다.
- set_video에서 정상적으로 사용된 뒤에는 정리합니다.
- 임의의 경로를 직접 전달받지 않도록 합니다.

## 7. set_video

set_video는 업로드된 임시 영상을 특정 강의 activity와 연결하는 기능입니다.

### 입력 값

| 필드 | 필수 | 설명 |
|---|---:|---|
| cmid | Y | 대상 activity 식별자 |
| draftitemid | Y | upload endpoint에서 발급한 임시 파일 ID |
| duration | N | 신뢰할 수 있는 영상 길이(초) |

duration 값이 제공되는 경우에는 유한한 양수인지 확인하는 것이 좋습니다.

### 성공 응답

성공 응답은 최소한 다음 형태를 권장합니다.

~~~json
{
  "success": true,
  "cmid": 123
}
~~~

플랫폼에 따라 다음과 같은 부가 정보를 추가할 수 있습니다.

- 저장된 filename
- 파일 URL
- duration
- 파일 변경 여부
- 진도 초기화 여부
- 플랫폼 내부 instance ID

## 8. 플랫폼별 데이터 매핑

각 브리지는 공통 개념을 해당 플랫폼의 실제 자원으로 변환합니다.

| 공통 개념 | Moodle | Rhymix | WordPress | GnuBoard5 |
|---|---|---|---|---|
| Course | course | 지정 게시판 또는 module | 논리적 publish target | bo_table |
| Activity/cmid | course module id | document_srl 또는 bridge id | post_id 또는 bridge id | wr_id 또는 bridge id |
| User | user.id | member_srl | WordPress user ID | mb_id |
| Video | Moodle file area | Rhymix attachment | Media Library | board file |
| Progress | plugin table | bridge progress table | bridge progress table | bridge progress table |

플랫폼 내부 ID 형식이 서로 다른 것은 문제가 되지 않습니다.

중요한 점은 AI Professor Lite에서 호출하는 외부 contract가 가능한 한 동일하게 유지되어야 한다는 것입니다.

## 9. 오류 응답

브리지의 오류 응답은 Moodle 스타일 JSON 형식을 사용하는 것을 권장합니다.

~~~json
{
  "exception": "moodle_exception",
  "errorcode": "invalidparameter",
  "message": "..."
}
~~~

Moodle Web Service와의 호환을 위해 HTTP 200 응답 안에 오류 JSON을 반환하는 방식도 사용할 수 있습니다.

다만 운영 환경의 로그에서는 다음 오류를 구분할 수 있도록 구현하는 것이 좋습니다.

- 인증 실패
- 잘못된 입력 값
- 업로드 실패
- 대상 강의 없음
- 내부 DB 오류
- 파일 처리 오류

## 10. 중복 생성과 Idempotency

create_activity는 새로운 게시물이나 activity를 생성하기 때문에 네트워크 오류가 발생했을 때 중복 생성 위험이 있습니다.

가능한 경우 브리지에서는 external_id 또는 별도의 idempotency key를 제공하는 것을 권장합니다.

AI Professor Lite에서도 activity 생성 결과를 확인하지 못한 경우 자동으로 다시 생성하기보다는, 실제 대상 플랫폼에서 생성 여부를 먼저 확인하도록 처리하는 것이 안전합니다.

set_video 역시 동일한 파일 연결 요청이 반복되더라도 데이터가 손상되지 않도록 구현하는 것이 좋습니다.

## 11. 보안 권장 사항

브리지를 실제 서비스 환경에 적용할 경우 다음 사항을 권장합니다.

- HTTPS를 사용합니다.
- 충분히 긴 난수 API Token을 사용합니다.
- 업로드 파일의 확장자와 MIME type을 확인합니다.
- path traversal이 발생하지 않도록 파일 경로를 직접 신뢰하지 않습니다.
- 임시 업로드 파일에는 만료 및 정리 정책을 적용합니다.
- 사용자 입력 HTML은 플랫폼에 맞는 sanitization 과정을 거칩니다.
- 학습 진도는 반드시 서버에서 다시 검증합니다.
- API Token이나 내부 경로 등 민감한 정보는 로그에 남기지 않습니다.

본 규격은 플랫폼 간 호환을 위한 최소 기준이며, 실제 운영 환경에서는 각 CMS의 보안 권장 사항도 함께 적용하는 것이 좋습니다.
