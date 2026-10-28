# 작업 로그 — 굿닥 Android 앱 설계도

## 대상

| 항목 | 값 |
|---|---|
| 앱 | 굿닥 (Goodoc) |
| 패키지 | `com.ksncho.hospitalinfo` |
| 번들 ID | `asia.fast-track.goodoc` |
| 버전 | 50.14.29423 (code 29423) |
| 빌드 환경 | ENV = production |
| SDK | target 36 / compile 36, ABI arm64-v8a |
| APK SHA-256 | `8742c34dd993a8f2f87c84020ee123f36caca3c542921d6a958c42c4de559aff` |
| 추출 기기 | Samsung Galaxy S20+ (SM-G986N), 시리얼 R3CN302BAYV |

## 결과물

- 설계도(HTML): `docs/gooddoc-android-blueprint.html`
- GitHub Pages: https://moonlightpark.github.io/hospitalinfo/docs/gooddoc-android-blueprint.html
- claude.ai 아티팩트: https://claude.ai/artifact/1LSH7VvpcrCf1avhkteZC5 (버전 2)

## 진행 경과

### 1단계 — 기기 연결
- adb로 기기 인식 시도. USB 디버깅 활성화와 재연결 과정을 거쳐 연결 성공.
- 패키지명이 `gooddoc`이 아니라 `com.ksncho.hospitalinfo`임을 aapt2 라벨(`굿닥`)로 확인.

### 2단계 — 1차 분석 (정적, 표층)
- `aapt2 dump xmltree`로 매니페스트(컴포넌트·권한·인텐트·메타데이터) 분석.
- `dexdump`로 DEX 4개의 클래스 30,737개와 자체 네이티브 모듈 메서드 목록 확보.
- Hermes 바이트코드(HBC v96) 헤더를 파싱해 문자열 테이블 53,783개 추출 → 화면 168개, 라이브러리, 도메인, URL 식별.
- 설계도 v1 (10개 장) 작성·배포.

### 3단계 — 2차 분석 (디컴파일, 심층)
- `hermes-dec`로 Hermes 번들을 의사 JS로 디컴파일(함수 48,863개).
- 번들에 포함된 GraphQL 문서 전문 197개를 자체 파서로 구조화 → 데이터 모델·연산 시그니처·fragment·enum 복원.
- 통신/인증/설정 레이어를 디컴파일 코드에서 교차 확인(엔드포인트 split link, 토큰 3종, 접수 WebSocket 헤더, 번들 설정 키 약 40개).
- 설계도 v2 (14개 장) 작성·배포.

### 커밋 이력
- `f6c01be` 설계도 v1 추가
- `2bda025` README에 GitHub Pages 링크 추가
- `448783c` 디컴파일 기반 상세판 v2로 확장

## 사용 도구

| 도구 | 용도 |
|---|---|
| REA (adb, pull_adb_package) | 기기 연결, APK 추출 |
| aapt2 (build-tools 36) | 매니페스트·라벨 |
| dexdump | DEX 클래스·메서드 |
| 자체 Python (HBC 파서) | Hermes 문자열 테이블 |
| hermes-dec | Hermes 바이트코드 디컴파일 |
| 자체 Python (GraphQL 파서) | 번들 내 GraphQL 문서 구조화 |

## 복원한 내용 요약

- 아키텍처: React Native New Architecture + Hermes, 패키지 3개(host/untact/engage)
- 화면 168개, 내비게이션 스택 8개, 딥링크 라우트 약 50개
- GraphQL 연산 197개(query 120 · mutation 74 · subscription 3), fragment 17개, 핵심 엔티티 8종
- 통신: Apollo split link(mobile/clinic), graphql-ws 구독, 접수 WebSocket, untact-ai REST
- 인증: access/refresh/device 토큰, raw fetch 토큰 회전
- 결제: PortOne v2 + 토스페이먼츠, OTA: CodePush
- 자체 네이티브 모듈 4종(GpsUtils, Integrity, LocalPush, SamsungHealth)
- Amplitude 이벤트 185개

## 보안 메모

- 번들 설정 객체에 `SENTRY_AUTH_TOKEN` 키 존재 → 이름상 서버/CI용 토큰으로, 클라이언트 번들 노출은 재검토 필요(값은 문서에 미기재).
- 함께 포함: Amplitude/Kakao/PortOne/Toss 키, Sentry DSN, `CRYPTO_MEMORY_KEY`.

## 한계 / 미확인

- 정적 분석만 수행. 앱 실행·네트워크 캡처 없음 → 실제 호출 순서, 요청·응답 스키마, 서버 내부 구조는 미확인(흐름도는 추론).
- 네이티브 Kotlin 본문 디컴파일(JADX)은 환경 보안 정책으로 미실행 → 메서드 시그니처만 사용.
- Hermes 디컴파일은 레지스터 기반 의사 코드 → 세밀한 분기 조건은 선별 확인.

## 추가 분석 후보 (미실행)

1. 동적 분석: 실행 중 트래픽 캡처(mitmproxy)로 실제 요청/응답·헤더·인증 흐름 검증.
2. GraphQL 스키마 introspection 또는 응답 샘플로 서버 타입 전체 확정.
3. 네이티브 모듈 본문(JADX/Frida)으로 루팅 탐지·암호화 로직 상세화.
4. 특정 기능(예: 비대면 진료 상태 머신, 결제)의 함수 단위 로직 추적.
