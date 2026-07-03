ERobot 제어, 하드웨어 제어 앱 개발 스택 선정

***신호 흐름(아키텍처)***
* ⚠️ **구조 변경됨 (2026-07-03)** — 아래 ESP32 구상은 [[ERobot 시스템 아키텍처]]로 대체. 확정: **[앱 → Wi-Fi → ITX PC → RPi5(ROS2) → Teensy 4.1(micro-ROS)]**. 앱 쪽 설계(패킷·지속연결·Watchdog)는 그대로 유효, 수신처만 ESP32→ITX PC로 변경.
* ~~**[앱 → Wi-Fi → ESP32(브릿지) → UART → Teensy 4.1]**~~ (구 구상)
	* 앱에서 보낸 제어 신호가 이 순서로 전달됩니다. ESP32가 Wi-Fi로 받고, UART(시리얼 선)로 Teensy에 넘겨주는 구조입니다.

***통신방법***
* **ESP32:** [하드웨어 부품]
	* **Wi-Fi 통신 기능을 가진 아주 작은 컴퓨터**입니다. Teensy 4.1은 Wi-Fi가 없으니, ESP32를 옆에 붙여서 "Teensy 대신 Wi-Fi 신호를 받아주는 다리" 역할을 시키는 것입니다.
* **TCP/IP (Socket):** **[통신 규약(개념)]**
	* 라이브러리가 아니라 통신을 하는 '방식'입니다. TCP/IP는 데이터를 쪼개서 순서대로 확실하게 전달하는 '규약'이고, Socket은 그 규약을 앱에서 실제로 다루기 위한 '입구'입니다.
	* 선정 이유: 데이터 유실 방지 + 실시간 제어 신호 전송에 적합.

***Framework***
- React Native (TypeScript)
	- iOS/Android 동시 개발 및 유지보수에 유리.

***Library***
* react-native-tcp-socket (TCP/IP 라이브러리)
	* 목적 : Wi-Fi 신호 수신 및 Teensy 4.1 데이터 전달
	* Wi-Fi 소켓 통신의 가장 표준적이고 강력한 라이브러리 (1순위)
* Zustand (상태 관리)
	* 목적 : 로봇의 실시간 상태(모터 속도, 수위 등) 관리
	* Redux보다 훨씬 가볍고 배우기 쉬움 → 로봇 상태 업데이트에 최적 (1순위)
* NativeWind (Tailwind CSS for RN)
	* 목적 : UI/UX 디자인
	* 웹 개발 경험이 있으면 가장 빠르게 깔끔한 'SaaS 스타일' UI 제작 가능
* react-native-joystick
	* 목적 : 상하좌우 움직임 제어용 조이스틱 UI (로봇 필수)

***처음 개발 시 주의사항 (실무 체크리스트)***
* **① 통신 지연(Latency) 관리**
	* 문제 : 버튼 누를 때마다 매번 연결(Connect)을 시도하면 제어가 끊김.
	* 해결 : 앱 실행 시 **한 번 연결하고 유지(Persistent Connection)**. 명령은 큐(Queue)에 담아 **50ms~100ms 간격**으로 전송하는 방식이 가장 안정적.
* **② 데이터 패킷 규격화 (Protocol)**
	* 문제 : 그냥 텍스트("go")를 보내면 통신 에러 시 기계가 오작동.
	* 해결 : `[Start Byte(1) + 명령 ID(1) + 값(2) + CheckSum(1)]` 처럼 **고정 크기 바이트 배열**로 전송해야 기계가 명령을 명확히 해석.
* **③ iOS/Android 권한 처리 (온보딩)**
	* 문제 : 권한을 묻지 않고 통신을 시도하면 앱 강제 종료 또는 연결 실패 (특히 iOS 로컬 네트워크 권한).
	* 해결 : 첫 화면(온보딩)에서 "이 앱은 로컬 네트워크 권한이 필요합니다" 안내 → 사용자 허용 후 연결 프로세스 시작.
* **④ 예외 상황 (Fail-safe)**
	* 문제 : 제어 중 Wi-Fi가 끊기면 로봇이 마지막 명령을 계속 수행 (직진 폭주, 물 쏟음 등).
	* 해결 : 통신 두절 시 Teensy 4.1이 **자동으로 모든 출력을 0(Stop)** 으로 만드는 **Watchdog Timer** 로직을 펌웨어 단에 반드시 심을 것. 앱이 아니라 **펌웨어**에 있어야 하는 이유 = 앱/네트워크가 죽어도 동작해야 하므로.

***통신 프로토콜 설계 (패킷 포맷)***
* 하나의 통신 채널에 모든 명령을 **Command ID**로 분류하여 전송.
* **Packet Structure:** `[StartByte(1)] + [CmdID(1)] + [Value(2)] + [CheckSum(1)]`
* **Command ID 예시:**
	* `0x01` : 바퀴 속도 제어
	* `0x02` : 관찰 렌즈(Probe) 제어
	* `0x03` : 물 공급(펌프) 토글
