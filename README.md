# hospitalinfo

굿닥(Goodoc) Android 앱(`com.ksncho.hospitalinfo` v50.14.29423)을 정적 분석해 재구성한 소프트웨어 설계도입니다.

- 설계도: [docs/gooddoc-android-blueprint.html](docs/gooddoc-android-blueprint.html) (브라우저로 열면 다이어그램이 렌더링됩니다)

## 담긴 내용

1. 앱 개요와 식별 정보
2. 시스템 구성도 (굿닥 서버와 외부 서비스)
3. 앱 레이어 아키텍처 (React Native + Hermes)
4. 기능 도메인별 화면 구성 (화면 168개)
5. 서버 API (GraphQL 연산, WebSocket, REST 경로)
6. 핵심 흐름 (모바일 접수, 비대면 진료부터 처방까지, 앱 시작)
7. 자체 네이티브 모듈
8. 권한과 보안
9. 외부 SDK
10. 분석 근거와 한계

## 분석 방법과 한계

연결된 기기에서 APK를 추출해 매니페스트, DEX 클래스·메서드 목록, Hermes 바이트코드 문자열 테이블을 분석했습니다. 앱을 실행하거나 네트워크 트래픽을 관찰하지 않았으므로 흐름도와 서버별 역할은 이름에 근거한 추론을 포함합니다. API 키 등 비밀 값은 문서에 포함하지 않았습니다.
