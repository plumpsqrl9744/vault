# ERobot 시스템 아키텍처

> 확정 구조 (다이어그램 기준, 2026-07-03). 기존 [[ERUT]]의 "앱→ESP32→UART→Teensy" 구상은 이 구조로 **대체됨** — ESP32 브릿지 대신 ITX PC + RPi5 계층 구조.

## 전체 구성도

**IP65 방수 케이싱 내부 (로봇 본체)**

```mermaid
flowchart TD
    subgraph CASE["IP65 방수 케이싱"]
        AOS["AOS 데이터 수집 보드<br/>(MC·PA)"]
        ITX["ITX_PC_보드<br/>(AOS 라이브러리 인터페이스<br/>및 데이터 송수신)"]
        RPI["RPi5 · Ubuntu 22.04 · ROS2<br/>(AI 트래킹 및 모터 제어 명령)"]
        TNSY["Teensy 4.1 · micro-ROS<br/>(구동체 실시간 제어)"]
        AOS <--> ITX
        ITX <--> RPI
        RPI <--> TNSY
    end
    subgraph NET["네트워크 구성"]
        GW["Gateway 라우터"] --> POE["PoE"] --> AP["AP"]
    end
    ITX <-. WIFI .-> AP
    AP <-. WIFI .-> APP["ERobot 컨트롤러 앱"]
    AP <-. WIFI .-> MON["모니터링 Windows 프로그램<br/>(노트북)"]
```

**Teensy 4.1 하위 구동계**

```mermaid
flowchart TD
    T["Teensy 4.1 · micro-ROS"]
    T --> TC1["트랜시버(C)"] --> P1["물공급 펌프(L)"]
    T --> TL["트랜시버(L)"] --> HL["홀더 Index 모터 L"]
    T --> C620L["C620"] --> ML["M3508 P19 바퀴 모터(L)"]
    T --> C620R["C620"] --> MR["M3508 P19 바퀴 모터(R)"]
    T --> TR["트랜시버(R)"] --> HR["홀더 Index 모터 R"]
    T --> TC2["트랜시버(C)"] --> P2["물공급 펌프(R)"]
    T --> TC3["트랜시버(C)"] --> HV["홀더 상하 모터"]
```

## 계층별 역할

