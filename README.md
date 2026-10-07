# 🐍 포즈로 조종하는 뱀게임

[Teachable Machine](https://teachablemachine.withgoogle.com/)으로 학습한 **포즈 모델**을 이용해, 웹캠 앞에서 몸동작으로 뱀을 조종하는 웹 게임입니다. 설치 없이 `index.html` 파일 하나로 동작합니다.

## 실행 방법

웹캠은 보안상 `https` 또는 `localhost`에서만 안정적으로 켜지므로 로컬 서버로 실행합니다.

```bash
python -m http.server 8000
```

1. 브라우저에서 `http://localhost:8000` 접속
2. **웹캠 시작** 클릭 → 카메라 권한 허용 (모델 로딩에 몇 초 걸림)
3. **게임 시작** 클릭 후 포즈로 조종

키보드 방향키로도 조종할 수 있고, 스페이스바로 일시정지/재개합니다.

## 동작 원리

```
웹캠 영상 ──▶ PoseNet (관절 17개 위치 추출) ──▶ TM 분류 모델 ──▶ 클래스별 확률
                                                                    │
            뱀 이동 (일정 간격 tick) ◀── 방향 결정 (신뢰도 기준 + 역방향 차단) ◀┘
```

### 1. 포즈 인식
- `@teachablemachine/pose` 라이브러리가 매 프레임(`requestAnimationFrame`) 웹캠 이미지를 받습니다.
- 먼저 **PoseNet**이 코, 어깨, 팔꿈치, 손목 등 관절 위치를 추출하고(`model.estimatePose`),
- 그 관절 정보를 Teachable Machine에서 학습한 분류 모델에 넣어 5개 클래스의 확률을 계산합니다(`model.predict`).
- 사용 모델: `https://teachablemachine.withgoogle.com/models/Ea-ai6NIQ/`

| 클래스 | 동작 |
|---|---|
| `up` | 뱀이 위로 이동 |
| `down` | 아래로 이동 |
| `left` | 왼쪽으로 이동 |
| `right` | 오른쪽으로 이동 |
| `neutral` | 방향 유지 (계속 직진) |

### 2. 방향 결정
- 확률이 가장 높은 클래스가 **신뢰도 기준**(기본 0.80) 이상일 때만 방향 명령으로 인정합니다. 애매한 자세에서 방향이 흔들리는 것을 막기 위함입니다.
- 현재 진행 방향의 **정반대로는 꺾을 수 없습니다** (예: 오른쪽으로 가는 중 `left` 무시). 자기 몸에 바로 부딪혀 죽는 것을 방지합니다.
- 인식된 방향은 바로 적용되지 않고 `nextDir`에 저장되었다가 다음 tick에 반영됩니다. 그래서 인식이 초당 수십 번 일어나도 게임 진행은 안정적입니다.

### 3. 게임 로직
- 20×20 격자, 설정한 속도(기본 220ms)마다 한 칸씩 이동합니다.
- 먹이(빨간 원)를 먹으면 몸이 한 칸 길어지고 점수 +1.
- 자기 몸에 부딪히면 게임 오버. **벽 통과**를 끄면 벽에 부딪혀도 게임 오버입니다.
- 최고 점수는 브라우저 `localStorage`에 저장됩니다.

## 화면 구성

- **왼쪽**: 게임 화면, 현재 점수 / 최고 점수
- **오른쪽 위**: 웹캠 영상 + 인식된 관절·골격, 현재 인식된 방향, 클래스별 확률 막대
- **오른쪽 아래**: 설정
  - **신뢰도 기준**: 낮추면 반응이 빨라지지만 오인식이 늘고, 높이면 정확하지만 둔해집니다.
  - **속도**: tick 간격(ms). 포즈 전환에 시간이 걸리므로 처음엔 느리게 시작하는 것을 추천합니다.
  - **벽 통과**: 켜면 화면 반대편으로 나옵니다.

## 사용 기술

- HTML / CSS / JavaScript (Canvas) — 빌드 과정 없음
- [TensorFlow.js](https://www.tensorflow.org/js) 1.3.1
- [@teachablemachine/pose](https://github.com/googlecreativelab/teachablemachine-community) 0.8

## 다른 모델로 바꾸기

Teachable Machine에서 **포즈 프로젝트**로 `up`, `down`, `left`, `right`, `neutral` 클래스를 학습시켜 업로드한 뒤, `index.html`의 `MODEL_URL`을 새 링크로 바꾸면 됩니다. 클래스 이름은 대소문자를 구분하지 않습니다.
