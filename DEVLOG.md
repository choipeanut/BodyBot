# BodyBot 개발 로그 — 명령 & 실행 기록

> 실시간 신체 모션 캡처 → 3D 캐릭터 미러링 웹앱  
> 브랜치: `claude/motion-capture-3d-character-1lXSa`

---

## 커밋 히스토리 요약

| # | 커밋 | 내용 |
|---|------|------|
| 1 | `ed677bb` | 초기 앱 생성 |
| 2 | `1076830` | 뼈 회전 로직 수정 + character.glb 기본 로드 |
| 3 | `82b7951` | 뼈 이름 자동 감지 |
| 4 | `d80b2a9` | 드래그앤드롭 + 파일 선택으로 모델 로드 방식 교체 |
| 5 | `59ff2df` | 줌·캐릭터 크기 슬라이더 범위 확장 |
| 6 | `e6e0857` | 힙 yaw(좌우 돌기) 추가 |
| 7 | `9455c18` | 몸 기울기(roll/pitch) 수정 — 전체 3D 방향 반영 |
| 8 | `8b45d6e` | 손가락 인식 추가 (MediaPipe Holistic) |
| 9 | `8606ae7` | 배경 밝기 조정 |

---

## 명령별 상세 기록

---

### 1. 최초 작업 명세 전달

**명령 (프롬프트)**
```
실시간 신체 모션 캡처 → 3D 캐릭터 미러링 프로그램 작업 명세서 [전체 스펙 문서]
```

**구현 내용**
- `index.html` 단일 파일로 전체 앱 작성 (빌드 도구 없음)
- **Three.js r158 CDN** — PerspectiveCamera, OrbitControls, GLTFLoader, 조명 3개, 그림자
- **폴백 박스맨** — `THREE.Bone` 계층으로 머리/몸통/팔×2/다리×2 리깅
- **MediaPipe Pose CDN** — 웹캠 스트림, 33개 랜드마크 추출
- **Canvas 2D 오버레이** — 관절·연결선 실시간 드로잉
- **좌표 변환** — MediaPipe → Three.js Vector3 → Quaternion.setFromUnitVectors
- **BONE_MAP** — Mixamo 뼈 이름 매핑 테이블
- **UI** — 스무딩 슬라이더, 캐릭터 크기, 모델 선택, 포즈 리셋, 배경 전환

**결과**
- `index.html`, `README.md` 생성 및 푸시
- 박스맨으로 즉시 실행 가능, `.glb` 파일 있으면 캐릭터 교체 가능

---

### 2. "잘 안돼. 몸을 잘 따라오지 않아. character.glb로 실행되게 해줘"

**문제 진단**
1. `setBoneDir` 에서 부모 본의 월드 쿼터니언을 무시하고 직접 `setFromUnitVectors` 적용 → 회전 틀어짐
2. 기본 모델이 박스맨 (character.glb가 기본이어야 함)

**수정 내용**
```
[기존] setFromUnitVectors(localAxis, worldDir) 직접 적용
[수정] 3단계 올바른 계산:
  1. bone.parent.getWorldQuaternion(parentWorldQ)
  2. localTarget = worldDir.applyQuaternion(parentWorldQ.invert())
  3. deltaQ = setFromUnitVectors(restDir, localTarget)
     finalQ = deltaQ * restQ
```
- `storeRestData()` 추가 — 자식 본 위치로 각 본의 레스트 방향 자동 계산
- `character.updateMatrixWorld(true)` 매 프레임 시작 시 호출
- `poseWorldLandmarks` 대신 `poseLandmarks` 사용 (미터 단위 오류 방지)
- HTML select 기본값 → `character.glb`
- 시작 시 `loadGLTFCharacter('./models/character.glb')` 우선 시도

**결과**
- 팔/다리/척추 방향 추적 정확도 개선
- character.glb 있으면 자동 로드, 없으면 박스맨 폴백

---

### 3. "다른 캐릭터를 아직 쓸 수가 없어"

**문제 진단**
- 뼈 수집 시 `isBone`만 체크 → GLB 익스포터에 따라 `Object3D`로 내보내는 경우 못 찾음
- BONE_MAP이 `mixamorig*` 이름만 하드코딩 → 다른 모델 이름 매핑 실패

**수정 내용**
```javascript
// 기존: isBone만 수집
character.traverse(obj => { if (obj.isBone) bones[obj.name] = obj; });

// 수정: 모든 named Object3D 수집
character.traverse(obj => { if (obj.name) bones[obj.name] = obj; });
```
- `resolveBoneMap()` 함수 추가
  - 1차: 정확한 이름 매칭
  - 2차: 대소문자 무시 부분 문자열 매칭
  - 지원 포맷: Mixamo, Reallusion CC, 일반 명명 규칙