| 계층 | 담당 | 성격 |
| --- | --- | --- |
| ERobot 컨트롤러 앱 | 사용자 조작 (RN + TypeScript) | UI/명령 발신 |
| 모니터링 Windows 프로그램 | 노트북 관제 | 모니터링 |
| ITX_PC_보드 ([[하드웨어 스펙 시트#3 GIGAIPC iTXL-6412A 산업용 마더보드 — MEHLJAT-SI|iTXL-6412A]]) | AOS 라이브러리 인터페이스·데이터 송수신, Wi-Fi로 외부와 통신하는 관문 | x86 Windows PC |
| RPi5 ([[하드웨어 스펙 시트#2 Raspberry Pi 5 SBC 후보 — 공식 Product Brief|스펙]]) | **Ubuntu 22.04 + ROS2** — AI 트래킹, 모터 제어 명령 생성 | 리눅스 SBC (판단 담당) |
| Teensy 4.1 ([[하드웨어 스펙 시트#1 Teensy 41 제어 MCU — Joy-IT 데이터시트|스펙]]) | **micro-ROS** — 구동체 실시간 제어 | 실시간 MCU (반사신경 담당) |
| AOS 데이터 수집 보드 | MC/PA 초음파 데이터 수집 | 센서 프론트엔드 |

## 구간별 연결 방식 (학습 포인트)

* **① 앱/노트북 ↔ AP ↔ ITX PC : Wi-Fi**
	* Gateway 라우터 → PoE → AP로 현장 무선망 구성. PoE = 랜선 하나로 전원+데이터를 AP에 공급 (AP 위치 자유).
	* 앱은 이 Wi-Fi망을 통해 ITX PC와 통신 (TCP Socket — [[ERUT]] 프로토콜 설계 그대로 유효).
* **② ITX PC ↔ RPi5 : 유선 (방식 미확정 [UNCLEAR])**
	* 케이싱 내부 보드 간 연결. 후보: 이더넷 직결(ROS2 DDS가 IP 기반이라 자연스러움) 또는 USB.
	* iTXL-6412A가 GbE 포트 2개라 하나를 RPi5 직결에 쓰는 구성이 유력.
* **③ RPi5 ↔ Teensy 4.1 : micro-ROS (시리얼 계열)**
	* **micro-ROS 구조**: RPi5에서 `micro-ROS Agent` 실행 ↔ Teensy에서 `micro-ROS Client`(펌웨어) 실행.
	* 전송(transport)은 통상 **USB/UART 시리얼** — Teensy가 ROS2 노드처럼 보이게 되어, RPi5의 ROS2 토픽(`cmd_vel` 등)을 Teensy가 직접 구독.
	* 기존 구상(ESP32가 Wi-Fi→UART 중계)이 필요 없어진 이유: RPi5가 그 자리를 대체하고 ROS2 생태계까지 얹음.
* **④ Teensy ↔ 바퀴 모터 : CAN 버스 (C620 ESC)**
	* **C620** = DJI RoboMaster **M3508 P19** 기어드 모터 전용 ESC(변속기), **CAN 통신으로 제어** (전류 제어 명령·엔코더 피드백).
	* Teensy 4.1은 CAN 컨트롤러 내장([[하드웨어 스펙 시트]] 참조) — 외부에 **CAN 트랜시버**(다이어그램의 '트랜시버' 박스)만 붙이면 됨.
* **⑤ Teensy ↔ 펌프/홀더 모터 : 트랜시버 경유**
	* 물공급 펌프 L/R, 홀더 Index 모터 L/R, 홀더 상하 모터 — 각각 트랜시버(C/L/R) 경유 제어. 구체 프로토콜 미확정 [UNCLEAR] (CAN 공유 버스 또는 개별 드라이버).

## micro-ROS Arduino 구현 참고 (Teensy 펌웨어)

> 출처: [micro-ROS/micro_ros_arduino](https://github.com/micro-ROS/micro_ros_arduino) README (2026-07-03 확인)

* **Teensy 4.1 공식 지원** — Supported 상태 (Teensy 4.0은 Not tested). Teensyduino 1.58.x 기반, 라이브러리 최소 v1.8.5.
* **설치**: 릴리스 페이지에서 ZIP 다운로드 → Arduino IDE `Sketch → Include Library → Add .ZIP Library...`
* **Teensyduino 패치 필수** — `platform.txt`를 micro_ros_arduino의 patched 버전으로 교체해야 빌드됨:
	```bash
	export TEENSYDUINO_VERSION=[버전]
	export ARDUINO_PATH=[경로]
	cd $ARDUINO_PATH/hardware/avr/$TEENSYDUINO_VERSION/
	curl https://raw.githubusercontent.com/micro-ROS/micro_ros_arduino/main/extras/patching_boards/platform_teensy.txt > platform.txt
	```
* **Transport**: README 기준 "Only USB serial transports are provided" — 우리 설계(RPi5↔Teensy USB 시리얼)와 정확히 일치. Wi-Fi transport는 일부 보드(ESP32 계열)용이라 Teensy에선 고려 대상 아님.
* **Agent 실행 (RPi5 쪽)** — Docker 한 줄:
	```bash
	docker run -it --rm -v /dev:/dev --privileged --net=host \
	  microros/micro-ros-agent:humble serial --dev [보드 포트] -v6
	```
	* ROS2 버전 태그(humble/kilted 등)를 RPi5의 ROS2 배포판과 맞출 것. Ubuntu 22.04면 **humble**이 정합.
* **주의**: precompiled 라이브러리는 **정적 메모리 풀이 사전 구성**돼 있음 — 노드/토픽/서비스 수가 기본 한도를 넘으면 colcon.meta 조정 후 재빌드 필요. 예제는 repo `examples/` 폴더 참조.
* **펌웨어 주입(업로드) 방식** — Teensy를 개발 PC에 직결하지 않고 **RPi5 경유**로 배포:
	* 개발 PC → (이더넷) → **RPi5에 SSH 접속** → 펌웨어 파일 전송 → RPi5 ↔ Teensy **USB 연결**로 주입. (전송→전송→주입의 릴레이 방식)
	* Why: 로봇이 IP65 케이싱에 밀폐돼 있어 Teensy USB 포트에 물리 접근이 어려움 — 케이싱 안 RPi5·Teensy가 이미 USB로 물려 있으니 그 선을 업로드 경로로 재사용.
	* RPi5 쪽에서 Teensy CLI 업로더(`teensy_loader_cli` 계열)로 주입하는 형태 — 구체 도구 확정 시 갱신 [UNCLEAR].

## Teensy 펌웨어의 역할 경계 — micro-ROS는 어디까지 해주나

> 핵심: **micro-ROS는 통신(귀)까지만, 전기 신호(손)는 직접 구현.**

```
[RPi5 ROS2 토픽] ──micro-ROS가 배달──> [Teensy 콜백 함수]
                                            │  ← 여기부터 직접 작성
                                            ▼
                              [CAN 프레임 / PWM / GPIO 전기 신호]
```

* **micro-ROS 라이브러리** : ROS2 토픽/서비스를 시리얼로 직렬화해 주고받는 **통신 계층만** 제공. "cmd_vel 도착 → 콜백 호출"까지가 끝. 모터 제어 방법은 **기재돼 있지 않음**.
* 전기 신호를 보내는 방법은 소스 3개에서 나옴:

***① 바퀴 모터 정답지 = DJI C620 사용자 매뉴얼***
* C620의 CAN 프로토콜이 매뉴얼에 표로 정의됨:
	* **제어** : CAN ID `0x200` 프레임 8바이트에 모터 1~4번 전류 명령을 16bit씩 (-16384~16384 = -20A~20A)
	* **피드백** : 각 모터가 `0x201`~`0x204`로 회전자 각도·RPM·실제 전류·온도 송신
* "어떤 바이트를 어떤 ID로 보내면 도는가"는 전부 이 매뉴얼이 정의.

***② Teensy에서 CAN 송신 = FlexCAN_T4 라이브러리***
* Teensy 4.x 내장 CAN 컨트롤러용 사실상 표준 Arduino 라이브러리.
```cpp
#include <FlexCAN_T4.h>
FlexCAN_T4<CAN1, RX_SIZE_256, TX_SIZE_16> can1;

// micro-ROS 콜백: cmd_vel 수신 → CAN 프레임 송신
void cmd_vel_callback(const void *msgin) {
  int16_t current = velocity_to_current(msg->linear.x);  // 직접 작성하는 변환
  CAN_message_t frame;
  frame.id = 0x200;                    // ← C620 매뉴얼이 정의한 ID
  frame.buf[0] = current >> 8;         // 모터1 전류 상위 바이트
  frame.buf[1] = current & 0xFF;       // 하위 바이트
  can1.write(frame);                   // ← 전기 신호는 여기서 출발
}
```
* 실제 전압 레벨(differential)은 **CAN 트랜시버 칩이 하드웨어로** 처리 — 코드는 프레임만 넘기면 됨. 다이어그램의 '트랜시버' 박스가 이 역할.

***③ 펌프·홀더 모터 = 해당 드라이버 데이터시트***
* CAN이 아니면 대개 PWM(`analogWrite`) 또는 GPIO(`digitalWrite`) — 핀·신호 규격은 각 드라이버 데이터시트가 정의. (드라이버 미확정 [UNCLEAR])

***펌웨어 뼈대 한 줄***
`micro-ROS 구독 콜백 → 값 변환(속도→전류) → FlexCAN_T4 송신` + 통신 두절 감지 시 전류 0 (Watchdog).

## 기존 구상에서 달라진 점

| 항목 | 기존 ([[ERUT]]) | 확정 아키텍처 |
| --- | --- | --- |
| Wi-Fi 수신 | ESP32 브릿지 | **ITX PC** (관문) |
| 명령 전달 | 앱→ESP32→UART→Teensy | 앱→ITX PC→**RPi5(ROS2)**→**Teensy(micro-ROS)** |
| 지능 계층 | 없음 (앱이 직접 제어) | **RPi5가 AI 트래킹·모터 명령 생성** |
| 모터 제어 | (미정) | **CAN 버스 + C620 ESC + M3508** |
| 앱 프로토콜 | 자체 바이너리 패킷 | 유효 — 단 수신처가 ITX PC로 변경. ITX PC↔RPi5 구간부터는 ROS2 토픽 |

* 앱 쪽 설계([[ERUT]]의 패킷 규격·지속 연결·Watchdog·개발 도구)는 그대로 유효 — 상대가 ESP32에서 ITX PC로 바뀌었을 뿐.
* Fail-safe(Watchdog)는 여전히 **Teensy 펌웨어 단**에 필수 — 상위 계층(RPi5·ITX PC·Wi-Fi) 어디가 죽어도 구동계는 멈춰야 함.
