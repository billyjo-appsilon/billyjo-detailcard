# 빌리조 AI 상세페이지 — 시스템 컨텍스트 메모

> **문서 위치**: `billyjo_ai_detailpage_rulebook.md` 와 함께 프로젝트 지식 베이스에 업로드.
>
> **역할**: 룰북이 "무엇을 평가할지"를 정의한다면, 이 문서는 "어떻게 작동시킬지"(아키텍처)와 "어떻게 보일지"(카드 디자인 사양)를 정의한다.

---

## 1. 시스템 아키텍처

### 1.1 핵심 결정: 실시간 AI 호출 X, 사전 생성 + 캐싱

| 시점 | 동작 | AI 호출 여부 |
|------|------|--------------|
| 제품 등록 | 5단계 파이프라인 → DB 저장 | ✅ 1회만 |
| 사용자 페이지 진입 | GTM이 page_view 트리거 → API로 캐시 조회 → DOM 렌더링 | ❌ 없음 |

GTM은 **단순 트리거 + 데이터 바인더** 역할만 한다. AI 추론은 등록 시점에 끝나 있어야 함.

### 1.2 제품 등록 시 5단계 파이프라인

```
1. 제품 등록 (관리자 입력)
   ↓ webhook
2. 출처 크롤 (제조사 공식 + 다나와)
   ↓
3. AI 추출 (패밀리 룰북 스키마에 맞춰)
   ↓
4. 교차 검증 (다중 출처 비교 + 신뢰도 점수)
   ↓
5. JSON 저장 (DB)
```

각 단계는 독립 실행 가능해야 함 — 한 단계 실패 시 그 단계만 재실행하면 됨.

### 1.3 GTM 측 처리 흐름

```
[유저 페이지 진입]
   ↓
[GTM: page_view 트리거 발화]
   ↓
[Custom HTML 태그: fetch(`/api/card/${productId}`)]
   ↓
[JSON 응답 → 페이지 내 컨테이너에 바인딩]
   ↓
[CSS·SVG로 렌더링 (이미 정의된 템플릿)]
```

GTM 안에서 AI를 호출하지 않는다. 캐시된 JSON만 가져온다. 응답 속도는 백엔드 API + 캐시 레이어 성능에 좌우됨 — 100ms 이내 목표.

---

## 2. 4중 검증 메커니즘

신뢰할 수 없는 정보가 카드에 노출되지 않도록 4개의 안전장치를 거친다.

| # | 메커니즘 | 작동 방식 |
|---|----------|----------|
| 1 | Source-grounded 프롬프팅 | AI에게 "크롤된 원문에 없는 내용은 절대 생성 금지" 지시. 추론·일반화 차단. |
| 2 | 필드별 출처 기록 | 모든 추출값에 `source_url` + `quoted_text` (원문 인용 한 줄) + `confidence` (0-1) 부여. |
| 3 | 다중 출처 교차 검증 | 핵심 항목(필터 종류, 살균 방식, 에너지 등급 등)은 제조사 공식 + 2차 출처(다나와 등)에서 동시에 확인. 두 값이 다르면 검토 큐로. |
| 4 | 신뢰도 임계치 | `confidence < 0.7` 또는 단일 출처에서만 확인된 핵심 항목은 자동으로 직원 검토 큐로 이동. |

### 2.1 빌리조 자체 상세페이지 정보의 위치

**중요 (v0.2)**: 빌리조(billyjo.co.kr) 제품 상세페이지에 등록된 정보는 **본사(제조사 또는 정식 유통사)가 직접 제공한 자료**이므로 1순위 출처에 준한다.

따라서:
- **공식 사이트(coway.com 등)에서 정보가 확인되지 않더라도** 빌리조 상세페이지에 명시된 스펙·기능은 신뢰 (`confidence ≥ 0.85`)
- 빌리조 + 공식 사이트 둘 다 확인되면 `confidence = 0.95+` (최고 신뢰)
- 빌리조에만 있고 공식 사이트는 정보 없음 → 단일 출처지만 **검수 큐 트리거에서 제외** (빌리조 자체가 본사 제공이므로)
- 빌리조와 공식 사이트가 다르면 빌리조 우선 (본사가 빌리조 측에 최신 정보 제공했을 가능성)

**출처 우선순위 갱신**:
1. 빌리조 상세페이지 + 제조사 공식 (둘 다 일치) → confidence 0.95+
2. 빌리조 상세페이지 단독 → confidence 0.85+ (본사 제공 신뢰)
3. 제조사 공식 단독 → confidence 0.85
4. 다나와 등 가격비교 단독 → confidence 0.6 (검수 큐 후보)
5. 어디에도 없음 → 평가 없음 (D 처리 + 4지표 중 3개 이상이면 카드 검수 큐)

### 2.2 제조사 공식 페이지 — 스펙 표 우선 참조 (v0.2)

**중요**: 제조사 공식 페이지에서 정보를 가져올 때 **상세페이지 전체를 읽지 말고, 페이지 하단의 "제품 스펙 표"만 참조**한다. 그 표에는 제품 모델명과 정식 스펙이 정리되어 있어 가장 신뢰성 있고 토큰·시간 효율적.

**작업 순서**:
1. 제품 페이지 진입 → 페이지 끝까지 빠르게 스크롤
2. 페이지 하단의 스펙 표(보통 `<table>` 또는 dl 형태)에서 모델명·제품명 일치 확인
3. 스펙 표 내용을 우선적으로 추출 (브랜드/방식/타입/관리/기능/규격/관리주기/렌탈정보 등)
4. 스펙 표에 없는 항목만 본문(상세설명 이미지·텍스트) 보충 참조

**근거**: 본문 마케팅 텍스트·이미지는 표현이 모호·과장될 수 있고 추출 정확도가 낮음. 스펙 표는 본사가 정리한 구조화 데이터로 confidence 0.9+ 부여 가능.

**예시 (세스코 EWBD351)**:
- 스펙 표 9행에 모든 핵심 스펙 정리 (브랜드·방식·타입·관리·기능·월렌탈료·규격·관리주기·렌탈정보)
- AI 카드 데이터는 이 9행에서 직접 추출. 본문 이미지·상세설명은 보조 참조만.