* 명령 전송 시 **Throttle(50ms)** 을 걸어 패킷 충돌 방지.

***iOS 권한 (구체 구현)***
* `Info.plist`에 **`NSLocalNetworkUsageDescription`** 키 추가 필수 — 이게 없으면 로컬 네트워크 접근 시 그냥 실패함.
* 첫 화면 온보딩에서 권한 승인 받은 뒤 연결 프로세스 시작 (위 주의사항 ③과 동일 맥락).

***개발 로드맵 (Action Plan)***
1. **통신로 확보** — 앱에서 보낸 데이터가 ESP32를 거쳐 Teensy 시리얼 모니터에 찍히는지 확인 (Hello World 통신).
2. **명령어 파싱** — Teensy 펌웨어에서 `switch-case`로 패킷 CmdID별 동작 구현.
3. **UI 연동** — `useTcpSocket` 커스텀 훅을 만들어 각 버튼(조이스틱, 펌프 토글)에서 함수 호출.
4. **상태 모니터링** — 로봇이 보내는 응답값을 읽어 Zustand에 저장하고 UI에 실시간 반영.

***pnpm 세팅 (Expo 프로젝트)***
* **프로젝트 생성**
	```bash
	pnpm create expo-app my-robot-app
	cd my-robot-app
	```
* **필수 설정: `.npmrc`** (프로젝트 루트에 생성)
	```text
	node-linker=hoisted
	```
	* 이유 : pnpm은 기본적으로 의존성을 심볼릭 링크로 격리 관리하는데, **React Native의 네이티브 빌드 도구(Metro Bundler 등)는 전통적인 평탄한 node_modules 구조를 기대**함. 이 설정으로 강제로 맞춰주는 것.
	* 트레이드오프 : hoisted로 하면 pnpm의 strict(유령 의존성 차단) 이점은 일부 반납 — RN에서는 어쩔 수 없는 호환 비용.
* **주요 명령어** : `pnpm add [패키지]` / `pnpm add -D` / `pnpm remove` / `pnpm start` (npm과 거의 동일)
* **빌드 오류 트러블슈팅** (react-native-tcp-socket 등 네이티브 라이브러리 추가 후)
	1. `.npmrc`에 `node-linker=hoisted` 있는지 확인
	2. `rm -rf node_modules && pnpm install` (의존성 재설치)
	3. `npx pod-install` (iOS 의존성 동기화)
* **핵심** : 프로젝트 **시작 시점**에 `.npmrc` 설정을 잊지 않는 것. 나중에 바꾸면 재설치 필요.

***개발 도구 (실시간 확인·디버깅)***
* **Expo Go** : 폰에 설치 후 QR 찍으면 코드 저장 즉시 반영.
	* 단, **네이티브 모듈(react-native-tcp-socket) 못 씀** → 이 프로젝트에서는 UI 스케치 용도까지만.
* **Development Build** (`npx expo run:android/ios`) : 네이티브 모듈 포함 커스텀 개발 앱.
	* TCP 소켓 쓰는 순간 필수 — **처음부터 이걸로 시작**해야 Expo Go→전환 삽질이 없음.
* **Fast Refresh** : 저장 시 상태 유지한 채 화면만 갱신 (RN 기본 내장).
* **React Native DevTools** (터미널에서 `j`) : 콘솔·네트워크·컴포넌트 트리·프로파일링. 구 Flipper 대체(Flipper는 deprecated).
* **Reactotron** : 상태 변화·로그 타임라인 관찰. **Zustand 상태(모터 속도, 수위) 실시간 관찰**에 특히 유용.
* **통신 디버깅 방법**
	* TCP 패킷은 위 툴들로 안 보임 → **Teensy 시리얼 모니터 + 앱 로그를 양쪽에 띄워 대조**가 실전 방법.
	* 하드웨어 없이 패킷 검증만 먼저 하려면 `nc -l [포트]`(netcat)로 **가짜 ESP32 서버**를 만들어 테스트.
* **주의** : 에뮬레이터/시뮬레이터는 로컬 네트워크 환경이 실기기와 달라서, ESP32 통신 테스트는 **같은 Wi-Fi에 물린 실기기**로 할 것.

***개발 순서 (팁)***
* 로봇 제어 앱은 **'하드웨어와 앱의 신뢰성'이 90%**.
* 복잡한 기능부터 만들지 말고, **[버튼 하나 → Teensy에 연결된 LED 켜기]** 라는 가장 단순한 통신 성공을 먼저 확보할 것. 그다음이 바퀴와 관찰 렌즈.
* 성공 기준은 화려한 기능이 아니라 **'안정적인 1바이트 전송'** — 이 구조면 백엔드 없이 P2P로 확장 가능.
