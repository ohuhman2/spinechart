# SpineChart App — 전체 개발 매뉴얼

> **수정 전 반드시 이 문서를 먼저 읽을 것**  
> 클리닉: Bob Woo Chiropractic Clinic, Chatswood NSW  
> 앱 URL: https://ohuhman2.github.io/spinechart/SpineChart.html  
> GitHub: ohuhman2/spinechart  
> 작성일: 2026-05-31

---

## 1. 앱 개요

SpineChart는 카이로프랙틱 클리닉 전용 PWA(Progressive Web App)입니다.  
단일 HTML 파일(`SpineChart.html`) 하나로 모든 기능이 구현되어 있습니다.

**핵심 기능:**
- 환자 등록 및 관리
- 방문(Visit)별 기록 — 날짜, 호소, 통증, 자세 사진
- AI 자세 분석 (Claude API via Cloudflare Worker)
- 논문 기반 랜드마크 측정 및 결과 표시
- SOAP Note 자동 생성
- PDF 2종 출력 (환자용, 원장용)
- 치료 경과 분석 (회차별 비교)

---

## 2. 기술 스택

| 항목 | 내용 |
|---|---|
| 구조 | 단일 HTML 파일 (CSS + JS 내장) |
| 저장소 | localStorage (`spinechart_db` 키) |
| AI 분석 | XMLHttpRequest → Cloudflare Worker → Claude Sonnet API |
| AI API 모델 | claude-sonnet-4-20250514 |
| Cloudflare Worker URL | https://spinechart-proxy.ohuhman1.workers.dev |
| API 키 저장 | localStorage `sc_api_key` |
| 플랫폼 | iOS Safari PWA (홈화면 추가) 최적화 |
| 이미지 압축 | 최대 320px, JPEG quality 0.5 |
| PDF 출력 | window.open() → 새 창에서 인쇄 |

---

## 3. 화면 구조 (8개 화면)

```
스플래시 (splash)
    ↓ (1.9초 후 자동)
환자 목록 (s-list) ←── 항상 기본 화면
    ├─ [👤+] FAB 버튼 → 환자 등록 (s-add)
    └─ 환자 카드 탭 → 환자 상세 (s-detail)
                           ├─ [+ 재진] 버튼 → 방문 추가 (s-visit)
                           │      └─ AI 분석 결과 → 저장
                           ├─ 방문 카드 탭 → 방문 상세 (s-vd)
                           └─ [📊] 버튼 → 경과 분석 (s-prog)
                                              └─ [📄] 버튼 → 환자용 PDF

설정 (s-settings) ← 우상단 ⚙️ 버튼
```

### 각 화면 ID 및 역할

| 화면 ID | 역할 | 주요 함수 |
|---|---|---|
| `s-list` | 환자 목록 | `renderList()` |
| `s-add` | 새 환자 등록 | `savePatient()` |
| `s-detail` | 환자 상세 + 방문 목록 | `openDetail()`, `renderVisits()` |
| `s-visit` | 방문 추가 (사진+SOAP+AI) | `openNewVisit()`, `saveVisit()`, `runAI()` |
| `s-vd` | 방문 상세 보기/편집 | `openVD()`, `renderVD()`, `rerunAI()` |
| `s-prog` | 경과 분석 (회차 비교) | `openProgress()`, `renderProgress()` |
| `s-settings` | 설정 (API키, 클리닉정보, 백업) | `saveApiKey()`, `saveSettings()` |

---

## 4. 데이터 구조 (localStorage)

```json
{
  "patients": [
    {
      "id": "uid",
      "name": "환자 이름",
      "dob": "1990-01-01",
      "gender": "Male/Female",
      "phone": "0400 000 000",
      "occupation": "직업",
      "visits": [
        {
          "id": "uid",
          "date": "2026-05-31",
          "type": "Initial/Follow-up",
          "complaint": "주 호소",
          "pain": 5,
          "posture": ["FHP (Forward Head)", "Rounded Shoulders"],
          "photos": {
            "front": "base64...",
            "back": "base64...",
            "left": "base64...",
            "right": "base64..."
          },
          "soap": { "s": "...", "o": "...", "a": "...", "p": "..." },
          "analysis": {
            "measurements": { ... },
            "landmarks": { ... },
            "postureFindings": [...],
            "overallStatus": "normal|mild|moderate|significant",
            "overallMessage": "...",
            "overlayedPhotos": {
              "front": "base64 (사진+오버레이 합성)",
              "back": "base64",
              "left": "base64",
              "right": "base64"
            }
          }
        }
      ]
    }
  ],
  "settings": {
    "name": "Bob Woo Chiropractic Clinic",
    "doctor": "Dr. Bob Woo",
    "address": "408/71-73 Archer St, Chatswood NSW 2067",
    "phone": "0430 460 941",
    "website": "https://bobwoochiropracticclinic.com.au"
  }
}
```

