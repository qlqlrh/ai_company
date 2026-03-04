# Design System — AI Company

> **용도**: 디자인 작업을 수행하는 모든 에이전트와 QA 에이전트가 공통으로 참조하는 시각 디자인 규칙 문서입니다.
>
> - **디자인 에이전트** (expert, 디자인 담당): 작업 전 이 문서를 읽고 규칙을 적용합니다.
> - **QA 에이전트**: ACTION_VISUAL 태스크 검증 시 이 문서의 기준을 적용합니다.

---

## 규칙의 두 레이어

```
┌─────────────────────────────────────────────────────────────┐
│  Layer A: 절대 규칙 (Non-Negotiable)                         │
│  레퍼런스가 있어도, CEO 지시가 있어도 반드시 지켜야 하는 것   │
│  → 글자 겹침 없음, 최소 폰트 크기, 텍스트 잘림 없음, 대비 등  │
├─────────────────────────────────────────────────────────────┤
│  Layer B: 스타일 결정 (우선순위 순서로 적용)                  │
│  1순위: ceo_references.analyzed_content.style_elements       │
│  2순위: ceo_instructions 명시적 지시                         │
│  3순위: 디자인 시스템 기본값 (없을 때만)                      │
└─────────────────────────────────────────────────────────────┘
```

---

## ★ Layer A: 절대 규칙

어떤 디자인 스타일을 따르든 **예외 없이** 지켜야 합니다.

### ① 텍스트 겹침 금지
```css
/* 모든 텍스트 컨테이너에 적용 */
overflow: hidden;
position: relative;  /* absolute 자식이 부모를 벗어나지 못하도록 */
```
- `position: absolute` 요소가 다른 텍스트 위에 겹치지 않는지 반드시 확인

### ② 텍스트 슬라이드 밖으로 잘림 금지
```css
.slide {
  overflow: hidden;
}
.text-block {
  word-break: keep-all;
  overflow-wrap: break-word;
  overflow: hidden;
}
```

### ③ 최소 폰트 크기 — 플랫폼별 기준

| 용도 | 일반 슬라이드 | **Instagram 카드뉴스** |
|------|-------------|----------------------|
| 본문 | 16px 이상 | **28px 이상** |
| 보조(캡션/번호) | 14px 이상 | **20px 이상** |
| 페이지 번호 | 12px 이상 | **16px 이상** |

> Instagram 카드뉴스는 스마트폰에서 1080px 이미지를 축소해서 봅니다.
> 실질적 표시 크기는 원본의 35~40% → **28px이 실제로는 약 10px로 보입니다.**

레퍼런스 이미지가 작은 텍스트를 쓰더라도 Instagram 카드뉴스 기준 이하는 금지.

### ④ 슬라이드 가장자리 최소 여백: 40px
```css
.slide {
  padding: 64px;  /* 기본값 */
}
/* 최소 40px는 유지 */
```

### ⑤ 텍스트-배경 최소 대비
- 본문 텍스트: 배경과 대비 4.5:1 이상
- 헤딩: 3:1 이상

### ⑥ 하단 회색 띠 금지 (body background 일치)
```css
/* body와 .slide의 background는 항상 동일 */
body { background: [배경색]; }
.slide { background: [배경색]; }  /* 동일한 값 */
```

### ⑦ 시리즈 전체 동일 폰트
- 같은 시리즈의 모든 슬라이드에 동일한 `font-family` 선언

---

## Layer B: 스타일 결정 (우선순위)

```
1순위: ceo_references.analyzed_content.style_elements  ← 레퍼런스 있으면 이것을 따름
       배경색, 폰트 패밀리, 색상 팔레트, 레이아웃 방식, 분위기

2순위: ceo_instructions 명시적 지시
       "다크 테마로", "민트 계열", "더 모던하게" 등

3순위: 디자인 시스템 기본값  ← 레퍼런스도 지시도 없을 때만
       라이트 에디토리얼 (크림 배경, Pretendard, 그린+앰버)
```

---

## 디자인 시스템

### 1. 컬러 시스템

**60-30-10 규칙**: 배경 60% + 서피스 30% + 강조 10%

**다크 테마 기본값:**
```css
--bg:             #0f0f17;
--surface:        #1a1a2e;
--surface-2:      #252540;
--primary:        #2563EB;
--accent:         #F59E0B;
--text:           #F0F0F0;
--text-secondary: #9090AA;
--border:         rgba(255,255,255,0.08);
```

