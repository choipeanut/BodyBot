# MotionMirror — 실시간 모션 캡처 → 3D 캐릭터 미러링

웹캠으로 신체 움직임을 실시간 추적하여 3D 캐릭터가 동일한 포즈를 따라하는 웹 앱.

## 실행 방법

```bash
# 방법 1: npx serve (권장)
npx serve .
# → http://localhost:3000

# 방법 2: Python
python -m http.server 8080
# → http://localhost:8080

# 방법 3: VS Code Live Server 확장 사용
```

> ⚠️ 웹캠은 localhost 또는 HTTPS 환경에서만 동작합니다.

## 3D 캐릭터 모델 준비 (선택)

모델 없이도 기본 박스맨으로 즉시 사용 가능합니다.

Mixamo 캐릭터 사용 시:
1. [mixamo.com](https://www.mixamo.com) 접속 (Adobe 계정 무료)
2. Characters 탭 → `Y Bot` 선택
3. Download → Format: `GLTF` 선택
4. `character.glb`로 이름 변경 후 `models/` 폴더에 저장
5. 브라우저 컨트롤에서 모델 드롭다운을 `character.glb`로 변경

## 사용법

1. 브라우저에서 앱 열기
2. **웹캠 시작** 버튼 클릭 → 카메라 권한 허용
3. 카메라 앞에서 팔/다리/몸통을 움직이면 3D 캐릭터가 따라함
4. 스무딩 슬라이더로 반응 속도 조절 (0 = 즉각, 0.95 = 매우 부드럽게)
5. 마우스로 3D 뷰 회전/확대 가능

## 파일 구조

```
BodyBot/
├── index.html       ← 모든 로직 (단일 파일)
├── models/
│   └── character.glb  ← 선택사항 (없으면 박스맨 사용)
└── README.md
```

## 기술 스택

- **MediaPipe Pose** — 33개 신체 랜드마크 실시간 감지
- **Three.js r158** — 3D 렌더링, 뼈대 제어
- **순수 HTML/CSS/JS** — 빌드 도구 불필요, CDN만 사용