---

## 5. AI 자세 분석 구조

### 분석 흐름
```
사진 촬영 (front/back/left/right)
    ↓ compressImg() — 320px, JPEG 0.5
callPostureAI()
    ↓ XMLHttpRequest (iOS PWA는 fetch 불가)
Cloudflare Worker (CORS 처리)
    ↓
Claude Sonnet API
    ↓ JSON 응답
drawOverlay() — 화면에 랜드마크 표시
applyFaceBlurCSS() — 얼굴 블러
aiResult 저장 → compositePhoto() — 사진+오버레이 합성 (PDF용)
renderResult() — 측정 결과 표시
SOAP O칸 자동 입력
```

### AI 응답 JSON 구조
```json
{
  "measurements": {
    "cva": 48.2,
    "shoulderAngle": 1.8,
    "hipAngle": 0.9,
    "headTilt": 1.2,
    "lateralShift": 1.5,
    "pelvicTilt": 11.0,
    "shoulderProt": 2.8,
    "effectiveHeadWeight": 6.2,
    "headShiftCm": 2.1,
    "shoulderShiftCm": null,
    "hipShiftCm": null
  },
  "landmarks": {
    "front": { "leftEar": {"x":0.35,"y":0.08}, ... },
    "back": { ... },
    "left": { "ear": {"x":0.55,"y":0.08}, "c7": {"x":0.48,"y":0.18}, ... },
    "right": { ... }
  },
  "postureFindings": ["CVA: 48.2° — Mild FHP ...", ...],
  "overallStatus": "mild",
  "overallMessage": "..."
}
```

---

## 6. 랜드마크 정의 (논문 기반)

> **⚠️ 핵심: 랜드마크 좌표는 이미지 비율 0.0~1.0 (x: 좌=0 우=1, y: 상=0 하=1)**

### 정면 / 후면 (front / back)

| 랜드마크 키 | 해부학적 위치 | 논문 근거 |
|---|---|---|
| `leftEar` / `rightEar` | **귓구슬 (Tragus)** — 귀 연골의 작은 돌기 | Kendall 2005 |
| `nose` | 코끝 | — |
| `leftShoulder` / `rightShoulder` | **Acromion 최외측 끝** — 어깨 가장 바깥 뼈 | Kendall 2005 |
| `leftHip` / `rightHip` | **ASIS** (전상장골극) 정면 / **PSIS** 후면 | Mazhar 2021 |
| `leftKnee` / `rightKnee` | 슬개골 중심 | Kendall 2005 |
| `leftAnkle` / `rightAnkle` | **외측복사뼈 (Lateral Malleolus)** | Kendall 2005 |

### 측면 (left / right)

| 랜드마크 키 | 해부학적 위치 | 논문 근거 |
|---|---|---|
| `ear` | **귓구슬 (Tragus)** — CVA 측정 기준점 | Shaghayegh Fard 2016 |
| `c7` | **C7 극돌기** — 목 가장 아래 튀어나온 뼈 | Shaghayegh Fard 2016 |
| `shoulder` | **Acromion 외측** | Kendall 2005 |
| `hip` | **Greater Trochanter (대전자)** — 허벅지 바깥 돌출부 | Kendall 2005 |
| `knee` | **비골두 (Fibular head) 약간 전방** | Kendall 2005 |
| `ankle` | **외측복사뼈 약간 전방** | Kendall 2005 |

---

## 7. 측정 항목 및 정상 기준값 (METRICS)

```javascript
const METRICS = [
  { id:'cva',               label:'CVA (Craniovertebral Angle)', unit:'°',  ok:50,  warn:45,  higher:true,  ref:'Shaghayegh Fard 2016' },
  { id:'shoulderAngle',     label:'Shoulder Level',              unit:'°',  ok:2,   warn:5,                 ref:'Kendall 2005' },
  { id:'hipAngle',          label:'Hip Level',                   unit:'°',  ok:1,   warn:3,                 ref:'Kendall 2005' },
  { id:'headTilt',          label:'Head Tilt',                   unit:'°',  ok:2,   warn:5,                 ref:'Kendall 2005' },
  { id:'lateralShift',      label:'Lateral Shift',               unit:'%',  ok:2,   warn:5,                 ref:'PostureScreen/Boland 2015' },
  { id:'pelvicTilt',        label:'Anterior Pelvic Tilt',        unit:'°',  ok:13,  warn:18,                ref:'Mazhar 2021' },
  { id:'shoulderProt',      label:'Shoulder Protraction',        unit:'%',  ok:3,   warn:7,                 ref:'PostureScreen Mobile' },
  { id:'effectiveHeadWeight',label:'Eff. Head Weight',           unit:'kg', ok:5.5, warn:10,                ref:'Hansraj 2014' },
]
```