**검수 큐 트리거 예시**
- 코웨이 CHP-7211N: 살균 방식이 공식 페이지에 명시 → confidence 0.95, 통과.
- 세스코 EWBD351: 능동 살균 정보가 공식 페이지에 없음, 빌리조 페이지에도 없음 → 위생관리 지표 `평가 없음` (D 처리), 카드는 노출 OK (4지표 중 1개만 D).
- 쿠쿠 CP-F603SW: 능동 살균 기능이 공식 페이지에 명시 없으나 빌리조 페이지에 "능동 살균 보유" 명시 → confidence 0.85, **통과** (빌리조 정보 신뢰).

---

## 3. 카드 디자인 사양

### 3.1 전체 카드 구조 (7개 슬롯, 위에서 아래 순서) — v0.2

```
[1. 헤더]              제품명 + "모델명: {번호}" + 한 줄 메타
                       모델번호는 var(--color-text-info) 파란색 Pretendard Bold
─────────────────────  border-bottom (구분선)
[2. 게이지 + 4지표]    SVG 270° 호 + 우측 4개 메트릭 등급 카드
                       게이지 중앙: 종합 평가 letter (큰 글씨) + ≈ 소수점평균 (보조)
                       4지표 셀: 라벨 + [한글 라벨 | letter] 배지
                       예: [최고 | S], [추천 | A], [평가 없음]
[Hero] (v0.2.7) .prod_view_top.right 안에 — 제품명 + 모델메타 + 가격박스 + 270° 게이지+4지표
                       (SLOT 1·2 이전 — 첫 화면에 평가 결과까지 노출, 의사결정 가속)

[3. 6-8칸 스펙요약]    3열 그리드 (배경: secondary)
                       기본 6 기술 스펙 + 선택 2 운영/계약 스펙 (방문관리·소유권 등)
                       방문관리 셀의 .sv는 .hq-btn pill로 시각 강조 (🏢 + 브랜드 파랑)
                       전문 용어 라벨 옆에 ⓘ ? 도움말 (클릭 popup)
                       v0.2.6: .rec 한 줄 산문 폐지 → 본 그리드로 흡수, 중복 제거
─────────────────────  border-top (섹션 구분)
[4. 기본정보 · 설치 형태] pill 옵션 (활성을 맨 앞에)
                       + 약정/의무사용/위약금 (.rental-terms)
                       + 핵심 강점 칩 영역 (.strengths, 4-6개)
                       강점: 가격 경쟁력 / 물리 강점 / 운영 강점 / 사은품
                       v0.2.6: .rec 산문 블록 제거 — SLOT 3·5와 중복 정보 단일화
─────────────────────
[5. 이런 분에게 추천해요 👀] Top 3 페르소나 카드 (3열, .rec-personas)
                       각 카드: 아이콘 + 이름 + [매우 추천/추천/권장] + 멘트 1줄
                              + 기능 버튼 3개
                       (% 표시 폐기, DB에만 저장)
─────────────────────
[6. 상세 스펙]         Step 1~3 (.step-card, 각 카드 사이 8px gap)
                       평상시: 번호·제목·등급 배지·핵심 요약 1줄
                       [자세히 보기 ▼] 클릭 → pill 옵션 전체 펼침
                       v0.2.7: 폰트 +35% 표준 (step-n 26px / title 16px / sum 14px)
                       v0.2.9: 카테고리형 스펙(필터·살균방식·컴프레서 등)은
                              .field.spec-compare 2단계 토글 (라벨 → sub-toggle → 비교표)
─────────────────────
[7. 예상 최대 지원금ㆍ빌리조 혜택 🎁] .gift-big 강조 박스 (그라데이션)
                       3행: [예상 최대 지원금] / [빌리조 단독] / [제조사 프로모션]
                       라벨 = 버튼형 (.gift-tag), 값의 핵심 수치는 파란색 Bold
```

### 3.1.1 카드 외부 위치·너비

```css
#ai-card-root {
  display: block;
  width: 50%;                    /* 빌리조 .prod_view_top > .right 와 동일 */
  margin: 0 0 0 auto;            /* 위·아래·좌측 모두 0, 우측 정렬은 auto */
  padding: 0 15px 20px;          /* .prod_table_wrap 좌우 15px와 동일 */
  box-sizing: border-box;
}
/* 1024-1279px: 빌리조 .right가 padding-left:10px 추가 → 카드도 동일 보정 */
@media (max-width: 1279px) and (min-width: 1024px) {
  #ai-card-root { padding-left: 25px }   /* 15 + 10 */
}
/* ≤1023px: 빌리조 컬럼이 풀폭 스택 → 카드도 풀폭 */
@media (max-width: 1023px) {
  #ai-card-root { width: 100%; margin: 0; padding: 0 15px 20px }
}
/* ≤600px: 좁은 모바일 정밀 패딩 (top 12px / right 7px / bottom 8px / left 0) */
@media (max-width: 600px) {
  #ai-card-root { padding: 12px 0 0 0; margin: 0 }
}
```

**중요 (v0.2)**:
- **마진은 모두 0** — 카드와 빌리조 콘텐츠 사이 수직 여백은 카드 자체 그림자·테두리로 분리. 외부 margin으로 띄우지 않음 (제거 대상 박스와 너비가 똑같이 정렬되려면 마진도 동일해야 함).
- **`padding-left: 25px` 1024-1279px 보정**의 근거: 빌리조 `.prod_view_top > .right`가 이 구간에서 `padding-left: 10px`이 들어가는데(`/css/sub_m.css`), 카드는 `.wide-inner` 직속이라 자동 보정이 안 됨 → CSS로 명시적 보정 필요.

삽입 위치: `<!-- 상품정보 -->` 섹션 (`.prod_view_bot`) 바로 위. 카드 위에는 빌리조 기존 이미지·가격·기본정보 영역이 유지된다.

### 3.2 종합 평가 게이지 SVG 사양 (v0.2.2 — 2줄 단순화)