**라이트 에디토리얼 (한국 카드뉴스 권장 — 기본값):**
```css
--bg:             #FAF7F3;
--surface:        #FFFFFF;
--surface-2:      #F5F2EC;
--primary:        #3D7A4D;
--accent:         #C96B1A;
--text:           #1A1A1A;
--text-secondary: #888888;
--border:         rgba(0,0,0,0.08);
--card-radius:    16px;
--card-shadow:    0 2px 12px rgba(0,0,0,0.06);
```

**컬러 사용 규칙:**
- 시리즈 전체에 동일한 `--primary`, `--accent` 사용
- 한 디자인에 최대 4가지 색상
- 그라데이션은 배경에만, 텍스트에는 금지
- 순수 #000000, #FFFFFF는 배경으로 사용 금지

---

### 2. 타이포그래피 시스템

**기본 폰트: Pretendard**

```html
<link rel="stylesheet" as="style" crossorigin
  href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/variable/pretendardvariable.min.css">
```

```css
font-family: 'Pretendard Variable', Pretendard, 'Noto Sans KR', sans-serif;
```

**크기 스케일 — Instagram 카드뉴스 기준:**
```
96px — 히어로
80px — Display XL
72px — Display L  (커버 타이틀 권장)
56px — Display M
48px — H1
40px — H2
32px — H3
28px — Body L     ★ Instagram 카드뉴스 본문 최솟값 ★
24px — Body M
20px — Caption L  (페이지 번호, 레이블)
16px — Caption S  (절대 최솟값)
```

**타이포그래피 규칙 (Instagram 카드뉴스):**
```css
/* 슬라이드 제목 */
font-size: 56px;
font-weight: 900;
line-height: 1.1;
letter-spacing: -0.03em;

/* 본문 — 반드시 28px 이상 */
font-size: 28px;
font-weight: 400;
line-height: 1.6;

/* 캡션 */
font-size: 20px;
font-weight: 500;
```

**★ 문장 내 bold/regular 혼합 패턴 (필수 기법):**
```html
<li><strong>터미널 기반</strong>으로 개발 환경에 직접 통합</li>
```
```css
li strong { font-weight: 700; color: inherit; }
li strong.accent { color: var(--primary); }
```

**계층 간격 규칙:** 인접한 두 폰트 크기의 비율은 **최소 1.25x**
- 나쁨: 28px, 32px (1.14x)
- 좋음: 28px, 40px (1.43x)

---

### 3. 레이아웃 & 간격 (8px 그리드)

```
4px  — 아이콘 내부
8px  — 인라인 요소 간격
16px — 컴포넌트 내부 패딩
24px — 컴포넌트 간 기본 간격
32px — 섹션 내 간격
48px — 섹션 간 간격
64px — 대형 섹션 간격
```

**카드/슬라이드 패딩:**
- 1080px 이상: 60-80px 사방
- **절대 금지**: 텍스트가 가장자리에서 24px 이하

---

### 4. HTML/CSS 구현 표준

**기본 HTML 구조 (라이트 에디토리얼):**
```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<link rel="stylesheet" as="style" crossorigin
  href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/variable/pretendardvariable.min.css">
<style>
* { margin: 0; padding: 0; box-sizing: border-box; -webkit-font-smoothing: antialiased; }

/* ★ body background = slide background 반드시 일치 ★ */
body {
  width: 1080px;
  height: 1350px;
  overflow: hidden;
  background: #FAF7F3;
  font-family: 'Pretendard Variable', Pretendard, 'Noto Sans KR', sans-serif;
}
.slide {
  width: 1080px;
  height: 1350px;
  background: #FAF7F3;  /* body와 동일 */
  padding: 72px 64px;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;
}
.section-last { flex: 1; }
</style>
</head>
<body>
<div class="slide"><!-- 내용 --></div>
</body>
</html>
```

---

### 5. Chrome Headless PNG 렌더링

```bash
# macOS
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --no-sandbox --disable-gpu \
  --screenshot="/path/to/output.png" \
  --window-size=1080,1350 \
  --hide-scrollbars \
  "file:///path/to/input.html" 2>/dev/null

# 렌더링 후 확인
ls -lh output.png  # 파일 크기 > 30KB여야 정상
```

**폴백 체인 (렌더링 실패 시):**
```
[Level 1] Chrome headless → 실패 시 Level 2
[Level 2] html2image (Python): pip install html2image
[Level 3] PIL/Pillow로 직접 이미지 생성
Level 3도 실패 시 → TOOL_ERROR 인터럽트
```