**주의:** `higher:true`인 항목(CVA)은 값이 높을수록 정상 (반대 로직)

### 논문 레퍼런스 요약

| 논문 | 측정 항목 | 정상값 |
|---|---|---|
| Shaghayegh Fard et al. (2016) J Bodyw Mov Ther | CVA (두경추각) | >50° 정상, 45-50° 경미, <45° 심각 |
| Kendall FP et al. (2005) Muscles: Testing & Function 5th Ed. | 어깨/골반/머리 기울기, Plumb Line | 어깨 <2°, 골반 <1°, 머리 <2° |
| Hansraj KK (2014) Surgical Technology International | 유효 머리 하중 | 0°=4.5kg, 15°=12kg, 30°=18kg, 45°=22kg |
| Mazhar A et al. (2021) | 골반 전방경사 | 남 ≤9.5°, 여 ≤13° |
| PostureScreen Mobile / Boland et al. (2015) | 측방 이동, 어깨 전방 | <2%, <3% |

---

## 8. 오버레이 시각화 (drawOverlay)

### 색상 코드 (색상별 의미)
| 색상 | 랜드마크 | 의미 |
|---|---|---|
| 🔵 `#00b4d8` | 귀, 코 | 머리 기준점 |
| 🟡 `#FFD600` | 어깨 (Acromion), C7 | 어깨/경추 기준점 |
| 🔴 `#ff6b6b` | 골반 (ASIS/대전자) | 골반 기준점 |
| 🩵 `rgba(0,180,216,.8)` | 무릎 | 하지 정렬 |
| ⚪ `#a8dadc` | 발목 | 기저점 |

### 선 종류
- **초록 점선** — 수직 중력선 (Plumb Line, 발목 중심 기준)
- **초록 실선** — 귀 수평선
- **노란 실선** — 어깨 수평선
- **빨간 실선** — 골반 수평선
- **파란 점선** — 무릎/발목 수평선
- **노란 실선 (측면)** — C7→Tragus (CVA 측정선)

---

## 9. 얼굴 블러 (Privacy Protection)

### 화면 표시용 (CSS 방식)
```
applyFaceBlurCSS(img, lm, slot, side)
→ backdrop-filter: blur(18px) 적용한 타원형 div를 귀 위치에 overlay
→ 호주 Privacy Act 준수 목적
```

### PDF 출력용 (픽셀레이션 방식)
```
pixelBlurFace(ctx, lm, W, H, side)
→ 캔버스에서 귀 좌표 기반 영역을 다운샘플링 후 업샘플링
→ compositePhoto() 내에서 호출
```

---

## 10. 사진 합성 구조 (PDF용)

```
compositePhoto(side)
→ 원본 사진 캔버스에 그림
→ pixelBlurFace() 호출 (얼굴 블러)
→ 오버레이 캔버스(랜드마크+선) 위에 합성
→ JPEG base64로 반환
→ aiResult.overlayedPhotos[side]에 저장
→ PDF buildPatientViewSection()에서 사용
```

**⚠️ 현재 알려진 문제:**
- `compositePhoto()`는 `runAI()` 완료 후 500ms 딜레이 후 실행됨
- 사진 저장 전 PDF 출력하면 오버레이 없는 원본 사진이 PDF에 들어감
- `buildPatientViewSection()`이 `overlayedPhotos` 없으면 원본 `photos`로 폴백

---

## 11. PDF 출력 2종

### 환자용 PDF (`printPatientReport()`)
- 경과 분석 화면(s-prog)에서 📄 버튼
- **포함 내용:** 클리닉 헤더, 환자 정보, 회차별 자세 사진+측정값, Clinical Findings, Measurement Trend 표
- **사진:** `overlayedPhotos` (랜드마크+블러 합성) 우선, 없으면 원본 `photos`

### 원장용 PDF (`printSOAPReport()`)
- 방문 상세(s-vd)에서 SOAP 출력 버튼
- **포함 내용:** 클리닉 헤더, SOAP Notes 전문, 자세 태그
- **필터:** 전체 / 날짜별 / 방문유형별 선택 가능

---

## 12. 핵심 함수 목록