- `showToast()` — 뼈 매핑 결과 시각적 피드백
  - 성공: `✓ 뼈대 11개 모두 매핑됨`
  - 부분: `✓ 뼈대 8/11개 매핑됨 (일부 누락)`
  - 실패: 실제 뼈 이름 일부 표시

**결과**
- 다양한 GLB 모델 자동 인식 가능

---

### 4. "애초에 모델 폴더도 없고, 내가 만들어서 넣어도 안된다고"

**문제 진단**
- 로컬 서버에서 `./models/character.glb` 경로로 파일을 제공해야 하는 구조
- 사용자가 파일 시스템에 직접 넣어도 서버가 서빙하지 않으면 접근 불가

**수정 내용**
- `gltfLoader.load(url)` → `gltfLoader.parse(arrayBuffer)` 방식으로 전환
- **드래그 앤 드롭** — 3D 패널에 GLB 파일을 끌어다 놓으면 즉시 로드
- **파일 선택 버튼** — `📂 GLB 불러오기` 버튼 → 파일 탐색기에서 선택
- 드롭 시 네온 테두리 하이라이트 오버레이 추가
- 시작 시 박스맨 기본 표시 (GLB fetch 시도 제거)
- 기존 모델 드롭다운 → `박스맨` 버튼으로 단순화

**결과**
- 폴더 구조, 서버 경로 완전히 불필요
- 어떤 GLB 파일이든 브라우저에 드래그 한 번으로 로드

---

### 5. "캐릭터 크기 많이 줄일 수 있게 해줘. 아니면 스크롤로 멀어질 수 있게 하던가"

**수정 내용**

| 항목 | 기존 | 수정 |
|------|------|------|
| OrbitControls minDistance | 1 | 0.5 |
| OrbitControls maxDistance | 12 | **50** |
| 크기 슬라이더 min | 0.5 | **0.05** |
| 크기 슬라이더 max | 3 | **5** |
| 크기 슬라이더 step | 0.1 | 0.05 |

**결과**
- 스크롤 휠로 매우 멀리 줌아웃 가능
- 슬라이더로 캐릭터를 거의 점 크기까지 축소 가능

---

### 6. "몸의 각도도 반영을 해줘라"

**문제 진단**
- 힙 본에 아무 회전도 적용 안 됨 → 몸이 좌우로 돌아도 캐릭터는 정면만 봄

**수정 내용**
- 힙 본에 **yaw 회전** 추가
```javascript
// 어깨 + 힙 수평 벡터의 평균으로 yaw 계산
const shLat = lShoulder - rShoulder (XZ 투영)
const hpLat = lHip - rHip (XZ 투영)
const lat   = normalize(shLat + hpLat)
const yaw   = -atan2(lat.z, lat.x)  // 미러 보정
→ Quaternion.setFromAxisAngle(Y, yaw) → hipBone
```

**결과**
- 몸을 좌우로 돌면 캐릭터도 같이 회전

---

### 7. "몸 안기울어지잖아! 내가 몸을 옆으로 비틀면 얘도 비틀어야해"

**문제 진단 (2가지)**
1. `poseWorldLandmarks`는 미터 단위(예: x=0.15m)인데 `(x - 0.5)` 변환 적용 → 기울기 계산 완전히 망가짐
2. 힙에 yaw만 적용, 좌우 기울기(roll)·전후 기울기(pitch) 누락

**수정 내용**
```javascript
// 1. worldLandmarks 제거 → poseLandmarks만 사용
applyPoseToModel(results.poseLandmarks); // 변경 전: || poseWorldLandmarks

// 2. 힙을 3D 회전 행렬로 완전 교체
bodyUp    = normalize(shoulderMid - hipMid)   // 몸의 위 방향
bodyRight = normalize(lShoulder - rShoulder)  // 몸의 오른쪽 방향
bodyFwd   = normalize(bodyRight × bodyUp)     // 앞 방향 (오른손 법칙)
bodyRightOrtho = normalize(bodyUp × bodyFwd) // 재정규화

mat    = Matrix4.makeBasis(bodyRightOrtho, bodyUp, bodyFwd)
worldQ = Quaternion.setFromRotationMatrix(mat)
→ hipBone.quaternion ← worldQ (부모 로컬 변환 적용)
```
- 힙 업데이트 직후 `character.updateMatrixWorld(true)` 호출 → 척추/팔/다리가 새 힙 기준으로 동일 프레임 내 계산

