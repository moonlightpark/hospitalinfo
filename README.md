# hospitalinfo

굿닥(Goodoc) Android 앱(`com.ksncho.hospitalinfo` v50.14.29423)을 정적 분석해 재구성한 소프트웨어 설계도입니다.

**[페이지로 이동](https://moonlightpark.github.io/hospitalinfo/docs/gooddoc-android-blueprint.html)** (GitHub Pages)

- 설계도: [docs/gooddoc-android-blueprint.html](docs/gooddoc-android-blueprint.html) (브라우저로 열면 다이어그램이 렌더링됩니다)

## 담긴 내용 (상세판 v2, 14개 장)

1. 개요
2. 시스템 구성도 (굿닥 서버와 외부 서비스, GraphQL split link)
3. 앱 레이어 아키텍처 (React Native + Hermes)
4. 기능 도메인별 화면 구성 (화면 168개)
5. 내비게이션·딥링크 라우팅 (`goodoc://` 경로 템플릿)
6. 서버 API·환경설정 (엔드포인트, 번들 설정 키)
7. GraphQL 데이터 모델 (핵심 엔티티 ERD, enum, 연산 카탈로그 197개)
8. 핵심 흐름 (접수, 비대면 진료, 토큰 회전)
9. 자체 네이티브 모듈
10. 결제·실시간·OTA (PortOne v2, WebSocket, CodePush)
11. 권한과 보안
12. 외부 SDK
13. 분석 이벤트 (Amplitude)
14. 분석 근거와 한계

## 분석 방법과 한계

연결된 기기에서 APK를 추출해 ① 매니페스트(aapt2), ② DEX 클래스·메서드(dexdump, 30,737 클래스), ③ Hermes 바이트코드 디컴파일(hermes-dec, 함수 48,863개)과 문자열 테이블, ④ 번들에 포함된 GraphQL 문서 197개를 분석했습니다.

앱을 실행하거나 네트워크 트래픽을 관찰하지 않았으므로, 흐름도와 일부 서버 역할 구분은 이름·구조에 근거한 추론을 포함합니다. 네이티브 Kotlin 본문은 메서드 시그니처만 사용했습니다. API 키·토큰·DSN 등 비밀 값은 문서에 포함하지 않았습니다.
