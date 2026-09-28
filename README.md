# ROSCamp Robot Automation · 김형준 포트폴리오

ROS 2와 AI를 활용한 지능형 무인 마트 자동화 프로젝트입니다. 결제 이후 생성된 작업을 서버에서 관리하고, 로봇 시스템과 LLM 명령 처리 구조로 연결하는 팀 프로젝트에서 김형준이 담당한 서버·LLM·데이터 계층과 직접 수정한 로봇 제어 코드를 포트폴리오용으로 정리했습니다.

<p>
  <a href="https://github.com/kimhyounjun"><img src="https://img.shields.io/badge/GitHub-kimhyounjun-181717?style=flat-square&logo=github" alt="GitHub" /></a>
  <img src="https://img.shields.io/badge/ROS%202-Jazzy-22314E?style=flat-square&logo=ros" alt="ROS 2" />
  <img src="https://img.shields.io/badge/FastAPI-Backend-009688?style=flat-square&logo=fastapi&logoColor=white" alt="FastAPI" />
  <img src="https://img.shields.io/badge/MySQL-Database-4479A1?style=flat-square&logo=mysql&logoColor=white" alt="MySQL" />
  <img src="https://img.shields.io/badge/OpenAI-LLM-412991?style=flat-square&logo=openai&logoColor=white" alt="OpenAI" />
</p>

## 개인 기여 범위

원본 README의 팀 역할과 Git 이력을 기준으로 서버·FastAPI·MySQL·LLM 영역을 선별했습니다. 또한 Git 이력에서 김형준 이름으로 확인되는 `robot/pinky_1waypoint_567_lcd.py` 수정본을 함께 포함했습니다. 다른 팀원의 로봇팔·비전·자율주행 실험 코드는 제외했습니다.

| 담당 영역 | 구현 내용 |
| --- | --- |
| FastAPI 서버 | 도메인별 API 구조, 헬스 체크, CORS, 정적 웹 연결, 서버 실행 구성 |
| 데이터 계층 | SQLAlchemy 모델·스키마·DB 세션·환경설정 구조 |
| LLM 라우터 | 자연어 로봇 명령을 simple/complex/dangerous로 분류하고 모델과 액션 JSON을 선택 |
| 마트·주문 도메인 | 상품·주문·결제·사용자 모델과 LLM 질의 연동 구조 |
| 로봇 통합 | 서버에서 로봇 동작을 호출할 수 있도록 LLM·ROS 2 연동 도메인 구성 |
| 직접 수정 코드 | Pinky Nav2 웨이포인트 이동, LED 상태 표시, LCD 수령 안내와 복귀 대기 로직 |

## 프로젝트 흐름

```text
사용자 결제
    ↓
FastAPI 서버에서 주문·작업 생성
    ↓
LLM 라우터가 자연어 명령을 안전한 액션 JSON으로 변환
    ↓
ROS 2 로봇 도메인과 연동
    ↓
Pinky 웨이포인트 이동·상태 표시
    ↓
목적지 도착 후 LCD 안내 및 복귀 대기
```

## 파일 구조

```text
roscamp-robot-automation-portfolio/
├── fastapi-server/                         # 메인 FastAPI 서버
│   ├── app/
│   │   ├── api/                            # 도메인별 API 라우터
│   │   ├── core/                           # 설정·로깅
│   │   ├── db/                             # DB 세션·초기화
│   │   ├── models/                         # 사용자·상품·주문·결제 모델
│   │   ├── schemas/                        # API 요청·응답 스키마
│   │   └── services/                       # LLM 서비스·라우팅 로직
│   ├── run.py                              # 서버 실행 진입점
│   └── requirements.txt                    # 서버 의존성
├── llm-router/                              # 독립 실행형 LLM 로봇 명령 라우터
│   ├── main.py                              # 명령 분류·모델 선택·액션 JSON 생성
│   └── run.sh                               # 라우터 실행 스크립트
├── AImarket.py                              # 마트·주문·로봇 연동 애플리케이션
├── AImarketdb.py                            # 마트 DB 연결·조회 모듈
├── robot/
│   └── pinky_1waypoint_567_lcd.py           # 직접 수정한 Pinky 이동·LED·LCD 제어
├── .env.example                             # 환경변수 설정 예시
└── requirements.txt                          # 공통 Python 의존성
```

## 핵심 구현

### 1. 도메인 분리형 FastAPI 서버

`fastapi-server/app/main.py`는 서버의 진입점으로 다음 영역을 분리해 연결합니다.

| 도메인 | 역할 |
| --- | --- |
| Main | 사용자·상품·주문·결제 등 마트 비즈니스 처리 |
| Control | 시스템 상태와 운영 모니터링 |
| LLM | 자연어 명령과 로봇 제어 액션 연결 |
| Robot | ArUco·로봇 동작 요청을 ROS 2 영역과 연결 |