**결과**
- 좌우 기울기(roll) + 좌우 돌기(yaw) + 전후 기울기(pitch) 모두 반영
- 척추 본은 상체 상대 기울기만 추가로 담당 (이중 반영 없음)

---

### 8. "손가락까지 인식할 수 있도록 해주렴"

**수정 내용**
- **MediaPipe Pose → Holistic** 전환 (포즈 + 양손 손가락 동시 인식)
  ```
  CDN: @mediapipe/holistic@0.5.1675471629
  결과: results.poseLandmarks + results.leftHandLandmarks + results.rightHandLandmarks
  ```
- BONE_MAP에 손가락 30개 추가
  ```
  엄지 3마디 × 2손 = 6
  검지/중지/약지/새끼 각 3마디 × 4손가락 × 2손 = 24
  합계 30개 (Mixamo 명명 + 일반 패턴 후보)
  ```
- `FINGER_SEGS_L` / `FINGER_SEGS_R` 정의
  ```javascript
  // MediaPipe Hands 21점 기준
  // 엄지: 1→2, 2→3, 3→4
  // 검지: 5→6, 6→7, 7→8  ... 새끼: 17→18, 18→19, 19→20
  ```
- `applyHandToModel()` — 손 랜드마크 세그먼트 방향을 `setBoneDir`로 각 마디 본에 적용
- 오버레이 업그레이드
  - 파란색 `#44ddff` → 왼손
  - 주황색 `#ffaa44` → 오른손

**결과**
- 손가락 굽힘/펴기, 각 마디 방향 실시간 반영
- GLB 캐릭터에 손가락 본이 있어야 동작 (Mixamo Y Bot 등 포함)
- Holistic은 Pose보다 무거워 저사양 기기에서 fps 저하 가능

---

### 9. "배경 좀 밝게 해주셈"

**수정 내용**

| 항목 | 기존 | 수정 |
|------|------|------|
| 배경색 | `#0a0a1a` (거의 검정) | `#2a2a3a` (어두운 보라-회색) |
| 안개색 | `#0a0a1a` | `#2a2a3a` |
| 바닥색 | `#050510` | `#1a1a2e` |
| 그리드색 | `#111133` | `#3a3a5a` |
| 배경 드롭다운 '어둡게' 기본값 | `#0a0a1a` | `#2a2a3a` |

**결과**
- 전체적으로 밝아진 다크 테마 유지
- 배경 드롭다운 '밝게' 선택 시 `#dde8ff` (하늘색)으로 전환 가능

---

## 현재 앱 기능 요약

### 인식 범위
| 부위 | 인식 방법 | 본 적용 |
|------|-----------|---------|
| 몸통 전체 방향 | 어깨+힙 3D 프레임 | Hips |
| 상체 기울기 | 어깨 중점 - 힙 중점 | Spine |
| 고개 | 코 - 어깨 중점 | Neck |
| 팔 (상완/전완) | 어깨→팔꿈치→손목 | LeftArm, LeftForeArm 등 |
| 다리 (대퇴/하퇴) | 힙→무릎→발목 | LeftUpLeg, LeftLeg 등 |
| 손가락 (각 마디) | MediaPipe Holistic 21점 | Thumb1~3, Index1~3 등 |

### 모델 로드 방법
- GLB 파일을 3D 뷰에 **드래그 앤 드롭**
- `📂 GLB 불러오기` 버튼으로 파일 선택
- 기본: 박스맨 (파일 없어도 즉시 실행)

### 컨트롤
| 컨트롤 | 기능 |
|--------|------|
| 스무딩 슬라이더 | 0 = 즉각 반응, 0.95 = 매우 부드럽게 |
| 캐릭터 크기 | 0.05 ~ 5배 |
| 마우스 드래그 | 3D 뷰 회전 |
| 스크롤 | 0.5 ~ 50 거리 줌 |
| 포즈 리셋 | 모든 본을 레스트 포즈로 복원 |
| 배경 전환 | 어둡게(#2a2a3a) / 밝게(#dde8ff) |

---

## 기술 스택

```
Three.js r158 (CDN)        — 3D 렌더링, 본 제어
MediaPipe Holistic 0.5     — 포즈 + 양손 손가락 인식
GLTFLoader (Three.js 애드온) — GLB 모델 파싱 (ArrayBuffer)
순수 HTML/CSS/JS            — 빌드 도구 없음, 단일 index.html
```