```
viewBox: "0 0 100 100"
크기: 100×100 px (이전 92px → 확대)
호 경로: M 25.25 74.75 A 35 35 0 1 1 74.75 74.75
호 각도: 270° (좌하 → 우하)
선 두께: stroke-width="6"
선 끝 모양: stroke-linecap="round"

배경 호: stroke="#dfdfdf" (var(--color-border-tertiary))
점수 호: stroke="#0838F8" (var(--color-text-info))
점수 길이: stroke-dasharray="{score × 1.65} 165"
         (165 = 270° 호의 전체 길이, score=100일 때 가득 참)

중앙 텍스트 (Pretendard, 정확히 2줄):
  "종합 평가" (위, y=40, font-size 9, var(--color-text-secondary))
  "{letter}"  (가운데, y=62, font-size 26, weight 700, 등급 컬러)
             예: "A" (등급 A의 초록색)

두 텍스트 사이 약 2px 시각적 간격. letter는 호의 중심에 정렬.
```

**중요 (v0.2.2)**: 보조 텍스트 표시 금지. "≈ 2.0", "3지표 기준 · 2.0" 같은 소수점·산출 방식 라벨은 게이지에 노출하지 않는다. 부분 평균 계산(평가 없음 지표 제외)도 letter 산출 로직에만 사용 — 게이지는 항상 단일 letter로 표시되어 시각 단순화. 산출 디테일(평균 소수점·제외 지표 수)이 필요하면 카드 외부 메타 영역이나 검수 큐 백오피스에서 확인.

### 3.3 점수·등급 산정 공식