서버는 CORS 설정, 환경별 설정, 구조화 로깅, 정적 웹 화면 제공, 헬스 체크를 포함하도록 구성했습니다.

### 2. 데이터 모델과 환경 설정

- `app/models/`: 사용자·상품·주문·결제 SQLAlchemy 모델
- `app/schemas/`: Pydantic 기반 요청·응답 검증
- `app/db/session.py`: 데이터베이스 엔진과 세션 관리
- `app/core/config.py`: DB URL, OpenAI 키, CORS 등 환경변수 관리
- `app/db/init_db.py`: 초기 데이터베이스 구성

민감 정보는 `.env`로 분리하고 `.env.example`에는 설정 형식만 남겼습니다.

### 3. LLM 기반 로봇 명령 라우터

`llm-router/main.py`는 사용자의 자연어 명령을 바로 로봇에 전달하지 않고, 먼저 의도를 분류한 뒤 구조화된 액션으로 변환합니다.

```text
자연어 명령
    ↓
위험도·복잡도 분류
    ├── simple    → 빠른 모델
    ├── complex   → 고성능 모델
    └── dangerous → 안전 규칙을 적용한 제한 처리
    ↓
ActionJSON(type, params)
```

지원하는 대표 액션은 `move_twist`, `set_led`, `set_emotion`, `noop`이며, 이동 속도·회전 속도·실행 시간의 범위를 검증하도록 설계했습니다.

### 4. Pinky 웨이포인트·상태 표시

`robot/pinky_1waypoint_567_lcd.py`는 김형준 이름으로 커밋된 직접 수정 코드입니다.

- Nav2 `NavigateToPose` 액션 클라이언트로 웨이포인트 순차 이동
- 특정 웨이포인트에서 LED를 빨간색으로 표시하고 대기
- 목적지 도착 시 LCD에 상품 수령 안내 표시
- 수령 대기 시간이 지나면 첫 웨이포인트로 복귀
- 액션 성공·실패 상태와 이동 중 상태를 내부 플래그로 관리

## 기술 스택

| 구분 | 기술 | 사용 목적 |
| --- | --- | --- |
| Robot Platform | ROS 2 Jazzy, Pinky, JetCobot | 로봇 시스템 연동 |
| Backend | Python, FastAPI, Uvicorn | API 서버와 작업 관리 |
| Database | MySQL, SQLAlchemy | 사용자·상품·주문·결제 데이터 |
| Validation | Pydantic | API 요청·응답 검증 |
| AI | OpenAI API | 자연어 명령 해석과 액션 생성 |
| Robot Control | Nav2, ROS 2 Action, ROS 2 Topic/Service | 이동·상태 제어 |
| UI | LCD, LED, Static Web | 사용자 안내와 로봇 상태 표시 |

## 실행 방법

### FastAPI 서버

```bash
cd fastapi-server
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
python run.py
```

### LLM 라우터

```bash
cd llm-router
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
./run.sh
```

### Pinky 제어 코드

`robot/pinky_1waypoint_567_lcd.py`는 ROS 2와 Pinky 전용 패키지가 설치된 로봇 환경에서 실행합니다.

```bash
python3 robot/pinky_1waypoint_567_lcd.py
```

실제 실행 전 ROS 2 환경, `pinky_interfaces`, `pinky_lcd`, Nav2 액션 서버와 LED 서비스를 확인해야 합니다.

## 포트폴리오 확인 포인트

1. `fastapi-server/app/main.py`에서 서버 도메인 분리와 API 진입 구조를 확인할 수 있습니다.
2. `fastapi-server/app/core`, `app/db`, `app/models`, `app/schemas`에서 백엔드 기본 구조를 확인할 수 있습니다.
3. `llm-router/main.py`에서 자연어 명령 분류와 안전한 액션 JSON 변환을 확인할 수 있습니다.
4. `robot/pinky_1waypoint_567_lcd.py`에서 Nav2 이동·LED·LCD 상태 제어를 확인할 수 있습니다.

## 보안 및 범위

- API 키, DB 비밀번호, 로봇 IP와 SSH 비밀번호는 환경변수로 관리합니다.
- 다른 팀원이 담당한 로봇팔·카메라 캘리브레이션·YOLO 신호등·AMCL 실험 코드는 이 저장소에서 제외했습니다.
- 원본 팀 저장소의 전체 구현은 아래 링크에서 확인할 수 있습니다.

## 관련 링크

- 개인 GitHub: [kimhyounjun](https://github.com/kimhyounjun)
- 원본 팀 저장소: [roscamp-repo-1](https://github.com/addinedu-roscamp-9th/roscamp-repo-1)
