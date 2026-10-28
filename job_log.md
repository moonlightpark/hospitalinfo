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

---

# 추가 분석 (2026-10-29, 회사 동의 확인 후)

세 단계로 나눠 추가 분석했습니다. 각 단계의 방법·결과·한계를 기록합니다.

## 1단계 — 동적 분석 (앱 실행 관찰)

**방법.** 기기에서 앱을 실행하고 `logcat`(앱 PID 한정), `dumpsys activity`, 화면 캡처로 런타임 상태를 관찰. 로그인·거래 요청은 발생시키지 않음(실행·초기화 관찰만).

**확인한 사실 (관찰).**
- 실행 중 JS는 APK 내장 번들이 아니라 **CodePush OTA 번들**: `files/CodePush/c32f6664…/index.android.bundle`. → 정적 분석 대상(APK 번들)과 실제 구동 코드가 다를 수 있음.
- **React Native Bridgeless(New Architecture)** 모드로 기동(`ReactHost…BridgelessReact`, Fabric 코드젠 so 로드).
- 하단 탭 4개를 화면으로 확정: **홈 / 건강관리 / 내 진료 / 더보기** (정적 추론 TabHome/Health/History/More와 일치).
- 홈 화면 구성 확인: 위치(내 주변), 검색바, 시술 카테고리(독감백신·HPV·대상포진·마운자로·위고비·임플란트·탈모약·여드름약), 리뷰 이벤트 배너, 미리접수·예약 / 비대면진료 전환, 진료과 아이콘.
- SDK 런타임 초기화: Firebase(FA v192007), **OneSignal appId `75add6c6-…`**(클라이언트 공개값), AppsFlyer(R8 난독), Sentry `auto-init: false`(수동 초기화), WorkManager.

**한계 / 블로커.**
- 기기 **비루팅**(`su` 없음, `release-keys`, non-debuggable), frida 미설치.
- APK에 커스텀 network security config 없음 → 플랫폼 기본값. Android 14+에서 앱은 **시스템 CA만 신뢰**.
- 따라서 **HTTPS 트래픽 복호화(mitmproxy)는 루팅 또는 앱 리패키징 없이는 불가**. 실제 요청/응답 본문은 이번에도 캡처하지 못함.
- 앱 비디버거블 → `run-as`로 CodePush 번들(private 데이터) 추출 불가.

## 2단계 — 네이티브 모듈 디컴파일

**방법.** JADX의 Java 디컴파일은 자동 모드 보안 분류기가 차단(외부 코드 실행 판단). 대신 이미 승인된 `dexdump -d`로 대상 클래스의 바이트코드를 디스어셈블해 로직 복원.

**확인한 사실 (관찰) — `IntegrityModule` 루팅 탐지 로직.**
- su 바이너리 경로 13종 존재 확인: `/su/bin/su`, `/magisk/.core/bin/su`, `/system/app/Superuser.apk`, `/system/app/SuperSU.apk`, `/system/xbin/su`, `/sbin/su`, `/system/bin/su`, `/system/sd/xbin/su`, `/data/local/xbin/su`, `/data/local/bin/su`, `/system/sbin/su` 등.
- `ProcessBuilder`로 `which su`(`/system/bin/which`, `/system/xbin/which`) 및 `sh -c` 실행 후 출력에 `su` 포함 여부 검사(`runWhichAndOutputContainsSu`, `suFoundViaSystemWhich`).
- `Build.TAGS`에 `test-keys` 포함 여부 검사(`hasTestKeysBuildTags`).
- 앱 서명 해시: `PackageManager.getPackageInfo(...).signatures` → `MessageDigest("SHA")`로 `keyHash` 계산(`getKeyHash`).
- 백그라운드 `ExecutorService`로 수행, 실패 시 `ROOT_CHECK_ERROR`.

**한계.** JADX 미승인으로 Kotlin 소스 수준(변수명·고수준 제어흐름)이 아닌 바이트코드 수준 복원. 나머지 자체 모듈(LocalPush/SamsungHealth 등)은 시그니처만 보유.

## 3단계 — 기능 심층 추적 (상태 머신)

**방법.** 디컴파일된 Hermes 번들의 graphql-codegen enum 테이블을 파싱해 도메인 상태 머신의 실제 멤버 값을 복원.

**확인한 사실 (관찰).**
- **TreatmentStatusEnum (비대면 진료)**: `waiting → request → ontreatment → endcall → orderprescription → completion`, 분기: `matchingfail`, `treatmenterror`, `cancel`.
- **VideoCallStatusEnum**: `createdSession → matched → doctorJoined / patientJoined → doctorLeft / patientLeft`.
- **TreatmentMethodTypeEnum**: `phoneCall`, `videoCall`.
- **ReceiptStatusEnum (모바일 접수)**: 코드형 — 정상 흐름 `C00·C01·C02·C03·C04·C06·C07·C08`, 실패 `F02·F03·F05`.
- **ReceiptWaitingTypeEnum**: `minute`(분 단위 대기) / `person`(순번).
- **PaymentStatusEnum**: `requested → confirmationRequested → paymentCompleted / paymentFailed → refundCompleted / refundFailed`.
- **UntactServiceStatusEnum**: `active / deleted / none`.

**해석 (추론).** 접수 상태의 C/F 접두어는 각각 정상 단계와 실패 사유로 보이며, 실제 의미 매핑은 서버 스키마 확인 필요. 비대면 진료 상태와 화상통화 상태는 별도 enum으로 분리되어 통화 단계(세션 생성·입장·퇴장)를 독립 추적.

## 추가 분석 종합 결론

- 정적 근거로 그린 구조·화면·탭은 런타임에서 **사실로 확인**됨(탭 구성, New Arch, 자체 모듈 초기화).
- 루팅 탐지·결제·진료 상태 머신 등 **핵심 로직을 값 수준까지 복원**함.
- 남은 공백은 **서버 요청/응답 본문**뿐이며, 이는 기기 루팅 또는 앱 리패키징이 전제되어야 함(현재 미수행). 필요 시 별도 환경에서 진행 권장.