**개별 지표 점수 → letter 등급** (절대 규칙 #13 매핑 적용):
```
S    = 90-100점
A+   = 85-89
A    = 80-84
B+   = 70-79
B    = 60-69
C+   = 50-59
C    = 40-49
평가 없음 = score < 40 OR confidence < 0.7 (letter "D" 사용 금지)
```

**종합 평가**:
```
종합 점수 = sum(평가 있는 지표 점수들) ÷ count(평가 있는 지표)
            ※ '평가 없음' 지표는 평균에서 제외 + 분모도 감소

종합 등급 = 종합 점수를 위 매핑표에 적용
```

**소수점 표기**: 종합 등급은 letter만 표시하고 소수점 평균은 보조 라벨로 별도 표기 (예: "A · ≈ 2.0").

지표 4개는 패밀리마다 다름 (룰북 참조). 모두 0~100 스케일.

### 3.3.1 등급 컬러 팔레트 (CSS 변수)

```css
--g-1:   #0838F8;  /* S = 최고 */
--g-1-5: #1a87ac;  /* A+ = 적극추천 */
--g-2:   #16a34a;  /* A = 추천 */
--g-2-5: #4ec727;  /* B+ = 우수 (예약) */
--g-3:   #84cc16;  /* B = 좋음 */
--g-3-5: #c2ce15;  /* C+ = 적합 (예약) */
--g-4:   #facc15;  /* C = 보통 */
--g-d:   #999;     /* 평가 없음 */
```

### 3.4 CSS 디자인 토큰

빌리조 사이트 실제 CSS에서 추출한 매핑값 (base.css·common.css·shop.css 빈도 1-3위 기반):

| 토큰 | 실제 hex | 용도 |
|------|---------|------|
| `--color-text-info` | **#0838F8** | 빌리조 브랜드 메인 컬러 (강조·등급 S·pill on) |
| `--color-text-primary` | #2a2a2a | 본문·제품명 |
| `--color-text-secondary` | #6a6a6a | 라벨·보조 텍스트 |
| `--color-text-tertiary` | #999 | 메타·pill off |
| `--color-background-primary` | #ffffff | 카드 배경 |
| `--color-background-secondary` | #f7f7f7 | 6스펙·페르소나·사은품 박스 배경 |
| `--color-background-info` | #e8edff | pill 활성·rec 형광펜·핵심강점 영역 배경 |
| `--color-border-tertiary` | #dfdfdf | 구분선·테두리·pill off 외곽 |
| `--border-radius-md` | 8px | 내부 박스 |
| `--border-radius-lg` | 12px | 카드 자체 |

빌리조 사이트는 아직 CSS 변수 정의가 없으므로 카드 `:root` 안에 폴백 정의 필요 (`#ai-card-root { --color-text-info: #0838F8; ... }`).

등급 컬러 9종(`--g-1` ~ `--g-d`)은 절대 규칙 #13 + 3.3.1 참조.

### 3.5 "이런 분에게 추천해요" 페르소나 카드 (v0.2)

```
3열 그리드 (.rec-personas), 각 카드 (.rec-p):
- 헤더 (.rec-p-head):
  - 좌측: Tabler Icons (18px, var(--color-text-info))
  - 중앙: 페르소나 이름 (.rec-p-title, 13px, Pretendard Bold)
  - 우측: 추천 강도 라벨 (Pretendard Bold)
    - 1순위: .rec-p-level-1 → "매우 추천" (파란 채움 배경)
    - 2순위: .rec-p-level-2 → "추천" (옅은 파란 배경, 파란 글씨)
    - 3순위: .rec-p-level-3 → "권장" (투명 배경, 파란 테두리·글씨)
- 추천 멘트 (.rec-p-msg, 11.5px, var(--color-text-secondary), 1줄):
  "[이 제품 강점 사용 시나리오] — 이래서 추천해요!" 톤
- 기능 버튼 3개 (.feat-btns):
  - .feat-btn 칩: 흰 배경 + 파란 테두리 + 파란 글씨, Pretendard Bold 11px
  - 강점 키워드 3개 (예: "230mm 슬림", "월 17,900원", "필터교환알림")
- 배경: var(--color-background-secondary), border 0.5px solid border-tertiary
```

**중요**: 구성비 %는 카드 표면에 노출 금지 (DB에만 저장). 사용자에겐 정성적 라벨(매우 추천/추천/권장) 만 노출 — 숫자 노출 시 "16%는 낮다" 잘못된 인식 유발.

### 3.6 상세 스펙 Step 카드 (v0.2 — 자세히 보기 접기)

```
.step-card (배경: secondary, padding 12px 16px)
├─ .step-head (가로 flex)
│  ├─ .step-n (22×22 원, 배경 info, 흰 글씨, Pretendard Bold 11px)
│  ├─ .step-title (13.5px, Pretendard Bold) ← "정수성능", "위생관리"
│  └─ .step-grade (우측 정렬, 등급 배지 [한글 라벨 | letter])
├─ .step-summary (12px, secondary, 1줄)
│  핵심 키워드는 <strong>로 강조 (Pretendard Bold + primary)
│  예: "<strong>냉수·온수·정수</strong> 3가지 출수 · <strong>UF 중공사막 4단계</strong>"
└─ <details class="step-details">  ← 평상시 접힘
   ├─ <summary>자세히 보기 ▼</summary>  (작은 파란 버튼)
   └─ .step-fields (펼침 시 노출)
      ├─ .field (각 field):
      │  ├─ .field-l (11.5px secondary, 항목명+(개수)+보조설명)
      │  └─ .pills (활성/비활성/본사관리 버튼 등 칩 나열)
```

### 3.7 Pill 옵션 + 본사관리 버튼 (v0.2)

**일반 pill**:
```
- 비활성 .pill: 테두리 0.5px var(--color-border-tertiary), color tertiary, 투명 배경
- 활성   .pill.on: color info, background-info, border info
- 크기:    font-size 11px, padding 4px 10px, border-radius 999px
- 표시:    각 옵션 텍스트 (✓ 등 별도 아이콘 금지, 색상으로만 구분)
- 활성 pill은 Pretendard Bold (font-weight:700)
```

**본사관리 버튼 (`.hq-btn`, 절대 규칙 #7)** — 본사·방문관리·코디 케어처럼 운영적 차별점이 있는 항목은 일반 pill 대신 별도 버튼:
```
- 배경: var(--color-text-info) (파란 채움)
- 글자: 흰색, Pretendard Bold 11px
- prefix 이모지: 🏢
- padding: 5px 12px, border-radius: 8px
- 표시:  "본사관리", "코디 방문관리" 등
```

### 3.8 핵심 강점 영역 (`.strengths`, v0.2 신설)

기본정보 슬롯 하단에 위치하는 강점 요약 영역:
```
- 배경: var(--color-background-info) (옅은 파란)
- 좌측 액센트: border-left: 3px solid var(--color-text-info)
- padding: 14px
- 내부 칩 (.strength-chip):
  - 흰 배경 + 옅은 파란 테두리 (#d6e0fb)
  - Tabler 아이콘 + 텍스트 (Pretendard Bold 12px)
  - padding 7px 12px, border-radius 8px
- chip 개수: 4-6개
- chip 종류 가이드:
  - 가격 경쟁력: 월 렌탈료, 약정 기간, 카드 할인 등
  - 물리 강점: 슬림 사이즈, 대용량, 디자인 등
  - 운영 강점: 본사관리, 4개월 코디, 무상 A/S 등
  - 사은품·혜택: 초기비용 0원, 평생보증 등
```

### 3.9 스펙요약 6-8칸 + 도움말 (`<details class="help">`, v0.2.6 확장)

**그리드 구조** — PC 3-col / 모바일 (≤600px) 2-col. 기본 6칸은 패밀리별 기술 스펙(룰북 §4 참조), 선택 7-8번째 칸은 빌리조 운영/계약 정보:

```html
<div class="specs">
  <!-- 1-6: 기술 스펙 (패밀리별) -->
  <div class="sc"><div class="sl">렌탈료</div><div class="sv">월 17,900원</div></div>
  <div class="sc"><div class="sl">필터종류 <details class="help">…</details></div><div class="sv">UF 중공사막</div></div>
  ...
  <!-- 7: 방문관리 (운영, .hq-btn pill) -->
  <div class="sc"><div class="sl">방문관리</div><div class="sv"><span class="hq-btn">4개월 코디</span></div></div>
  <!-- 8: 소유권 이전 (계약) -->
  <div class="sc"><div class="sl">소유권</div><div class="sv">60개월 후 자동 이전</div></div>
</div>
```

**중복 흡수 원칙 (v0.2.6 신설)**: 같은 정보가 SLOT 3·SLOT 4(`.strengths`·`.rental-terms`)·SLOT 5(페르소나 멘트)에 중복되지 않도록 단일화. 우선순위 `.specs > .strengths > .rec(폐기) > 페르소나 멘트`. 폐기된 `.rec` 한 줄 산문 블록은 v0.2.6에서 그리드로 흡수 후 제거.

**도움말 (`<details class="help">`)**
- ⓘ ? 아이콘: 15×15 원, 회색 테두리, 호버/오픈 시 파랑
- popup: 흰 배경, 300px 폭, 위쪽 꺾쇠 화살표, box-shadow
- 적용 대상 (소비자에게 어려운 용어):
  - F01: 필터 종류, 살균 방식
  - F03: 에너지 등급, 인버터 종류, HEPA 등급
  - F05: 컴프레서 종류, 메탈쿨링
  - F06: 모터 종류 (DD/BLDC/인버터), 건조 방식
  - F11: 매트리스 구조 (포켓스프링/메모리폼/하이브리드)
- 운영 셀(방문관리·소유권)은 일반 소비자에게 직관적이므로 도움말 미부착 원칙. 단 "평가 없음" 처리된 살균방식 등은 도움말로 처리 사유 안내.

### 3.10 사은품 박스 (`.gift-big`, v0.2 강화)

```
- 배경: linear-gradient(135deg, --color-background-info → #f5f8ff)
- 테두리: 1px solid #d6e0fb
- padding: 16px 18px

.gift-row (3행, 사이 점선 구분):
- .gift-tag (라벨, 버튼형):
  - 흰 배경 + 파란 테두리·글씨, Pretendard Bold 11px
  - min-width: 108px, 가운데 정렬
  - 라벨 종류:
    - [예상 최대 지원금] — 빌리조 내부 DB
    - [빌리조 단독] — 빌리조만의 혜택
    - [제조사 프로모션] — 제조사 자체 프로모션
- .gift-content (값):
  - 핵심 수치는 <strong> Pretendard Bold + var(--color-text-info) 파란
  - 예: "<strong>월 최대 25,000원 청구할인</strong>"
- 첫 행 값이 "내부 DB 연동 예정"이면 .gift-db-badge (파란 채움 배지)로 표기
```

### 3.11 헤더 모델명 표기 (v0.2 신설)

```html
<div class="meta">
  <span class="model-label">모델명: </span>
  <span class="model-num">EWBD351</span>
  · 데스크형 · 4개월 방문관리 · 230×517×510mm
</div>
```
- `.model-label`: 일반 폰트, var(--color-text-tertiary)
- `.model-num`: Pretendard Bold + var(--color-text-info) (파란 강조)

### 3.11.1 모바일 (≤600px) 컴팩트 사양 (v0.2.3)

좁은 화면에서 카드 길이를 줄이기 위한 박스 패딩·폰트 정밀 조정:

```css
@media (max-width: 600px) {
  /* 게이지 + 4지표 영역 */
  #ai-card-root .summary { gap: 10px; padding: 10px 0; flex-direction: column }
  #ai-card-root .mgrid { grid-template-columns: 1fr 1fr; gap: 6px }

  /* 4지표 카드 — 1줄 row 레이아웃 (라벨 좌·배지 우) */
  #ai-card-root .m {
    flex-direction: row;
    align-items: center;
    justify-content: space-between;
    padding: 7px 10px;
    gap: 6px;
  }
  #ai-card-root .m .ml { white-space: nowrap; flex-shrink: 0 }
  #ai-card-root .m .grade-badge { flex-shrink: 0; margin-left: auto }
  #ai-card-root .ml { font-size: 11px }
  #ai-card-root .grade-badge { font-size: 11.5px; padding: 3px 9px }

  /* 6스펙 */
  #ai-card-root .specs { grid-template-columns: 1fr 1fr; padding: 6px 4px }
  #ai-card-root .sc { padding: 5px 4px }
  #ai-card-root .sv { font-size: 13px; margin-top: 3px }
}
```

**중요 (v0.2.3)**:
- **4지표 카드는 모바일에서 1줄로 강제** — 데스크톱(column)과 다른 레이아웃. 카드 당 높이 약 28px (이전 column 시 약 50px) 절약.
- 라벨·배지는 `flex-shrink: 0`으로 줄어들지 않게. 라벨이 길어도 `white-space: nowrap`으로 한 줄 유지.
- 패딩 축소는 모바일 전용 (데스크톱 룰 유지).

### 3.12 카드 식별 라벨 (.ai-card-tag, v0.2.2)

카드 상단 좌측 모서리에 부착되는 브랜드 식별 배지.

```html
<div id="ai-card-root">
  <span class="ai-card-tag">빌리조 제품분석</span>
  <div class="card">...</div>
</div>
```

```css
#ai-card-root .ai-card-tag {
  position: absolute;
  top: -1px;                       /* 카드 상단에 거의 붙음 (이전 -10px → 떠 있는 인상 제거) */
  left: 34px;
  background: var(--color-text-info);  /* #0838F8 빌리조 브랜드 블루 */
  color: #fff;
  font-size: 11px;
  padding: 3px 9px;
  border-radius: 5px;
  font-family: 'Pretendard', sans-serif;
  font-weight: 700;
  letter-spacing: 0.2px;
  z-index: 1;
  line-height: 1.3;
}
#ai-card-root .ai-card-tag::before {
  content: "📊 ";                  /* 분석·차트 의미 */
  font-size: 12px;
}
```

**규칙**:
- 텍스트는 정확히 **"빌리조 제품분석"** — 사용자에게 빌리조 자체 브랜드 분석으로 인식되어야 함. "AI", "자동생성", "Generated by AI" 등 내부 용어 노출 금지.
- 이모지는 `::before`로 분리 — 텍스트 변경 시에도 이모지는 유지. 📊 차트 이모지는 데이터·등급 기반 분석 정체성과 일치.
- 위치는 좌측 34px 고정 (카드 모서리에 살짝 안쪽). top: -1px로 카드 외곽에 거의 맞닿게 안착.

---

## 4. 검증된 제품 카드 예시 (참고용)

### 4.1 코웨이 CHP-7211N (F01 정수기)

```yaml
헤더: "코웨이 아이콘 정수기 2.0"
메타: "CHP-7211N · 6개월 약정 · 셀프형/매니저형"
4지표:
  렌탈료: 88
  정수성능: 92
  위생관리: 96
  편의기능: 95
종합점수: 93
6스펙: [월 28,400원, RO 필터, 4개, 5개+, UV+자동, 데스크형]
페르소나 Top 3:
  - IoT 선호 1-2인 가구 (32%, ti-device-mobile)
  - 디자인 중시 신혼 (24%, ti-heart)
  - 위생 민감 가족 (18%, ti-shield-check)
출처: coway.com, cowayrental.shop, prod.danawa.com
```

### 4.2 삼성 AF80F25D29WRS (F03 공조)

```yaml
헤더: "삼성 비스포크 AI 무풍콤보 갤러리 2in1"
메타: "AF80F25D29WRS · 2in1 홈멀티 · 25평+6평 · 1등급"
4지표:
  가격대: 70
  냉방·제습: 96
  공기청정·위생: 95
  AI·편의기능: 98
종합점수: 90
6스펙: [500만원대, 25+6평, 1등급, e-HEPA·PM1.0, UV LED, 2in1 홈멀티]
페르소나 Top 3:
  - 30평대 4인 가족 (38%, ti-home)
  - 알러지·호흡기 민감 가족 (24%, ti-leaf)
  - 스마트홈 30-40대 (18%, ti-device-mobile)
출처: samsung.com/sec/air-conditioners/package-af80f25d29wrt-d2c/AF80F25D29WRT/
```

이 둘은 카드 골격이 동일하지만 평가 항목이 완전히 다르다는 것을 보여주는 검증 케이스. 새 패밀리 룰북을 만들 때 이 패턴을 따른다.

---

## 4.5 빌리조 페이지 보조 변경 — 카테고리 메뉴 (v0.2.3)

본 프로젝트는 AI 카드 외에도 **빌리조 본 사이트의 특정 영역을 시각 개선**하는 보조 CSS를 함께 배포한다. 카드와 별개의 영역이지만 같은 빌리조 페이지 안에서 일관된 톤을 유지하기 위함.

### 4.5.1 대상 영역

`.mobile__gnb .gnb__cateogry .category__wrap` — 빌리조 모바일 카테고리 메뉴 (제품 상세 페이지 등 모바일 환경에서 표시되는 카테고리 네비게이션).

**빌리조 원본 디자인** (`/css/common.css` 2673~2702):
- 가로 스크롤 텍스트 나열 (white-space: nowrap, overflow: auto)
- line-height 45px, font-size 14px, 텍스트만
- `.on` 활성: color #dd5119 (주황) + 아래 주황 밑줄 (`::after`)

**새 디자인 (v0.2.3 적용)**:
- 버튼형 박스 (회색 배경, 1px 테두리, 5-6px radius)
- 폰트 12.5px Pretendard Medium, padding 5-6px 10-13px
- 활성(`.on` 또는 첫 카테고리): 파란 채움 (`#0838F8`) + 흰 글씨 Bold
- 호버: 옅은 파란 (`#e8edff`) + 파란 테두리·글씨
- **최대 2줄 제한** (max-height 64px, ≤480px 60px)
- 2줄 초과 시 **▼ 토글 버튼** (JS 동적 생성, 우측 하단)
- **fade 그라데이션 금지** — ▼ 토글만으로 잘림 표시

### 4.5.2 핵심 규칙

| 항목 | 값 |
|------|-----|
| 셀렉터 specificity | 0,0,4,1 (빌리조 원본과 동일 → cascade 순서로 우선) |
| 패딩 | `7px 10px 8px` (≤480px: `6px 9px 7px`) |
| max-height | 64px (≤480px: 60px) |
| 폰트 | 12.5px Pretendard Medium |
| 활성 컬러 | `#0838F8` (빌리조 브랜드 메인) |
| 원본 `.on:after` 처리 | `display: none` (주황 밑줄 제거) |
| 토글 버튼 위치 | 우측 하단 (bottom: 4px, right: 8px) |
| 토글 동작 | JS 1회 init, `dataset.catToggleInit`로 중복 방지, `resize` 이벤트 재측정 |

### 4.5.3 배포 방식

옵션 1 (권장): 빌리조 페이지 템플릿 `<head>`에 `<style id="billyjo-category-pills">` + `<script id="billyjo-category-toggle">` 블록 직접 추가
옵션 2: 별도 파일 `/css/category-pills.css` + `/js/category-toggle.js`로 분리

빌리조 원본 `/css/common.css` 직접 수정 X — 오버라이드로 처리하여 원본 보존.

---

## 4.7 Hero 재배치 + Step 폰트 +35% (v0.2.7 최종)

### 4.7.1 SLOT 1·2 → .prod_view_top.right Hero 이전

**의도**: 이미지·가격 영역과 AI 카드 평가(게이지)를 한 화면에 묶어 의사결정 가속. 제품명 중복 제거(.prod_view_top에 이미 있음).

**JS 시밍** — DOMContentLoaded + setTimeout [50, 200, 800]:
```javascript
function buildHero(){
  var right = document.querySelector('.prod_view_top .right');
  var aiRoot = document.getElementById('ai-card-root');
  if (!right || !aiRoot || right.dataset.bjHeroBuilt) return;

  // SLOT 1 메타 라인 추출
  var metaSrc = aiRoot.querySelector('.head .meta');
  if (metaSrc && !right.querySelector('.bj-hero-meta')) {
    var metaDiv = document.createElement('div');
    metaDiv.className = 'bj-hero-meta';
    metaDiv.innerHTML = metaSrc.innerHTML;
    right.querySelector('.prod_name').appendChild(metaDiv);
  }

  // SLOT 2 (.summary) 이동
  var summary = aiRoot.querySelector('.card > .summary');
  if (summary && !right.querySelector('.bj-hero-summary')) {
    var wrap = document.createElement('div');
    wrap.className = 'bj-hero-summary';
    wrap.appendChild(summary);
    right.appendChild(wrap);
  }
  right.dataset.bjHeroBuilt = '1';
}
```

**필수 CSS** (요약):
```css
/* 레이아웃 안전화 — .prod_view_top float → flex */
#container .wide-inner > .prod_view_top{
  display:flex !important; flex-wrap:wrap !important;
  gap:24px !important; align-items:flex-start !important;
}
.prod_view_top > .left, .prod_view_top > .right{
  flex:1 1 360px !important; min-width:280px !important;
  float:none !important; width:auto !important;
}
@media (max-width:1023px){
  .prod_view_top{ flex-direction:column !important }
  .prod_view_top > .left, .prod_view_top > .right{
    flex:1 1 auto !important; max-width:100% !important; width:100% !important;
  }
}

/* CSS 변수 스코프 복원 — SVG 게이지는 #ai-card-root 안 변수를 사용하므로 hero 위치에도 재정의 필수 */
.prod_view_top .right{
  --color-text-info:#0838F8; --color-text-secondary:#6a6a6a;
  --color-border-tertiary:#dfdfdf;
  --g-1:#0838F8; --g-1-5:#1a87ac; --g-2:#16a34a; --g-2-5:#4ec727;
  --g-3:#84cc16; --g-3-5:#c2ce15; --g-4:#facc15; --g-d:#999;
}

/* 시각 융합 — .prod_view_top 하단 + AI 카드 상단 단일 컨테이너처럼 */
#container .wide-inner > .prod_view_top{
  border-bottom:0 !important; border-bottom-left-radius:0 !important;
  border-bottom-right-radius:0 !important; padding-bottom:22px !important;
}
#ai-card-wrapper{ margin-top:0 !important }
#ai-card-root .card{
  border-top:0 !important; border-top-left-radius:0 !important;
  border-top-right-radius:0 !important; margin-top:0 !important;
  padding-top:18px !important;
}
#ai-card-root .card > .specs:first-child{
  border-top:1px solid #f0f0f0; padding-top:14px;
}

/* AI 카드 본체에서 이동 원본·중복 숨김 */
#ai-card-root .head{ display:none !important }
#ai-card-root > .card > .summary{ display:none !important }
#ai-card-root .ai-card-tag{ display:none !important }
```

**중요 함정**:
1. **CSS 변수 미정의 → 게이지 안 보임**: hero 위치(`.prod_view_top .right`)에 `--g-2` 등 변수 재정의 누락 시 SVG stroke·text fill이 검정 또는 사라짐. 텍스트 "A"만 보이는 증상.
2. **float clearfix 누락 → 카드 겹침/잘림**: `.right`가 hero summary로 길어지면 float 컨테이너가 enclose 못해 다음 형제 요소(AI 카드)와 겹침. flex 변환 필수.
3. **`.ai-card-tag` (📊 빌리조 제품분석 배지)**: `position:absolute; top:-1px`로 카드 상단 위로 떠 있어 `.prod_view_top` 영역 침범 → "잘린 카드" 시각. hero에 이미 제품 정체성이 있으므로 `display:none` 처리.

### 4.7.2 SLOT 6 Step 폰트 +35%

기본(`.step-n 18px/10.5px`, `.step-title 12.5px`, `.step-sum 11.5px`) → 본문 폰트 14px 수준으로 확대. 시각 위계 명확화. 정확한 CSS는 CLAUDE.md #23 참조.

### 4.7.3 `.prod_view_bot.card.mt40` 렌탈사 비교 유지 + sticky 위젯 (v0.3.0)

inject.js가 내부에 `.bb-inner` 위젯(공급자 탭·개월 pill·장바구니/렌탈신청 버튼)을 동적 렌더. 실제 운영 페이지에서는 **`display:none` 처리 금지** — 위치(병합 영역 다음) 그대로 노출 필수.

**v0.3.2 fixed 위젯 정식화** (v0.3.0 sticky → v0.3.1 fixed 정정 → v0.3.2 안전망 강화):

- `body #container .wide-inner > .prod_view_bot.card.mt40, .prod_view_bot.card.mt40 { position:fixed; bottom:0; left:0; right:0; z-index:99999; border-radius:14px 14px 0 0; box-shadow:0 -8px 24px rgba(0,0,0,0.08) }` — viewport 하단에 진짜 고정. 더블 selector로 specificity 동급화 (v0.2.4 룰 (0,1,4,0) override)
- 와이드 PC(≥1500px)에서 `left:50%; transform:translateX(-50%); max-width:1500px`로 중앙 정렬
- `▾/▴` 폴딩 토글 + 핸들 탭으로 펼침/접힘 (max-height transition 0.32s, 펼침 520px + 내부 overflow-y:auto / 접힘 64px / 모바일 58px)
- 핸들 마크업: grip indicator + 제품명 + 대표 가격 + 우측 토글 버튼
- 버튼 3종: `[장바구니]` `[🎁 렌탈+사은품 신청 — 브랜드 파랑 채움]` `[💬 상담신청 — 아웃라인 보조]`
- "렌탈 신청" 단독 명칭·전화기 아이콘 폐기. 사은품 강조로 가치 인식 향상
- body에 `padding-bottom:88px` (모바일 80px) — fixed 위젯이 마지막 콘텐츠 가리지 않게
- 접근성: role="button", tabindex=0, Enter/Space 키보드, aria-label

**JS inline fallback (v0.3.2)** — CSS specificity 충돌 안전망:
```javascript
function forceFixedStyle(wrapper){
  wrapper.style.cssText += ';position:fixed!important;bottom:0!important;' +
    'left:0!important;right:0!important;z-index:99999!important;margin:0!important;';
}
function preEnhance(){
  var wrapper = document.querySelector('.prod_view_bot.card.mt40');
  if (wrapper) forceFixedStyle(wrapper);
}
preEnhance();  // DOMContentLoaded 전에도 즉시 실행
```

JS 시밍은 inject.js 결과 DOM 후처리 — DOMContentLoaded + setTimeout [100, 400, 1200, 3000] + MutationObserver(6초 후 disconnect). `.bb-inner` 부재 시에도 핸들은 fallback 텍스트(`.prod_name b` + `.top_min_price b`)로 표시. 자세한 사양은 CLAUDE.md 절대 규칙 #25 참조.

**모바일 헤더 (v0.3.2)** — `.new-gnb__wrap`·`.new-gnb`·`.bj-internet-li` ≤768px 명시 숨김. `.header__top { gap:0 }` + 햄버거 `margin-right:5px` + 우측 아이콘 그룹 `margin-left:auto`로 [햄버거-5px-로고-자유공간-아이콘] 정렬 명시. 자세한 사양은 CLAUDE.md 절대 규칙 #21 참조.

### 4.7.4 검증 페이지

- 외부 공개: https://billyjo-header-layouts.vercel.app/final.html
- 비교 페이지: index에서 canonical / merge_a / merge_b / merge_c / step_1·2·3 / final 비교 가능
- 권장 검증 폭: 1400 → 1280 → 1024 → 768 → 480px

---

## 4.6 빌리조 페이지 보조 변경 — 좁은화면 헤더 패치 (v0.2.5)

### 4.6.1 문제 (원인 진단)

빌리조 본 사이트는 `cdn.jsdelivr.net/gh/billyjo-appsilon/billyjo-inject@.../inject.js`를 모든 페이지에서 로드해 헤더 DOM을 런타임에 재구성한다:

- **PC > 768px**: `header.new-header` 첫 자식으로 새 row를 만들고 `leftGroup`(로고+`.new-gnb__wrap`) / `rightGroup`(`.gnb__right`+`.search__wrap`)으로 2-그룹 flex 배치. `rightGroup`에 인라인으로 `display:flex; align-items:flex-end; gap:24px; flex-shrink:0; white-space:nowrap; margin-left:auto`를 박는다.
- **≤ 768px**: 이벤트 링크(`<li>이벤트/기획전 바로가기!!</li>`)를 떼서 `#bj-top-banner`로 분리, 검색·장바구니 아이콘만 남은 `<ul>`을 `#bj-header-icons`로 ID 부여 후 `.header__top`에 appendChild.

세 가지 부작용:

1. **PC 좁은 폭 (≈1024-1280px)**: rightGroup이 `flex-shrink:0`이라 줄어들지 않고 `white-space:nowrap`으로 줄바꿈도 막힘. leftGroup이 `flex:1; min-width:0`이라 강제 압축되어 카테고리(`.new-gnb`)가 컨테이너를 벗어나 paint되며 rightGroup 위로 흘러넘침 (=올라타는 현상).
2. **모바일 헤더 가림**: `<style>@media(max-width:768px){header:not(.bj-ready){opacity:0!important}}</style>`가 `.bj-ready` 클래스 부여 시점에 의존. inject.js 지연·실패 시 헤더 자체가 안 보임.
3. **햄버거 중복**: `.gnb__hamburger`(JS와 연동된 사이드 메뉴 트리거)와 `.hamburger__btn`(`.new-gnb__wrap` 내부)이 좁은 PC에서 동시 노출.

### 4.6.2 처리 (v0.2.5 절대 규칙 #21)

**(a) JS 시밍** — inject.js 결과 DOM에 안정 클래스(`.bj-inj-row` / `.bj-inj-left` / `.bj-inj-right`) 부여 + `.bj-ready` 강제 + 이벤트 링크 복원:

```javascript
function tagInject(){
  var header = document.querySelector('header.new-header');
  if (!header) return;
  if (!header.classList.contains('bj-ready')) header.classList.add('bj-ready');
  var firstRow = header.firstElementChild;
  if (firstRow && firstRow.tagName === 'DIV' && !firstRow.classList.contains('bj-inj-row')) {
    var inlineCss = (firstRow.getAttribute('style') || '').replace(/\s+/g,'');
    if (inlineCss.indexOf('display:flex') >= 0 && inlineCss.indexOf('padding:28px') >= 0) {
      firstRow.classList.add('bj-inj-row');
      Array.prototype.forEach.call(firstRow.children, function(c){
        var s = (c.getAttribute('style') || '').replace(/\s+/g,'');
        c.classList.add(s.indexOf('margin-left:auto') >= 0 ? 'bj-inj-right' : 'bj-inj-left');
      });
    }
  }
  restoreMobileIcons();  // #bj-top-banner의 이벤트 링크를 #bj-header-icons 첫 <li>로 복원 후 banner 제거
}
// 실행 트리거: DOMContentLoaded + setTimeout [50, 200, 600, 1500, 3000] + MutationObserver (5초 후 disconnect)
```

**(b) CSS 3-stage breakpoint**:

| Breakpoint | 동작 |
|---|---|
| `≤1280px` | `.bj-inj-row{flex-wrap:wrap}` + `.bj-inj-right{flex-shrink:1; white-space:normal; flex-wrap:wrap}` + `.gnb__right{position:static; top:0}` — rightGroup이 줄어들거나 다음 줄로 떨어지게 |
| `≤1024px` | `.bj-inj-row{flex-direction:column}` — rightGroup이 leftGroup 아래로 별도 행 분리 |
| `≤768px (모바일)` | `.header__top`을 `display:flex` 1행 정렬: `[햄버거 · 로고 · 이벤트·검색·장바구니]`. 검색폼(`.top__right`) 숨김 — 검색 아이콘 클릭으로 `.m_search_popup` 사용. `#bj-top-banner{display:none}` |

**(c) 모바일 가림 방지** (specificity 동률 → 동일 클래스 중첩):

```css
header.new-header.new-header{ opacity:1 !important; visibility:visible !important }
@media (max-width:768px){
  header.new-header.new-header:not(.bj-ready){ opacity:1 !important; visibility:visible !important }
}
```

**(d) 공통**: `.hamburger__btn` 숨김 (좌상단 햄버거는 `.gnb__hamburger`만 유지 — `.mobile__aside` 사이드 메뉴 JS와 연동). 제품 다중 이미지 썸네일(`.prod_thumbs li a`)에 `border:1px solid #dfdfdf` 추가.

### 4.6.3 배포·검증

- 검증 페이지: `billyjo_10914_actual_v023.html` (canonical) — 외부 공개 URL https://billyjo-header-layouts.vercel.app
- 3개 레이아웃 비교본(`_layout1/2/3.html`)을 거쳐 **Layout 1 (rightGroup 자연 줄바꿈)** 채택. 2/3은 아카이브.
- 권장 검증 폭: `1400px → 1280px → 1024px → 768px → 480px`
- 본 사이트 직접 적용 시: `<style>` 블록 + 인라인 `<script>`를 빌리조 페이지 템플릿 `<head>` 끝(inject.js 스크립트 태그 직전 또는 직후) 추가. 원본 CSS/JS 직접 수정 X.

---

## 5. 권장 프로젝트 지시사항 (System Prompt)

프로젝트 설정 → Instructions 에 아래 텍스트 그대로 넣으면 모든 새 채팅이 일관된 컨텍스트로 시작됨.

```
이 프로젝트는 빌리조(billyjo.co.kr)의 렌탈 가전 제품 상세페이지를 AI로 자동 생성하는 
시스템의 설계·구현 작업 공간입니다.

핵심 기준:
- 평가 기준은 지식 베이스의 `billyjo_ai_detailpage_rulebook.md` (v0.1)를 따릅니다.
- 시스템 아키텍처와 카드 디자인 사양은 `billyjo_ai_detailpage_context_memo.md` 를 따릅니다.
- 14개 카테고리 패밀리(F01~F14) 중 어디에 속하는지 먼저 분류한 뒤 룰북 적용.
- 모든 제품 정보는 제조사 공식 페이지 + 다나와에서만 가져오며, 출처 URL과 원문 
  인용을 함께 기록합니다. 출처 없는 내용 생성 금지.
- 카드는 7개 슬롯 구조 (헤더 / 게이지+4지표 / 6-8 스펙요약 / 기본정보 / 페르소나 Top 3 / 
  상세 스펙 Step1-3 / 예상 지원금·사은품)를 지킵니다. SLOT 3은 기본 6 기술 스펙 + 선택 
  2 운영/계약 스펙 (방문관리·소유권 등). 중복 정보는 .specs로 단일화하고 산문 영역에서 제거.
- 종합점수 = (지표1+2+3+4)÷4, 100점 만점.

운영 우선순위: F01(정수)·F03(공조)·F05(냉장)·F11(가구) 4개 패밀리 먼저 운영화. 
자동차렌탈(F12)·업소용(F14)은 별도 스키마 검토 필요.

검증 완료 케이스: 코웨이 CHP-7211N, 삼성 AF80F25D29WRS. 새 작업은 이 둘의 패턴을 
기준점으로 삼습니다.
```

---

## 6. 통합 후 권장 첫 작업

새 프로젝트에서 시작하기 좋은 첫 채팅 주제 (우선순위 순):

1. **F05 냉장가전 룰북 상세화** — 빌리조 인기 카테고리, LG/삼성 비스포크 등 검증할 제품 풍부.
2. **F11 가구·침구 룰북 상세화** — 매트리스 케어형은 빌리조 특화 상품, 페르소나 정의가 가전과 달라 별도 정의 필요.
3. **카테고리 라우터 구현** — 빌리조 어드민 등록 시점에 F01~F14 자동 매칭하는 로직.
4. **출처별 크롤러 모듈 설계** — 코웨이/쿠쿠/세스코/삼성/LG 5사 우선.
5. **검토 큐 백오피스 UI 와이어프레임** — 신뢰도 미달 항목 처리.
6. **GTM 태그 + API 캐시 조회 구현** — 사용자 노출 단계 마무리.

---

**문서 끝**
