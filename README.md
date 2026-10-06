# AI Professor Lite Bridge Specification

본 저장소는 **AI Professor Lite의 멀티플랫폼 배포 방식과 영상 학습 진도 추적 규칙을 일관되게 관리하기 위한 공통 규격 저장소**입니다.

AI Professor Lite는 기본적으로 Moodle과의 연동을 중심으로 설계되어 있습니다. 그러나 실제 교육 환경에서는 WordPress, Rhymix, GnuBoard와 같은 다양한 플랫폼을 함께 사용할 수 있기 때문에, 각 플랫폼에서 동일한 방식으로 강의를 배포하고 학습 진도를 관리할 수 있도록 공통 규격을 정리하였습니다.

**Status:** v0.1 Draft  
**Updated:** 2026-10-07

언어: **한국어** | [English](README_EN.md) | [简体中文](README_ZH.md) | [日本語](README_JA.md)

## 개요

AI Professor Lite 본체에 플랫폼별 예외 처리를 계속 추가하는 방식보다는, 각 플랫폼이 필요한 호환 기능을 브리지 형태로 제공하는 방식을 사용합니다.

이를 통해 AI Professor Lite는 기존 Moodle 배포 흐름을 유지하면서도, 각 플랫폼의 고유한 게시물·회원·파일 관리 방식을 그대로 활용할 수 있습니다.

~~~text
AI Professor Lite
        |
        | Moodle-compatible contract
        v
+-------------------------------+
| Platform Bridge               |
| - Moodle                      |
| - WordPress                   |
| - Rhymix                      |
| - GnuBoard5                   |
| - future adapters             |
+-------------------------------+
        |
        v
Native users / posts / files / progress
~~~

이 구조에서 AI Professor Lite는 대상 플랫폼의 내부 구조를 직접 알 필요가 없습니다. 각 브리지가 AI Professor Lite가 필요로 하는 최소한의 Moodle-compatible API와 Simple Video Tracker 동작을 구현합니다.

## 규격 문서

- [Bridge API 규격](docs/BRIDGE_SPEC.md)
- [영상 학습 진도 규격](docs/PROGRESS_SPEC.md)
- [공통 테스트 벡터](test-vectors/progress-v0.1.json)

국제 협업 시에는 영문 규격을 기준 문서로 사용하며, 한국어·중국어·일본어 문서는 동일한 내용을 설명하는 번역본으로 관리합니다. 공통 규격이 변경되는 경우 4개 언어 문서와 테스트 벡터도 함께 갱신하는 것을 원칙으로 합니다.

## 현재 구현 현황

| 구현 | 역할 | 현재 확인 기준 |
|---|---|---|
| simplevideotracker | Moodle native activity module | Moodle 기준 구현 |
| aiprof-rx | Rhymix bridge | 0.2.2 / Rhymix 2.1.3+ / PHP 7.4+ |
| aiprof-wordpress | WordPress bridge | 0.1.1 |
| aiprof-gnuboard | GnuBoard5 bridge | 0.1.4 / GnuBoard5 5.4+ / PHP 7.4+ |

위 내용은 **2026-10-07 기준 각 저장소의 구현 상태를 바탕으로 정리한 스냅샷**입니다. 이후 각 프로젝트의 버전이나 지원 범위는 변경될 수 있습니다.

## 설계 원칙

본 규격은 다음 원칙을 기준으로 합니다.

1. AI Professor Lite 본체의 플랫폼별 분기를 가능한 한 최소화합니다.
2. 각 브리지는 전체 Moodle을 재현하지 않고, 실제 배포에 필요한 API만 구현합니다.
3. 회원, 게시물, 첨부파일, 데이터베이스 등은 각 CMS의 고유 기능을 활용합니다.
4. 학습 진도는 클라이언트 값이 아니라 서버 검증 결과를 기준으로 처리합니다.
5. 동일한 재생 구간을 여러 번 시청하더라도 중복하여 진도로 계산하지 않습니다.
6. 과도한 앞으로 건너뛰기는 클라이언트와 서버 양쪽에서 제한합니다.
7. 동일한 입력에 대해 각 플랫폼이 가능한 한 동일한 진도 결과를 반환하도록 공통 테스트 벡터를 사용합니다.

## 공통 규격을 두는 이유

플랫폼별 브리지는 내부 구현 방식이 서로 다릅니다. 예를 들어 WordPress는 WordPress REST API와 Media Library를 사용하고, Rhymix는 Rhymix의 모듈·회원·첨부파일 체계를 사용하며, GnuBoard는 게시판과 회원 테이블을 기준으로 동작합니다.

이처럼 내부 구조는 달라도 다음과 같은 외부 동작은 동일하게 유지할 수 있습니다.

- AI Professor Lite에서 강의를 생성합니다.
- 브리지를 통해 대상 플랫폼에 강의를 게시합니다.
- 영상을 업로드하고 강의와 연결합니다.
- 로그인한 학습자의 시청 진도를 저장합니다.
- 반복 시청 구간은 중복 계산하지 않습니다.
- 비정상적인 앞으로 건너뛰기를 제한합니다.
- 설정된 기준 이상을 시청하면 완료 상태로 처리합니다.

따라서 이 저장소는 각 브리지의 소스 코드를 하나로 합치기 위한 프로젝트가 아니라, **서로 다른 구현이 같은 동작을 유지할 수 있도록 기준을 정의하는 역할**을 합니다.

## 변경 정책

공통 계약을 변경할 경우 다음 프로젝트에 미치는 영향을 함께 검토하는 것을 권장합니다.

- AI Professor Lite Moodle client
- Moodle Simple Video Tracker
- Rhymix bridge
- WordPress bridge
- GnuBoard5 bridge
- 4개 언어 규격 문서
- 공통 테스트 벡터

기존 브리지와 호환되지 않는 변경이 필요한 경우에는 기존 규격을 덮어쓰기보다는 새로운 spec version으로 구분하여 관리합니다.