---

### 6. ❌ 절대 금지 패턴

**[패턴 1] 하단 회색 띠**
```
원인: body background ≠ .slide background
✅ 수정: body background = 마지막 섹션 배경과 일치, .section-last { flex: 1; }
```

**[패턴 2] 슬라이드 간 폰트 불일치**
```
✅ 수정: 시리즈의 모든 슬라이드에 동일한 <link> 복사
```

**[패턴 3] 폰트 3종 이상 혼용**
```
✅ 수정: 본문 폰트 1종 + 코드 폰트 1종만
```

**[패턴 4] 빈 공간을 억지로 채움**
```
✅ 수정: 하단 여백은 의도적 디자인 요소 (breathing room)
```

---

## ★ Design Brief (코드 작성 전 필수 작성)

**모든 디자인 작업을 시작하기 전에 아래 형식으로 Brief를 작성합니다.**
Brief 없이 코딩을 시작하면 일관성 없는 결과물이 나옵니다.

```
## Design Brief — [태스크명]

포맷: [가로×세로px] / 플랫폼: [Instagram 카드뉴스 / 포스터 / 웹]
총 슬라이드 수: [N장]

스타일 (Layer B 우선순위 적용):
  - 배경 톤: [라이트 에디토리얼 / 다크 / 기타]
  - 분위기: [미니멀 / 플레이풀 / 프로페셔널]
  - 레이아웃: [여백 기반 / 카드 박스 / 기타]

컬러 팔레트:
  - bg:       #_______
  - primary:  #_______
  - accent:   #_______
  - text:     #_______

타이포그래피 (Instagram 기준):
  - 제목:   폰트 56-72px / 900
  - 본문:   폰트 28-32px / 400  ← 최솟값 28px
  - 캡션:   폰트 20px / 500

슬라이드별 구성:
  [슬라이드 1]: Cover — [제목, 서브, 시각적 앵커]
  [슬라이드 2]: Body  — [시각적 앵커, 소제목, 항목]
  [마지막]:     Ending — [요약, CTA]
```

---

## ★ Self-Check Checklist (렌더링 전 필수)

```
[절대] 텍스트 안전:
  ☐ Instagram 카드뉴스 본문 28px 이상, 캡션 20px 이상
  ☐ 텍스트 겹침 없음: position:absolute 요소 확인
  ☐ 텍스트 잘림 없음: overflow:hidden + word-break:keep-all
  ☐ 슬라이드 가장자리 여백 40px 이상

[절대] 배경/렌더링:
  ☐ body background = .slide background (하단 회색 띠 방지)
  ☐ .slide에 overflow:hidden 설정
  ☐ Pretendard CDN 링크 존재

[절대] 대비:
  ☐ 본문 텍스트: 배경과 4.5:1 이상
  ☐ 헤딩: 3:1 이상

[절대] 완성도:
  ☐ placeholder 텍스트 없음 ("내용", "텍스트", "..." 등)

[품질] 타이포그래피:
  ☐ 계층 비율 1.25x 이상
  ☐ bold/regular 혼합 적용
  ☐ 폰트 크기 4종류 이하

[품질] 레이아웃:
  ☐ 슬라이드마다 시각적 앵커 1개 (아이콘/대형숫자/이모지)
  ☐ 간격이 8의 배수
  ☐ 흰 카드 박스 과사용 없음

[품질] 시리즈 일관성:
  ☐ 모든 슬라이드 동일 font-family
  ☐ --primary, --accent 전 슬라이드 동일
```

---

## 텍스트 오버플로우 방지 패턴

```css
/* 한 줄 텍스트 */
.title {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* 멀티라인 텍스트 */
.body-text {
  display: -webkit-box;
  -webkit-line-clamp: 4;
  -webkit-box-orient: vertical;
  overflow: hidden;
  word-break: keep-all;
}
```

---

## ACTION_VISUAL 태스크 완료 기준

| 산출물 | 완료 조건 |
|--------|---------|
| PNG 이미지 | `company/outputs/`에 PNG 존재 + 크기 > 30KB |
| HTML 슬라이드 | `company/outputs/`에 HTML 존재 + 크기 > 1KB |

❌ HTML만 있고 PNG 없음 (PNG가 요구된 경우) → 완료 아님
❌ 빈 파일(0 bytes) → 완료 아님
