# 시골쥐 도시에 오다! (Jump Cheese)
STM32F10x(Cortex‑M3) 기반 2D 러너/점프 회피 게임. SPI TFT LCD에 그래픽을 표시하고, 타이머·외부 인터럽트로 게임 루프와 입력을 처리합니다. ARM 아키텍처 수업의 `GAME_PROJECT` 예제를 레퍼런스 삼아 로직/그래픽/하드웨어 제어를 재구성했습니다.

> **플랫폼**: STM32F10x 시리즈 MICOM, Cortex‑M3  
> **디스플레이**: 320×240 SPI TFT (예: ILI9341 호환)  
> **언어/툴**: C, GCC/ARM/Keil/STM32CubeIDE, ST‑Link

---

## 🎮 게임 소개
- **제목**: 시골쥐 도시에 오다!  
- **목표**: 고양이·올빼미 등 **충돌 요소를 피하고**, 날아오는 **치즈를 먹어 점수**를 올립니다.  
- **컨셉**: 동화 *「시골쥐와 도시쥐」*에서 모티브. 도시를 경험한 시골쥐가 **도시를 탈출해 고향으로 돌아가는 여정**.

### 장르
2D / 런게임 / 어드벤처 / 점프 회피

---

## 🕹 조작
| 동작 | 기본 키 |
|---|---|
| 게임 시작 / 재시작 | **KEY5** |
| 점프 | **KEY0** (JOG Up) |

> 키 매핑은 EXTI 인터럽트 핸들러(예: `stm32f10x_it.c`)에서 조정 가능합니다.

---

## 🧠 게임 상태
게임은 **MENU → PLAYING → GAMEOVER**의 3가지 상태로 구성됩니다.

- **MENU**: 시작 화면 및 BGM(옵션). 키 입력 시 게임 시작.  
- **PLAYING**: 자동 달리기, **KEY0**로 점프. 치즈 먹으면 점수 +10. 충돌 시 게임오버.  
- **GAMEOVER**: 게임오버 화면 및 BGM(옵션) 출력. **KEY5**로 재시작.

---

## 🧀 점수 & 아이템
- **치즈** 1개 먹을 때마다 **+10점**.  
- **치즈 누적**으로 **버스트/무적** 상태 발동(지속시간·배수는 `define.h` 상수로 조정).  
  - 버스트 중에는 플레이어 이펙트와 **이동 속도 배수** 적용.

---

## 🐾 적/충돌 규칙
- **고양이(CAT)**: 화면 우측에서 삼각형 형태로 등장, 좌측으로 이동.  
- **올빼미(OWL)**: 상단에서 **가변 높이**의 직사각형 장벽으로 좌측 이동.  
- **무적(버스트) 상태**에서는 충돌 판정 무시.

---

## 🔧 하드웨어 연결(기본값)
> 실제 핀 매핑은 보드에 따라 다를 수 있습니다. 아래는 기본 드라이버의 매핑 예시입니다.

### LCD(TFT, SPI1)
| 기능 | 핀 |
|---|---|
| LCD_CS | **PA4** |
| LCD_LED(백라이트) | **PA11** |
| LCD_RST | **PB4** |
| LCD_RS(DC) | **PB9** |
| SPI1_SCK | **PA5** *(일반적 기본값)* |
| SPI1_MOSI | **PA7** *(일반적 기본값; MISO 미사용)* |

- 디스플레이 해상도: **320×240**  
- 방향/메모리 액세스 모드는 드라이버 초기화 시 설정됩니다.

### 기타
- **Buzzer**: **TIM3** PWM/주파수 출력 사용.  
- **주기 타이머**: **TIM4**(게임 루프), **TIM2**(BGM/효과음 타이밍) 등.  
- **키 입력**: GPIO 외부 인터럽트(EXTI) 기반. KEY0/KEY5 등 조합.

---

