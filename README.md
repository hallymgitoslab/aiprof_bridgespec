# AI Professor Lite Bridge Specification

AI Professor Lite의 멀티플랫폼 배포 브리지와 영상 학습 진도 추적 동작을 일관되게 유지하기 위한 공통 규격 저장소입니다.

**Status:** v0.1 Draft  
**Updated:** 2026-10-07

언어: **한국어** | [English](README_EN.md) | [简体中文](README_ZH.md) | [日本語](README_JA.md)

## 목적

AI Professor Lite는 Moodle용 배포 흐름을 기준으로 동작합니다. WordPress, Rhymix, GnuBoard 같은 다른 플랫폼은 AI Professor Lite 본체를 플랫폼별로 분기하지 않고, 필요한 Moodle-compatible Web Service와 Simple Video Tracker 계약을 브리지에서 구현합니다.

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

## 규격 문서

- [Bridge API 규격](docs/BRIDGE_SPEC.md)
- [영상 진도 규격](docs/PROGRESS_SPEC.md)
- [공통 테스트 벡터](test-vectors/progress-v0.1.json)

영문 규격 파일을 국제 협업 시 기준 문서로 사용하고, 한·중·일 문서는 동일 규격의 번역본으로 유지합니다. 구현 변경 시 네 언어 문서와 테스트 벡터를 함께 갱신하는 것을 원칙으로 합니다.

## 현재 구현 기준

| 구현 | 역할 | 현재 확인 기준 |
|---|---|---|
| simplevideotracker | Moodle native activity module | Moodle용 기준 구현 |
| aiprof-rx | Rhymix bridge | 0.2.2 / Rhymix 2.1.3+ / PHP 7.4+ |
| aiprof-wordpress | WordPress bridge | 0.1.1 |
| aiprof-gnuboard | GnuBoard5 bridge | 0.1.4 / GnuBoard5 5.4+ / PHP 7.4+ |

이 표는 2026-10-07 저장소 상태를 기준으로 한 스냅샷입니다.

## 설계 원칙

1. AI Professor Lite의 플랫폼별 분기를 최소화합니다.
2. 브리지는 필요한 Moodle API subset만 구현합니다.
3. CMS 고유의 사용자, 게시물, 파일, DB 체계는 각 브리지가 사용합니다.
4. 진도 판정은 서버 권위(server-authoritative)를 원칙으로 합니다.
5. 반복 시청 구간은 중복 집계하지 않습니다.
6. 과도한 앞으로 건너뛰기는 서버에서 제한합니다.
7. 같은 입력에 대해 모든 브리지가 동일한 진도 결과를 내도록 공통 테스트 벡터를 사용합니다.

## 변경 정책

공통 계약을 변경할 때는 다음을 함께 검토합니다.

- AI Professor Lite Moodle client
- Moodle Simple Video Tracker
- Rhymix bridge
- WordPress bridge
- GnuBoard5 bridge
- 네 언어 규격 문서
- 공통 테스트 벡터

호환성 파괴 변경은 새 spec version으로 분리합니다.
