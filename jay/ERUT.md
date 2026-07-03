ERobot 제어, 하드웨어 제어 앱 개발 스택 선정

***통신방법***
* **ESP32:** [하드웨어 부품]
	* **Wi-Fi 통신 기능을 가진 아주 작은 컴퓨터**입니다. Teensy 4.1은 Wi-Fi가 없으니, ESP32를 옆에 붙여서 "Teensy 대신 Wi-Fi 신호를 받아주는 다리" 역할을 시키는 것입니다.
* **TCP/IP (Socket):** **[통신 규약(개념)]**
	* 라이브러리가 아니라 통신을 하는 '방식'입니다. TCP/IP는 데이터를 쪼개서 순서대로 확실하게 전달하는 '규약'이고, Socket은 그 규약을 앱에서 실제로 다루기 위한 '입구'입니다.

***Framework***
- React Native (TypeScript)

***Library***
* react-native-tcp-socket (TCP/IP 라이브러리)
	* 목적 : Wi-Fi 신호 수신 및 Teensy 4.1 데이터 전달
* 