## 🗂 프로젝트 구조
```
├─ main.c               # 게임 루프·상태 전환·스폰/충돌·UI
├─ stm32f10x_it.c       # 인터럽트(ISR): TIM2/TIM4, EXTI, USART1 등
├─ clock.c              # 시스템 클록 초기화
├─ core_cm3.c           # CMSIS 코어 유틸리티
├─ graphics.c           # 글자 출력(Lcd_Printf 등)
├─ lcd.c                # SPI1 기반 TFT LCD 드라이버(320x240)
├─ key.c                # 키 스캔/인터럽트 초기화
├─ led.c                # LED 제어
├─ runtime.c            # 힙(sbrk) 등 런타임 보조
├─ device_driver.h      # 하드웨어 레지스터/매크로
├─ define.h             # 게임 상수(사이즈, 속도, TIMER_PERIOD 등)
└─ ...                  # (필요 시 추가)
```

---

## 🏗 빌드 & 실행
> **Toolchain 예시**: Keil MDK‑ARM, STM32CubeIDE, arm‑none‑eabi‑gcc + Makefile  
> **디버거**: ST‑Link v2

1) **프로젝트 생성/설정**  
- MCU: **STM32F10x (Cortex‑M3)**  
- C 표준 C11 또는 C99 권장, 최적화 `-O0 ~ -O2`  
- CMSIS/표준 주변장치 헤더 포함

2) **소스 추가**  
- 위 목록의 `.c/.h` 파일을 프로젝트에 추가합니다.

3) **링커/부트 설정**  
- 코드에서 **VTOR(벡터 테이블) 오프셋**을 `0x08003000`으로 설정하는 경우가 있습니다.  
  - 부트로더 또는 섹션 배치에 맞춰 **링커 스크립트**의 벡터 테이블 주소를 일치시키세요.  
  - 기본 부트 없이 사용한다면 오프셋을 `0x08000000`대로 조정하거나, 링커 스크립트를 수정하세요.

4) **플래시 & 실행**  
- ST‑Link로 다운로드 후 보드를 리셋해 실행합니다.

---

## 🔩 주요 모듈 개요
- **Game Loop**: `MainLoop()` – 주기 타이머 기반으로 **스폰/이동/충돌/점수/타이머**를 처리.  
- **State Manager**: `ResetGame()`, `Game_Init()`, `Draw_Start_Menu()`, `Draw_GameOver_Screen()`  
- **Player**: `Player_Jump_Start()`, `Player_Update()`, `Player_Draw()`  
- **Enemies**: `Cat_Init/Spawn/Update()`, `Owl_Init/Spawn/Update()`  
- **Item**: `Cheese_Init/Spawn/Update()` – **점수/버스트 트리거**  
- **Audio**: `MUSIC_ON()`, `GAMEOVER_MUSIC_ON()` – **TIM3**로 톤 생성, **TIM2** 타이밍  
- **Display**: `Lcd_Draw_Box/Line/Triangle/Printf()` – 배경/도형/텍스트 렌더링

---

## ⚙️ 주요 파라미터(예)
- **프레임 주기**: `TIMER_PERIOD` (기본 60ms 권장)  
- **점프 초기 속도/중력**: `JUMP_INIT_VY`, `GRAVITY`  
- **치즈/적 스폰 간격**: `CHEESE_SPAWN_INTERVAL`, `CAT_SPAWN_INTERVAL`, `OWL_SPAWN_BASE`  
- **버스트 조건/지속**: `CHEESE_FOR_BURST`, `BURST_DURATION_MS`  
> 상수는 `define.h`에서 조정하세요.

---

## 📸 스크린샷/데모
아래 자리에 사진·GIF를 추가하세요.
```
docs/
 ├─ screenshot-menu.png
 ├─ screenshot-playing.png
 └─ demo.gif
```

---

## 🗺 향후 개선 아이디어
- 배경/오브젝트 스프라이트 추가, 난이도 커브, 점수 콤보  
- EEPROM/Flash에 **하이스코어 저장**  
- 적 패턴 다양화, 파워업 아이템 추가  
- 사운드 믹싱(효과음/BGM 채널 관리)

---

## 📜 라이선스
MIT 또는 프로젝트에 맞는 라이선스를 `LICENSE` 파일로 추가하세요.

## 🙏 Acknowledgements
- ARM Cortex‑M3 / STM32F10x 학습 자료  
- ILI9341 호환 TFT LCD 드라이버 레퍼런스