| 함수 | 역할 | 주의사항 |
|---|---|---|
| `init()` | 앱 초기화, localStorage 로드 | 앱 시작 시 1회 실행 |
| `renderList()` | 환자 목록 렌더링 | 검색 필터 포함 |
| `openDetail(pid)` | 환자 상세 열기 | `curPtId` 설정 |
| `openNewVisit(pid)` | 재진 방문 추가 화면 | `curPtId` 설정, 사진/AI 초기화 |
| `saveVisit()` | 방문 저장 | photos, soap, analysis 모두 포함 |
| `runAI()` | AI 분석 실행 (신규 방문) | XHR 사용 (fetch 아님) |
| `rerunAI(vid)` | AI 재분석 (방문 상세) | 분석 결과를 visit에 직접 저장 |
| `callPostureAI()` | AI API 실제 호출 | Cloudflare Worker 경유 |
| `drawOverlay()` | 화면에 랜드마크 오버레이 | 캔버스 z-index:2 |
| `applyFaceBlurCSS()` | 화면 얼굴 블러 | CSS backdrop-filter |
| `compositePhoto()` | 사진+오버레이 합성 (PDF용) | Promise 반환 |
| `pixelBlurFace()` | PDF용 얼굴 블러 | 픽셀레이션 방식 |
| `renderResult()` | 측정 결과 UI 렌더링 | METRICS 배열 참조 |
| `buildPatientViewSection()` | PDF 사진+측정값 섹션 | overlayedPhotos 우선 |
| `printPatientReport()` | 환자용 PDF 출력 | s-prog에서 호출 |
| `printSOAPReport()` | 원장용 SOAP PDF | s-vd에서 호출 |
| `generateSOAP()` | 신규 방문 SOAP 자동 생성 | s-visit에서 호출 |
| `generateSOAPforVD()` | 방문 상세 SOAP 재생성 | s-vd에서 호출 |
| `go(id)` | 화면 전환 | hist 배열에 push |
| `goBack()` | 뒤로 가기 | hist 배열에서 pop |
| `save()` | localStorage 저장 | 모든 데이터 변경 후 호출 |
| `load()` | localStorage 로드 | init()에서만 호출 |

---

## 13. 수정 시 체크리스트

수정 전 반드시 확인:

- [ ] 수정하는 함수가 어떤 함수와 연결되어 있는지 확인
- [ ] `METRICS` 배열 변경 시 → `renderResult()`, `buildPatientViewSection()`, `printPatientReport()` 모두 영향
- [ ] 랜드마크 키 이름 변경 시 → AI 프롬프트 + `drawOverlay()` + `applyFaceBlurCSS()` + `pixelBlurFace()` 모두 수정
- [ ] PDF 출력 수정 시 → `buildCSS()`, `buildPatientViewSection()`, `printPatientReport()`, `printSOAPReport()` 확인
- [ ] `runAI()` 수정 시 → `rerunAI()`도 동일하게 수정 (두 함수가 병렬 존재)
- [ ] 화면 추가 시 → `go()` / `goBack()` / `hist` 배열 동작 확인
- [ ] **기존 기능 절대 삭제 금지** — 추가만 할 것

---

## 14. 현재 알려진 문제 / TODO

| 우선순위 | 문제 | 원인 | 해결 방향 |
|---|---|---|---|
| 🔴 높음 | PDF에 랜드마크 오버레이 없음 | `compositePhoto()` 타이밍 문제 + `rerunAI()` 미저장 | 저장 완료 후 PDF 출력 보장 |
| 🔴 높음 | PDF에 얼굴 노출 | CSS blur는 화면만, PDF는 별도 처리 필요 | `pixelBlurFace()` 적용 확인 |
| 🟡 중간 | AI 분석 후 바로 PDF 출력 불가 | 분석→저장→방문상세→PDF 흐름이 너무 긺 | AI 분석 완료 후 바로 PDF 버튼 추가 |
| 🟡 중간 | `buildPatientViewSection()` 측면 측정값 | `fhp` 키로 참조하나 AI는 `cva`로 반환 | 측면 섹션 측정값 키 업데이트 |
| 🟢 낮음 | 환자용 PDF 설명이 기술적 | 환자가 이해하기 어려운 용어 | 환자 친화적 설명 추가 |

---

## 15. 캐시 클리어 방법

GitHub에 올린 후 URL에 버전 번호 추가:
```
https://ohuhman2.github.io/spinechart/SpineChart.html?v5
```
매번 숫자 올리면 됨 (`?v2` → `?v3` → `?v4` ...)

---

## 16. 클리닉 정보

| 항목 | 내용 |
|---|---|
| 클리닉명 | Bob Woo Chiropractic Clinic |
| 원장 | Dr. Bob Woo |
| 전화 | 0430 460 941 |
| 주소 | 408/71-73 Archer St, Chatswood NSW 2067 |
| 웹사이트 | bobwoochiropracticclinic.com.au |
