# Archify 전수조사 분석 정리 (한국어)

> 이 문서는 Archify 저장소를 전수조사하고, 실제로 CLI를 실행해 검증한 결과를
> 정리한 분석 자료입니다. 설치·사용법, 정체(스킬/플러그인/MCP), 토큰 필요 여부,
> AI 에이전트 구축 활용, 수익화 전략, React/PHP 구현 가능성, 교육 콘텐츠
> 제작 가능성을 다룹니다.

---

## 📍 저장소 정보

| 항목 | 값 |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/archify |
| **원본 저장소 (upstream)** | https://github.com/tt-a1i/archify |
| **프로젝트 페이지** | https://tt-a1i.github.io/archify/ |
| **시나리오 가이드** | https://tt-a1i.github.io/archify/guide.html |
| **Proof Lab (검증 산출물 갤러리)** | https://tt-a1i.github.io/archify/gallery.html |
| **설치 도우미 페이지** | https://tt-a1i.github.io/archify/start.html |
| **기반 프로젝트** | https://github.com/Cocoon-AI/architecture-diagram-generator (MIT v1.0) |
| **실제 저장소 매핑 사례** | https://github.com/mco-org/mco (커밋 `9f1a1cf` 추적) |
| **Trendshift** | https://trendshift.io/repositories/31352 |
| **분석 기준 버전** | `v2.17.0-dev.1` (개발 버전, 정식 릴리스 아님) |
| **분석 일자** | 2026-10-07 |

---

## 1. 한 줄 요약

> **"AI 에이전트가 말로 받은 시스템 설명을 → 검증된 타입 JSON으로 바꾸고 →
> 혼자 돌아가는 인터랙티브 HTML 도면으로 결정론적으로 컴파일해주는
> Node.js 렌더링·검증 엔진 + 에이전트 스킬"**

핵심 구조: **AI가 그림을 그리는 게 아니라, AI가 데이터를 쓰고 Archify가 그림을 그린다.**

```
사용자 말  →  AI가 타입 JSON IR 작성  →  Archify 9단계 검증  →  자기완결형 HTML
            (머리: 무엇을 그릴지)      (손: 정확하게 그린다)
```

---

## 2. 기본 정보 (실측)

| 항목 | 값 |
|---|---|
| 라이선스 | MIT (상업적 이용·수정·재배포 자유) |
| 언어 | 순수 Node.js ESM (`.mjs`), Node ≥ 18 |
| **런타임 의존성** | **0개** (`node_modules` 없음을 직접 확인) |
| devDependencies | `ajv`, `parse5`, `saxes`, `simple-icons` (테스트·생성 전용) |
| 코드 규모 | `.mjs` 약 63,400줄 + 뷰어 JS 8,700줄 |
| 테스트 | 테스트 파일 **114개** (CI: Ubuntu / macOS / Windows) |
| 배포 패키지 | `archify.zip` = 1.88MB |
| 전체 저장소 | 79MB (대부분 docs 이미지) |
| 스폰서 | Supercode (supercode.sh), EverMind / Raven |

**의존성 0개**가 중요: 설치 시 `npm install`이 필요 없고, 에어갭 환경에서도 작동.

---

## 3. 실행 검증 결과 (직접 실행)

### doctor
```
$ node bin/archify.mjs doctor
Archify doctor

[ok] Node.js v22.22.0 (requires >=18)
[ok] Core template
[ok] Example renderer
[ok] Live preview runtime
[ok] Visual-check runtime
[ok] Output path safety runtime
[ok] Scenario recipe guide
[ok] Progressive authoring references
[ok] Architecture compare runtime and proof fixtures
[ok] Standalone schema validators
[ok] architecture renderer, schema, and example
[ok] workflow renderer, schema, and example
[ok] sequence renderer, schema, and example
[ok] dataflow renderer, schema, and example
[ok] lifecycle renderer, schema, and example

Archify is ready.
```
15개 항목 전부 `[ok]`, 종료 코드 0.

### deliver (실제 렌더링)
```
$ node bin/archify.mjs deliver architecture examples/web-app.architecture.json \
    out.html --quality showcase --json
{
  "ok": true,
  "specification": { "sha256": "483350f5...", "bytes": 3793 },
  "artifact":      { "sha256": "426db7bd...", "bytes": 811809 },
  "validation": { "checksPassed": 9, "checkCount": 9,
                  "compositionProfile": "showcase",
                  "errors": 0, "warnings": 0 }
}
```
**JSON 3.8KB → HTML 812KB.** 그 안에 SVG 도면 + 뷰어 JS + JetBrains Mono 폰트(약 96KB)
전부 포함. 서버·인터넷 없이 열림.

### 네트워크 호출 검증
```
$ grep -rn "fetch(\|https\.get\|http\.request\|ANTHROPIC\|OPENAI\|API_KEY\|apiKey" \
    archify/renderers archify/bin archify/scripts archify/delta
--- (결과 없음) ---
```
**렌더러·CLI·스크립트·델타 엔진 전체에 네트워크 호출 0건, API 키 코드 0건.**

유일한 통신: 업데이트 알림 1개. URL이 하드 고정되어 있음:
```js
// archify/scripts/update-contract.mjs:3
export const DEFAULT_MANIFEST_URL =
  'https://tt-a1i.github.io/archify/skill-updates/archify/stable.json';

// archify/scripts/check-update.mjs:1348
if (manifestUrl !== DEFAULT_MANIFEST_URL)
  throw new UpdateContractError('unexpected manifest URL');
```

### 로케일 지원 검증
```
$ grep -o "'[a-z][a-z]\(-[A-Z][A-Z]\)\?'" archify/renderers/shared/i18n.mjs | sort -u
'en'
'zh-CN'
```
**한국어 뷰어 UI 미지원.** 영어/중국어만. → 기여 기회 (아래 7장 참조)

---

## 4. 5가지 다이어그램 타입

각 타입마다 전용 JSON 스키마 + 전용 렌더러가 따로 있음 (범용 자동 레이아웃 엔진이 아님).

| 타입 | 쓰는 곳 |
|---|---|
| `architecture` | 컴포넌트·서비스·클라우드/보안 경계·인프라 |
| `workflow` | 프로세스·승인 게이트·툴 호출·런북·CI/CD (schema v2) |
| `sequence` | API 호출 체인·요청 생명주기·비동기 추적·리턴 |
| `dataflow` | 파이프라인·ETL/ELT·데이터 계보·거버넌스·PII |
| `lifecycle` | 상태 전이·재시도·대기·종료 상태 |

어떤 걸 쓸지 모를 때:
```bash
node bin/archify.mjs guide "Show an API request with Redis cache miss" --json
```

### 컴포넌트 타입 7종 (색이 자동 결정됨)
```
frontend   backend   database   cloud   security   messagebus   external
```
AI가 색 코드를 직접 쓰지 않음. 역할만 적으면 디자인 시스템이 색을 정함.

### variant 4종
```
default   emphasis(강조)   security(보안)   dashed(점선=비동기/부수적)
```

---

## 5. 핵심 차별점 — "검증 가능한 도면"

### 5-1. 9단계 원자적 검증

`showcase` 프로필에서 **9개 검사 전부 + 오류 0 + 경고 0**이어야 산출물 교체.

| 검사 | 내용 |
|---|---|
| 스키마 | 필드가 규칙에 맞나 |
| 레이아웃 | 상자 겹침, 여백 충분 |
| HTML/SVG | 생성된 코드 유효성 |
| 경로(route) | 선이 불투명 노드를 뚫나 |
| **라벨 간격 (Clean Label Gate)** | 라벨 마스크가 다른 경로와 4px 이상 (`showcase`), 2px 미만은 경고 (`standard`) |
| **모호한 공유 복도 게이트** | 무관한 두 직교 관계가 8px 이상 같은 레인 공유 → 거부 (가짜 합류/분기로 보이기 때문) |
| 도달성 | 고아 노드 |
| 유한성 | NaN/Infinity 좌표 |
| 자동 포트 분산 | 화살표가 한 점에 뭉침 방지 |

> 주의: 영수증에 검사가 **4개만** 나오면 기본 검증이며 **showcase 합격이 아님.**
> (`SKILL.md` 명시)

### 5-2. 구조화된 수리 영수증 (Structured Repair Receipt)

가장 베낄 가치 있는 패턴. 실패 시 Node 스택트레이스가 아니라 이 JSON을 반환:

```json
{
  "schemaVersion": 1,
  "ok": false,
  "stage": "composition",
  "diagnostics": [{
    "code": "layout/label-clearance",          // 안정적 규칙 코드
    "severity": "error",
    "subject": { "relationship": "api-sql" },   // 정확히 어디
    "evidence": { "measured": 2.4, "threshold": 4, "unit": "px",
                  "collidesWith": "cache-read-through" },  // 실측값
    "supportedFixes": ["labelDy", "labelAt", "via"]        // 허용된 수단만
  }]
}
```

4요소가 핵심:

| 요소 | 역할 | 없으면 |
|---|---|---|
| `code` | 안정적 규칙 ID | AI가 같은 오류를 매번 다르게 인식 |
| `subject` | 정확한 대상 | AI가 어딜 고쳐야 할지 추측 |
| `evidence` | 실측 수치 | AI가 얼마나 고쳐야 할지 모름 |
| `supportedFixes` | **허용된 수단 화이트리스트** | AI가 창의적으로 망침 |

렌더러 자식 프로세스는 private 구조화 경계를 emit → CLI가 Node 스택을 기계 출력에
절대 복사하지 않음. 알 수 없는 내부 실패는 "명시적으로 미분류"로 남고
**발명된 수정을 제시하지 않음.**

### 5-3. 원자적 납품 (Atomic Verified Delivery)

```
1. 같은 폴더에 임시 후보 생성 (out.html.candidate-xxxx)
2. 후보를 렌더링
3. 9개 검사 실행
4. SHA-256 + 바이트 수 영수증 생성
5. 통과 → rename 한 번으로 교체 (유일한 커밋 지점)
   실패 → 후보 삭제, 기존 산출물은 바이트 단위로 보존
```

DB 트랜잭션과 같은 원리. 발표 직전 수정이 깨져서 기존 파일까지 날아가는 사고를
구조적으로 방지.

### 5-4. 정직성(Truthfulness)이 코드로 강제됨

**주장 3분할:**

| 명령 | 증명하는 것 | 증명 못 하는 것 |
|---|---|---|
| `deliver` | 결정론적 산출물 검사 통과 | 브라우저에서 실제로 잘 보이는지 |
| `visual-check` | 실제 브라우저의 제한된 동작 | **예쁜지** |
| 사람/이미지 모델 | 지각적 품질 | — |

`SKILL.md` 원문:
> "0이 아닌 종료 코드를 성공으로 설명할 수 없다. 수행하지 않은 시각 검사를
> 주장하지 말라."
> "실패한 납품 경로에 `visual-check`를 실행하지 말라: 실패한 후보가 아니라
> 낡은 last-good 산출물을 검사하게 된다."

**편법(위조) 금지 목록 — 하나하나 명시적으로:**
- "관계 라벨은 의미 데이터다. 모든 유의미한 라벨을 보존하라. **삭제는 기하 수정이 아니다.**"
- "`overflow: hidden`, 잘린 콘텐츠, 내부 다이어그램 스크롤러, 늘린 SVG 높이,
  더 작은 타이포그래피로 **합격을 위조하지 말라.**"
- "검증을 통과하려고 엔지니어링 프로필을 제거해서는 안 된다. 사실을 수리하거나
  진단을 정직하게 보고하라."

**뷰어 인터랙션도 정직:** 포커스, 상/하류 도달성, 경로, 역할 비교, 스토리는
작성된 노드·관계만 재사용. 토폴로지를 발명하지 않고 런타임 영향도를 주장하지 않음.

### 5-5. 소스 증거 (Repository Evidence) — 요청 시에만

아키텍처 노드에 `SRC n` 뱃지 → **하나의 공개 커밋에 고정된(pinned)** Git 검증 파일의
정확한 라인 범위를 엶. 실제 사례: `mco-org/mco` 커밋 `9f1a1cf` 추적 지도가
`docs/cases/mco-runtime.*`에 커밋되어 있음.

평범한 도면은 소스 연결 없이 깨끗하게 나옴.

### 5-6. 아키텍처 델타 (PR 리뷰용)

```bash
node archify/bin/archify.mjs compare architecture base.json head.json delta.html --json
```

검증된 두 스냅샷을 **Before / Delta / After**로 비교 + 기계 판독 영수증:
```
추가(added) / 삭제(removed) / 변경(changed) / 이동(moved) / 재라우팅(rerouted)
```

**의도적으로 안 하는 것:** 영향도(impact), 리스크(risk), 머지 안전성(merge safety)을
추론하지 않음. → 책임 경계가 명확. 판단은 사람이 함.

Review Navigator: 변경 행 클릭으로 이동, 또는 변경당 1400ms 자동 재생.
대상이 없거나 중복이면 내비게이터를 비활성화하면서도 정적 증거는 유지 (fail-closed).

---

## 6. 결과물 HTML — 812KB의 정체

```
out.html (812KB)
├── SVG 도면          실제 그림 (벡터, 무한 확대)
├── 뷰어 JS           검색·포커스·경로추적·내보내기 (8,700줄)
├── CSS (4 프리셋)     classic / signal-flow / blueprint / editorial
├── JetBrains Mono    폰트 내장 (약 96KB) — 오프라인에서도 글씨 동일
└── 접근성 레이어      키보드 조작, 포커스 표시, prefers-reduced-motion
```

### 키보드 조작

| 기능 | 키 |
|---|---|
| 사실 기반 다이어그램 가이드 | `?` |
| 노드 검색·포커스 | `/` |
| 상류/하류 작성된 도달성 추적 | 노드 포커스 → `Upstream`/`Downstream` |
| 방향성 경로 탐색 + 여정 검사 | `R` 또는 `PATH` |
| 1~2개 의미 역할 비교 | `L` 또는 `LENS` |
| 실시간 전체 레이더 | `M` 또는 `MAP` |
| 가이드 스토리 재생 / 챕터 이동 | `P` / `[` `]` |
| 프레젠테이션 스테이지 | `F` |
| 비주얼 스타일 / 테마 / 내보내기 | `S` / `T` / `E` |
| 줌 / 리셋 | `+` `-` `0` |

### 딥링크 (실용성 최고)

```
out.html#focus=<id>
out.html#focus=<id>&reach=upstream|downstream
out.html#relation=<id>
out.html#route=<source>~<target>
out.html#lens=<kind>~<kind>
out.html#view=<view-id>
```

Before: "그 다이어그램에서 API 서버 밑에 있는 큐 보이시죠? 거기서..."
After: `out.html#focus=queue&reach=downstream` ← 이거 눌러보세요

### 내보내기

```
PNG (클립보드 복사 가능) / JPEG / WebP / SVG / WebM(모션)
공유 카드 1200×630        README·릴리스노트·소셜 규격
경로 공유 카드            추적한 경로 강조 + 전체 도면은 흐리게 유지
도달성 공유 카드          상/하류 결과 캡처 ("Authored upstream/downstream" 명기)
```

규칙: 임시 뷰어 상태(줌·포커스·애니메이션)는 절대 들어가지 않고, 항상 전체 도면 유지.
모션은 유한(finite)하고 `prefers-reduced-motion`을 존중하며 표준 내보내기에 들어가지 않음.

---

## 7. 폴더 구조

```
archify/                        ⭐ 실제 배포되는 스킬 본체 (= archify.zip 내용)
├── SKILL.md                    핵심! AI에게 주는 137줄 작성 지침서
├── bin/archify.mjs             CLI 엔트리 (14개 명령어)
├── schemas/*.schema.json       5개 타입 스키마 + common
├── renderers/
│   ├── architecture/ workflow/ sequence/ dataflow/ lifecycle/
│   └── shared/                 geometry, validator, i18n, text-fit,
│                               brand-marks, repository-evidence 등 19개 모듈
├── delta/architecture-delta.mjs
├── examples/*.json             타입별 참조 예시 (형태 참고용, 사실 복사 금지)
├── references/                 4개 심화 계약서
│   ├── authoring-contract.md   필드 enum, 간격 수식, 기하 수리 규칙
│   ├── delivery-contract.md    영수증 필드, 커버리지, 사이드카
│   ├── viewer-runtime.md       공유카드, 모션, 스토리, 딥링크
│   └── brand-marks.md          브랜드 캡처
├── migrations/workflow-v2.mjs
├── brand-marks/catalog.json
└── test/ (114개)

viewer/                         결과 HTML에 주입되는 뷰어 런타임 (8,700줄)
├── semantic-lens.js  focus.js  route-probe.js  node-finder.js
├── guided-views.js   intent-trace.js  motion-governor.js
├── export.js  export-cleanup.js  semantic-radar.js  reader-layout.js
├── viewer-camera.js  viewer-chrome-layout.js
└── template.source.html        단일 진실 소스 → 빌드 시 주입

docs/                           GitHub Pages 사이트
├── index.html gallery.html guide.html start.html
├── gallery/artifacts/*.html    ⭐ 11개 검증된 실제 산출물 (Proof Lab)
├── gallery/sources/*.json      그 JSON 소스
├── cases/mco-runtime.*         실제 공개 저장소 매핑 증거
├── authoring-cookbook.md       에이전트 쿠킅북 (+ zh-CN)
└── research-visual-evolution-round-{2..49}.md  ⭐ 48라운드 디자인 연구 기록

benchmarks/ordinary-model-floor/  "평범한 모델도 쓸 만한가" 품질 하한선 벤치마크
├── benchmark.mjs (561줄)  manifest.json
├── cases/ prompts/ (5개 타입별)
└── results/2026-07-26-pi-three-models*.json  (3개 모델 결과 3세트)

experiments/
├── v3-mermaid-validation/      ⭐ Mermaid 기본(A) / 테마(B) / Archify(C) 비교 실험
│   ├── RESULT.md INDEX.md
│   ├── output-A-stock/ output-B-themed/ output-C-archify/ (각 PNG)
│   └── sources/*.mmd  theme/archify-mermaid-config.json
└── visual-evolution/           DECISION-MAP.md + 프로토타입

integrations/deepseek-harness/   DeepSeek DSH 플러그인 연동
.agents/skills/archify-review/   ⭐ 저장소 자체 리뷰용 메타 스킬
scripts/                         빌드·갤러리·가이드·결정론적 ZIP 생성
```

### 먼저 열어볼 3곳

1. **`docs/gallery.html` (Proof Lab)** — 11개 검증 산출물 + JSON 소스 + 영수증
2. **`experiments/v3-mermaid-validation/RESULT.md`** — Mermaid 대비 비교 결론
3. **`archify/SKILL.md`** — 에이전트 스킬 설계 교과서 (137줄)

---

## 8. `SKILL.md` — AI 에이전트 설계 교과서

### frontmatter (Agent Skill 규격)
```yaml
---
name: archify
description: Create polished, validated architecture, workflow, sequence,
  data-flow, and lifecycle/state diagrams as explorable standalone HTML...
license: MIT
metadata:
  version: "2.17"
  author: tt-a1i
  based_on: Cocoon-AI/architecture-diagram-generator (MIT, v1.0)
---
```

### 영리한 장치 7가지

**1) Artifact First (분석 마비 방지)**
> "다음 툴 동작은 반드시 후보 파일을 작성해야 한다. 렌더러 내부를 들여다보기 전에
> 후보를 작성하라. **산문으로 정확한 좌표를 계획하지 말라.**"

**2) 읽기 금지 목록 (컨텍스트 절약)**
> "첫 후보 전에 `renderers/shared/geometry.mjs`, 렌더러 소스, 검증기 소스, 테스트,
> 벤치마크를 읽지 말라. **두 번의 집중 수정이 실패한 후에만** 구현을 들여다보라."

보통 프롬프트는 "이것들을 읽어라"만 쓴다. Archify는 **"이것들은 읽지 마라"** 를 명시.

**3) 점진적 공개 3단 구조**
```
항상 읽음 (137줄) : SKILL.md
조건부 (각 1개)   : schemas/<선택>.schema.json, examples/<선택>.json
막힐 때만         : references/authoring-contract.md
사용자가 물을 때만 : references/viewer-runtime.md, delivery-contract.md, brand-marks.md
명시적 금지        : geometry.mjs, 렌더러·검증기 소스, 테스트, 벤치마크 (첫 후보 전)
```

**4) 최소값 규칙 + 2라운드 상한 (무한 루프 방지)**
> "객관적 오류 수가 **새로운 최소값에 도달하는 동안** 집중 수정을 계속하라.
> **두 라운드 연속 최선값이 개선되지 않으면 멈추고** 미해결 진단을 정직하게 보고하라."

```
best = ∞;  stale = 0
while true:
    errors = validate()
    if errors == 0: break
    if errors < best: best = errors; stale = 0
    else: stale += 1
    if stale >= 2: report_honestly(); break
    fix_only(diagnostics[0].subject, diagnostics[0].supportedFixes)
```

**5) 1회 1수정**
> "진단이 요구하기 전에는 `via`, `channelX`, `channelY`, `labelAt`을 추가하지 말라.
> **수정당 진단된 기하 제어는 최대 1개.**"

**6) 명시적 상한**
> "하나의 명확한 주 경로, 짧은 side branch, 희소한 라벨, **주 노드 최대 12개**로 시작하라."
> `meta.views`는 **최대 5개** 챕터.

**7) 기본값 생략 선호**
> `meta.visual_preset` 생략 → classic / `meta.subtitle` 생략 / `meta.legend` 생략 → auto
> `meta.engineering_profile` 생략 / `meta.column_fit` 생략 → fixed

### 데스크톱 뷰포트 검증 요구
> "핸드오프 전 1440×900, 1600×1000, 1920×1080에서 실제 HTML을 열어라. 큰 디스플레이용
> 구성이면 2048×1320도 추가 확인. 모든 크기에서
> `document.documentElement.scrollWidth <= window.innerWidth` 및
> `scrollHeight <= window.innerHeight`를 요구하라."

---

## 9. 7가지 질문 답변

### 9-1. 설치 및 사용법

**가장 빠른 길:**
```bash
npx skills add tt-a1i/archify -g
```

**비대화형(CI/자동화):**
```bash
npx -y skills add tt-a1i/archify --skill archify --agent cursor --global --copy --yes
```

**설치 없이 1회 시험:**
```bash
npx skills use tt-a1i/archify@archify --agent codex
```

**에이전트별 설치 위치:**

| 에이전트 | 전역 | 프로젝트 로컬 |
|---|---|---|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Codex CLI | `~/.agents/skills/` | `.agents/skills/` |
| opencode | `~/.config/opencode/skills/` | `.opencode/skills/`, `.agents/skills/` |
| Cursor | (switcher 생성) | `.agents/skills/archify` |
| Raven | `~/.raven/workspace/skills/` (ZIP 수동) | — |
| Claude.ai 웹 | Settings → Capabilities → Skills 에 `archify.zip` 업로드 | (샌드박스 Node 접근에 의존) |
| Project Knowledge | `archify.zip` 업로드 | 프롬프트 주도 폴백만 |
| DeepSeek Harness | `dsh plugin --profile web add @tt-a1i/archify-dsh@0.1.0` | 커뮤니티, 비공식 |

**설치 확인:**
```bash
node bin/archify.mjs doctor
node bin/archify.mjs demo /tmp/archify-demo
```

**CLI 14개 명령어:**
```bash
# 진단
archify doctor
archify demo [output-directory]
archify examples
archify guide [scenario] [--json] [--lang en|zh]

# 생성
archify render  <type> <in.json> [out.html] [--quality standard|showcase] [--repo-root path]
archify deliver <type> <in.json> [out.html] [--json] [--open] [--quality ...]   # 최종 납품
archify preview <type> <in.json> [out.html] [--no-open] [--quality ...]         # 실시간 미리보기

# 검증
archify validate <type> <in.json> [--json] [--layout-json] [--quality ...]
archify check <out.html>
archify visual-check <out.html> [--json]
archify inspect <type> <in.json>

# 비교
archify compare architecture <base.json> <head.json> [out.html] [--receipt path] [--json]

# 기타
archify migrate workflow <old.json> <new.json> --to-schema 2 [--json]
archify brands [name|alias|domain|category] [--json]
archify brands capture <url> [--json]
```

**실시간 미리보기 루프 (권장 작업 방식):**
```bash
node bin/archify.mjs preview architecture my.json out.html --quality showcase
```
- `127.0.0.1` 랜덤 포트 바인딩 (루프백 전용, 외부 접근 거부)
- 내용 digest 폴링 + 디렉터리 감시 (에디터 rename 버스트도 포착)
- 실패 시 마지막 검증 통과 산출물이 바이트 단위로 유지 + 정확한 진단 표시
- 동일 소스 재빌드 안 함, 동일 산출물 리로드 안 함
- 서버 정보가 HTML에 들어가지 않음
- `SKILL.md`: "Never start preview by default" — 명시적 요청 시에만

**meta 설정 필드:**

| 필드 | 값 | 기본·주의 |
|---|---|---|
| `quality_profile` | `standard` \| `showcase` | **showcase 권장** (9검사 전부) |
| `locale` | `en` \| `zh-CN` | **생략 권장** (한국어는 필수 생략 + 영어 폴백 고지) |
| `animation` | `trace` | 생략 = 정적. 발표/데모용만 |
| `visual_preset` | `classic` \| `signal-flow` \| `blueprint` \| `editorial` | 생략 = classic |
| `subtitle` | 문자열 | 생략 권장 (제목 반복 금지) |
| `legend` | `auto` \| `all` \| `hidden` | 생략 = auto |
| `column_fit` | `fixed` \| `spread` | sequence 전용, 생략 = fixed |
| `engineering_profile` | `deployment-ownership` | 생략 기본. 켜면 fail-closed |
| `views` | 배열 | **최대 5개** 챕터 |

**`deployment-ownership` 프로필 (켜면 엄격):**
- 모든 비외부 컴포넌트에 owner 명시 필수
- 정확히 하나의 region에 속해야 함
- region + security-group 경계 둘 다 존재 필수
- DB는 반드시 private
- 각 private 그룹은 하나의 공유 region 안
- 멤버십 변경 연결은 실제 crossing 메커니즘 명시 필수
- 실제 인프라를 들여다보지 않음 (작성된 사실만 검사)

**업데이트 체크:**

| 항목 | 내용 |
|---|---|
| 하는 일 | 고정 URL 1개 GET → 알림만 표시 |
| 주기 | 성공 시 약 72시간 (±20%), 실패 시 6h → 24h |
| 보내는 것 | 일반 HTTP 메타데이터(IP, 시간)만 |
| 안 보내는 것 | 버전, 에이전트, 프로젝트 데이터, **프롬프트**, 계정/기기 ID, ETag |
| 다운로드/설치/실행 | **절대 안 함** |
| 끄기 | `ARCHIFY_UPDATE_CHECK_DISABLED=1` |

---

### 9-2. 플러그인? 스킬? MCP? → **Agent Skill**

> **Archify는 "Agent Skill"이다. 구체적으로는 "Node.js CLI를 번들한 스킬".
> MCP 서버 아님 / IDE 플러그인 아님.**

**증거:**
1. `SKILL.md`에 `name` + `description` frontmatter (Agent Skill 규격)
2. 설치 위치가 전부 `skills/` 디렉터리
3. 설치 명령어가 `npx skills add`
4. MCP 프로토콜 흔적 0 — `stdio`/`SSE` 전송, `tools/list`·`tools/call` 핸들러, `mcp.json` 전부 없음

**세 개념 비교:**

| | Skill (Archify) | MCP 서버 | IDE 플러그인 |
|---|---|---|---|
| 정체 | AI에게 주는 문서 + 번들 도구 | 상시 실행 프로세스 | 에디터 확장 |
| 설치 | 폴더에 파일 복사 | 서버 등록 + 실행 | 마켓플레이스 |
| 통신 | 없음 (AI가 파일 읽고 쉘 실행) | JSON-RPC (stdio/SSE) | 에디터 내부 API |
| 실행 주체 | **AI가 쉘로 `node` 호출** | 서버가 상주 | 에디터가 로드 |
| 상태 | 무상태 (매번 프로세스 1회) | 상태 유지 가능 | 세션 유지 |
| 인증 | 보통 없음 | 토큰 필요한 경우 많음 | 에디터 계정 |

**AI가 Archify를 쓰는 실제 순서:**
```
1. 사용자: "아키텍처 다이어그램 그려줘"
2. AI: 설치된 스킬 중 archify의 description 매칭 → 선택
3. AI: SKILL.md 읽음 (137줄)
4. AI: schemas/architecture.schema.json + examples/web-app.architecture.json 읽음
5. AI: my.json 작성 (Write 툴)
6. AI: Bash → node bin/archify.mjs validate architecture my.json --json
7. AI: 실패 시 수리 영수증의 그 한 곳만 고침 (최대 2라운드)
8. AI: Bash → node bin/archify.mjs deliver ... --json
9. AI: 결과 HTML 경로 + 영수증 + 검증 요약 보고
```

**필요 조건:** Bash(쉘) 실행 가능 환경 + Node ≥ 18. 네트워크·토큰·서버 불필요.

**쉘이 없을 때 폴백:** 아키텍처 SVG를 `assets/template.html`에 수동 배치 + CSS 의미 클래스
사용. 품질 보증은 크게 떨어짐 → 쉘 있는 환경(Claude Code, Cursor, Codex CLI) 권장.

---

### 9-3. API 토큰 필요? → **Archify 자체는 불필요**

| 항목 | 토큰 | 비용 |
|---|---|---|
| Archify 설치 | ❌ | 무료 (MIT) |
| Archify 렌더링 | ❌ | 무료, 오프라인 |
| Archify 검증 | ❌ | 무료, 오프라인 |
| Archify 내보내기 | ❌ | 무료, 오프라인 |
| Archify 델타 비교 | ❌ | 무료, 오프라인 |
| `visual-check` | ❌ | 무료 (로컬 브라우저) |
| 저장소 증거(`SRC`) | ❌ | 무료 (로컬 git만) |
| `brands capture <url>` | ❌ (HTTP GET만) | 무료 |
| 업데이트 알림 | ❌ | 무료, 끌 수 있음 |
| **JSON을 써주는 AI** | ✅ | Claude Pro/Max, Cursor, API 종량 등 |

**네트워크를 쓰는 경우 — 딱 3가지, 전부 선택적:**

| # | 무엇 | 통신 | 끄는 법 |
|---|---|---|---|
| 1 | 업데이트 알림 | 고정 URL 1개 GET | `ARCHIFY_UPDATE_CHECK_DISABLED=1` |
| 2 | `brands capture <url>` | 사용자가 준 URL로 로고 가져오기 | 명령을 안 쓰면 됨 |
| 3 | `visual-check` | 없음 (로컬 브라우저만) | — |

**실질 비용:** AI 구독료만. 이미 쓰고 있으면 추가 0원.

**토큰 소비 주의:** 저장소 분석을 시키면 AI가 많은 파일을 읽어 토큰을 꽤 씀.
말로만 설명하면 거의 안 듦. `SKILL.md`가 "스키마 1개 + 예시 1개만 읽어라",
"첫 후보 전에 렌더러 소스 읽지 말라"로 토큰 낭비를 적극 차단하도록 설계됨.

---

### 9-4. AI 에이전트 구축에 도움? → **압도적 YES (단, "설계 교재"로서)**

```
① 도구로서 쓰기   : 내 에이전트가 다이어그램 생성  → 중간
② 설계 교재로 쓰기 : 에이전트 설계 패턴 학습       → 최상
```

#### ① 도구로서

| 용례 | 가치 |
|---|---|
| 코드 리뷰 에이전트 + "이 PR의 구조 변경 도면" | 높음 |
| 온보딩 봇 + "이 서비스 구조도" | 높음 |
| 문서화 파이프라인 (저장소별 주간 갱신) | 높음 |
| 제안서/RFP 자동 생성 + 시스템 도면 | 높음 |
| 장애 대응 봇 + 런북 다이어그램 | 중간 |

CI 통합 쉬움 (의존성 0, `--json` 기계 판독, 종료 코드 명확):
```bash
node bin/archify.mjs validate architecture arch.json --quality showcase --json || exit 1
```

#### ② 설계 교재로서 — AI 에이전트 고질병 7가지와 처방

| 병 | 증상 | Archify 처방 |
|---|---|---|
| **1. 분석 마비** | 파일 50개 읽고 컨텍스트 소진, 결과물 0 | **Artifact First** — 다음 툴 동작은 반드시 파일을 쓴다 |
| **2. 컨텍스트 폭발** | 다 넣으면 초과, 적게 넣으면 품질 저하 | **점진적 공개 + 읽기 금지 목록** |
| **3. 거짓 성공 보고** | 실패했는데 "완료했습니다" | **주장 3분할 + 명시적 금지문** |
| **4. 무한 수정 루프** | 고치면 다른 데 깨지고 반복 | **최소값 규칙 + 2라운드 상한** |
| **5. 편법 통과** | 테스트 삭제하고 "통과!" | **위조 금지 목록** (편법을 하나씩 이름 붙여 금지) |
| **6. 오류 활용 불가** | 스택트레이스 주면 엉뚱한 곳 고침 | **수리 영수증** (code/subject/evidence/supportedFixes) |
| **7. 비싼 모델 의존** | 작은 모델로는 쓰레기 | **ordinary-model-floor 벤치마크** |

#### 복사용 체크리스트

```
□ 산출물 우선       첫 툴 동작은 반드시 파일을 쓴다. 분석 산문 금지.
□ 읽기 화이트리스트  항상/조건부/금지 3단으로 명시
□ 읽기 금지 목록    "~하기 전에 이것들을 읽지 말라"를 명시
□ 수리 영수증       code + subject + evidence + supportedFixes
□ 수단 화이트리스트  진단이 허용한 수단만 쓸 수 있게
□ 1회 1수정        수정당 제어 1개
□ 최소값 규칙       2라운드 연속 개선 없으면 멈추고 정직 보고
□ 주장 3분할       기계 검증 / 환경 검증 / 사람 판단 분리
□ 위조 금지 목록    상상 가능한 편법을 하나씩 이름 붙여 금지
□ 원자적 커밋       후보 → 검사 → rename. 실패 시 이전 상태 보존
□ 해시 영수증       입력/출력 SHA-256 + 바이트 수
□ 모델 하한선 테스트 싼 모델로 돌려보고 프롬프트를 조인다
□ 명시적 상한       "최대 12개", "최대 5챕터"
□ 범위 밖 선언      "이건 안 합니다"를 문서에 쓴다
```

**효율 최고 3개:** 수리 영수증 → 읽기 금지 목록 → 주장 3분할

#### 읽어야 할 파일 순서

| # | 파일 | 배울 것 |
|---|---|---|
| 1 | `archify/SKILL.md` | **스킬 설계 전체** (필수, 137줄) |
| 2 | `PRODUCT.md` | 제품 원칙 + **anti-references** |
| 3 | `archify/references/authoring-contract.md` | 심화 계약서 구조 |
| 4 | `benchmarks/ordinary-model-floor/README.md` | 품질 하한선 측정법 |
| 5 | `.agents/skills/archify-review/SKILL.md` | **메타 스킬** (가치/비용/영향 3축) |
| 6 | `REVIEWING.md` + `CONTRIBUTING.md` | 기여 규율 |
| 7 | `docs/research-visual-evolution-round-49.md` | 디자인 실험 기록법 |

`PRODUCT.md`의 anti-references가 특히 베낄 가치 있음:
> "밀집 대시보드 껍데기, 끝없이 동일한 카드 그리드, 장식용 글래스, 그라데이션 텍스트,
> 그리고 기타 **AI 생성 인터페이스 클리셰**."

→ "하지 말 것"을 명시하는 것만으로 AI 출력 품질이 올라감.

---

### 9-5. React나 PHP로 만들 수 있어?

**중요한 구분:**
```
Ⓐ Archify를 React/PHP로 재구현 (렌더러 재작성)  → 비권장
Ⓑ Archify를 React/PHP 앱에서 활용 (래핑)        → 권장
```

#### Ⓑ React 래핑 (권장)

**패턴 1 — iframe 임베드 (10분):**
```jsx
function ArchifyViewer({ src, focus, reach, route, view }) {
  const hash = view  ? `#view=${view}`
             : route ? `#route=${route}`
             : focus ? `#focus=${focus}${reach ? `&reach=${reach}` : ''}`
             : '';
  return (
    <iframe src={`${src}${hash}`} title="Architecture diagram"
      style={{ width: '100%', height: '100%', border: 0 }}
      sandbox="allow-scripts allow-same-origin" />
  );
}
```
딥링크가 공개 계약이라서 가능. 양방향 동기화는 `postMessage`로.

**패턴 2 — 생성 API 서버 (Express):**
```js
app.post('/api/diagram', async (req, res) => {
  const { type, spec } = req.body;
  const TYPES = ['architecture','workflow','sequence','dataflow','lifecycle'];
  if (!TYPES.includes(type)) return res.status(400).json({ error: 'bad type' });

  const id = randomUUID();
  const dir = path.join('/tmp/archify-jobs', id);
  const json = path.join(dir, 'spec.json');
  const html = path.join(dir, 'out.html');
  try {
    await writeFile(json, JSON.stringify(spec), { flag: 'wx' });
    const { stdout } = await run('node', [
      '/opt/archify/bin/archify.mjs', 'deliver', type, json, html,
      '--quality', 'showcase', '--json'
    ], { timeout: 60_000, maxBuffer: 8 * 1024 * 1024 });
    res.json({ receipt: JSON.parse(stdout), html: await readFile(html, 'utf8') });
  } catch (err) {
    try { return res.status(422).json(JSON.parse(err.stdout)); }
    catch { return res.status(500).json({ error: 'render failed' }); }
  } finally {
    await rm(dir, { recursive: true, force: true });
  }
});
```
장점: `--json` 기계 판독 영수증(성공·실패 모두), 종료 코드 명확,
의존성 0 → `node:22-alpine` + 복사만, 무상태 → 수평 확장 쉬움.

**패턴 3 — React 비주얼 에디터 (최고 기회):**
```
┌─────────────────┬──────────────────────┐
│  React 캔버스    │   Archify 미리보기    │
│  (드래그로 노드)  │   (iframe)           │
│  [Users]        │   검증: 9/9 ✅        │
│    ↓            │   또는 ⚠️ 진단 +       │
│  [API]→[Redis]  │     supportedFixes   │
├─────────────────┴──────────────────────┤
│  JSON 편집기 (CodeMirror + 스키마 검증)  │
└────────────────────────────────────────┘
```
Archify가 WYSIWYG를 "의도적 범위 밖"으로 선언 → 자리가 비어있음.
`supportedFixes`를 UI 버튼으로 바꾸면 "오류를 클릭으로 고치는 에디터"가 됨.
추천 라이브러리: `react-flow` + `@codemirror/lang-json` + `ajv`

**패턴 4 — Next.js 아키텍처 포털:** 생성된 HTML들을 검색·태그·버전 비교 가능한 사내 포털로.

#### Ⓐ 재구현 비권장 이유

| 재구현해야 하는 것 | 규모 |
|---|---|
| 5개 타입 전용 렌더러 | ~63,400줄 |
| 뷰어 런타임 | 8,700줄 |
| 기하 엔진 (경로·포트분산·간격·충돌) | `geometry.mjs` 등 |
| 9단계 검증기 | `validator.mjs` + 생성 검증기 |
| 라벨 텍스트 피팅 | `text-fit.mjs` |
| 워크플로 v2 레이아웃 컴파일러 | 별도 |
| 48라운드 디자인 결정 | **복제 불가** |
| 테스트 | 114개 |

MIT니까 그냥 쓰는 게 합리적. 단 **뷰어만 React로 재작성**하는 건 의미 있을 수 있음.

#### PHP

Node를 호출하는 **오케스트레이터** 역할은 적합. 렌더링 재구현은 비권장.

```php
// proc_open + 인자 배열 → 쉘 보간 없음 (주입 방지)
$cmd = [$this->nodeBin, $this->cliPath, 'deliver', $type, $json, $html,
        '--quality', 'showcase', '--json'];
$proc = proc_open($cmd, [1 => ['pipe','w'], 2 => ['pipe','w']], $pipes, $dir);
// ...
// 종료 코드 0이 아니면 성공이라 부르지 않음 (SKILL.md 원칙)
if ($exit !== 0) return ['receipt' => $receipt, 'html' => null];
```

| 항목 | 주의 |
|---|---|
| `shell_exec`/`exec` | 쓰지 말 것. **`proc_open` + 인자 배열** |
| `max_execution_time` | 렌더링 수초~수십초. 넉넉히 또는 큐 분리 |
| 동시성 | PHP-FPM 워커 × Node 프로세스 메모리 계산 |
| 호스팅 | **공유 호스팅 거의 불가** (Node 실행 권한). VPS/컨테이너 필요 |
| 큐 권장 | Laravel Queue / Symfony Messenger |

**현실적 구조:**
```
[PHP/Laravel]  ──HTTP──>  [Node 렌더링 마이크로서비스]
 인증·검증·큐·저장·권한        Archify CLI, 의존성 0, 무상태
```

#### 결론 요약

| 접근 | 난이도 | 가치 | 추천 |
|---|---|---|---|
| React + iframe 임베드 | ⭐ | 중 | ✅ 바로 |
| React + 생성 API | ⭐⭐ | 높음 | ✅ |
| **React 비주얼 에디터** | ⭐⭐⭐⭐ | **매우 높음** | ⭐ **최고 기회** |
| Next.js 아키텍처 포털 | ⭐⭐⭐ | 높음 | ✅ 사내 도구 |
| PHP + Node 마이크로서비스 | ⭐⭐⭐ | 중 | ✅ 기존 PHP 자산 시 |
| PHP에서 직접 호출 | ⭐⭐ | 중 | ⚠️ 호스팅 제약 |
| React/PHP로 렌더러 재구현 | ⭐⭐⭐⭐⭐ | 낮음 | ❌ |

---

### 9-6. 유튜브 강의 영상 제작 가능? → **매우 YES**

| 조건 | 평가 |
|---|---|
| 시각적 결과물 | ⭐⭐⭐⭐⭐ 다이어그램이 그냥 썸네일 |
| 빠른 성취감 | ⭐⭐⭐⭐⭐ 설치 한 줄 → 5분 내 결과 |
| 검색 수요 | ⭐⭐⭐⭐ "아키텍처 다이어그램", "AI 스킬" |
| 진입장벽 | ⭐⭐⭐⭐⭐ 무료 MIT, 토큰 불필요 |
| 한국어 콘텐츠 | ⭐⭐⭐⭐⭐ 사실상 없음 → **선점 기회** |
| 라이선스 | ⭐⭐⭐⭐⭐ MIT, 강의·수익화 자유 |
| 소재 깊이 | ⭐⭐⭐⭐⭐ 입문부터 에이전트 설계론까지 |

**리스크 2가지:**
- 개발 버전(`v2.17.0-dev.1`) → API 변경 가능 → 버전 명시 자막 + 고정댓글 업데이트
- 한국어 뷰어 UI 미지원 → 솔직히 말하고 "한국어 기여 PR 보내는 영상"으로 전환

#### 트랙 A — 입문 (유입)

| # | 제목 | 길이 |
|---|---|---|
| A1 | **"말 한마디로 시스템 아키텍처 도면 뽑기"** | 8분 |
| A2 | "Mermaid 쓰다가 Archify로 갈아탄 이유" | 10분 |
| A3 | "5가지 다이어그램 타입 완전정복" | 15분 |
| A4 | "내 저장소를 AI가 읽고 그린 아키텍처" | 12분 |
| A5 | "다이어그램 하나로 발표 끝내기" | 7분 |
| A6 | 쇼츠: "이 링크 누르면 내가 보던 그 화면" | 45초 |
| A7 | 쇼츠: "AI가 그린 그림이 검사에서 탈락하는 순간" | 50초 |
| A8 | 쇼츠: "812KB 파일 하나에 들어있는 것" | 40초 |

A2는 `experiments/v3-mermaid-validation/` 스크린샷을 그대로 활용 가능
(공식 비교 데이터가 이미 저장소에 있음).

#### 트랙 B — 실무

| # | 제목 | 길이 |
|---|---|---|
| B1 | **"PR 리뷰에 아키텍처 델타 붙이기"** | 15분 |
| B2 | "신입 온보딩 자료 자동화" | 12분 |
| B3 | "실시간 미리보기 루프로 작업하기" | 10분 |
| B4 | "데이터 파이프라인 PII 경계 그리기" | 14분 |
| B5 | "소스 증거(SRC) — 도면이 실제 코드와 연결됨" | 13분 |
| B6 | "CI에서 아키텍처 문서 깨지면 PR 차단하기" | 16분 |
| B7 | "deployment-ownership — fail-closed 배포 리뷰" | 15분 |

#### 트랙 C — 고급 (차별화 핵심)

| # | 제목 | 길이 |
|---|---|---|
| **C1** | **"잘 만든 AI 스킬 해부 — SKILL.md 137줄 완독"** | 25분 |
| **C2** | **"AI가 거짓 성공 보고하는 걸 막는 3가지 장치"** | 18분 |
| **C3** | **"수리 영수증 — AI에게 오류를 주는 올바른 방법"** | 20분 |
| C4 | "AI 무한 수정 루프 끊는 '최소값 규칙'" | 12분 |
| C5 | "컨텍스트 아끼는 점진적 공개 설계" | 15분 |
| C6 | "싼 모델로도 되게 만들기 — 품질 하한선 벤치마크" | 18분 |
| **C7** | **"내 스킬 만들기 — Archify 패턴으로 처음부터"** | 35분 |
| C8 | "48라운드 디자인 실험을 기록하는 방법" | 15분 |
| C9 | "제품 원칙 문서 쓰기 — anti-references의 힘" | 12분 |
| C10 | "React로 Archify 비주얼 에디터 만들기" | 40분 |
| **C11** | **"오픈소스에 한국어 지원 PR 보내기 (실시간)"** | 30분 |

**전략:**
- 첫 영상은 A1 + A6(쇼츠) 동시 공개
- **트랙 C가 차별화** — 도구 사용법은 경쟁 심하고 수명 짧음. 설계론은 경쟁 적고 수명 길고 시청자 질 높음
- C11은 1타 3피: 콘텐츠 + 오픈소스 기여 이력 + 커뮤니티 포지션
- 유튜브 자체로 돈 벌려 하지 말고 **유료 강의 유입 채널**로 활용

**썸네일 소재 (저장소에 이미 있음, MIT):**
```
docs/assets/archify-readme-hero.png
docs/assets/archify-live-proof.gif
docs/assets/archify-dark.png / -light.png
docs/assets/architecture-delta-proof.jpg
docs/assets/archify-demo-{story,route,lens}.png
docs/issue-52-visual-evidence/contact-sheet.png
experiments/v3-mermaid-validation/output-{A,B,C}/*.png
```

**영상 필수 자막:**
```
"Archify v2.17.0-dev.1 기준 (개발 버전)"
"뷰어 UI는 영어/중국어만 — 한국어는 영어로 폴백됩니다"
"Archify는 무료(MIT). AI 구독료는 별도입니다"
```

---

## 10. 수익화 전략 (9가지 경로)

### 10-0. 법적 토대 (최우선)

```
LICENSE: MIT
based_on: Cocoon-AI/architecture-diagram-generator (MIT, v1.0)
```

**MIT 허용:** 상업적 이용 / 수정 / 재배포 / 사유 소프트웨어 포함 / 유료 판매 / SaaS 호스팅
**MIT 요구 (딱 하나):** 저작권 고지 + 라이선스 전문 포함

이 저장소는 그걸 엄격하게 지킴 (CHANGELOG):
> "소스 및 패키지 스킬 배포물은 Cocoon AI의 정확한 MIT 저작권 고지를 유지하며,
> 패키지 스테이징과 스모크 테스트는 고지나 LICENSE가 누락·변경되면 fail-closed하고,
> 결정론적 ZIP은 저장소와 동일한 LICENSE 바이트를 갖는다."

**당신도 같은 수준으로 (1순위 처리):**
```
당신_제품/
├── LICENSE
└── THIRD_PARTY_NOTICES.md
    ├── Archify (MIT) — Copyright (c) tt-a1i
    ├── architecture-diagram-generator (MIT) — Copyright (c) Cocoon AI
    └── JetBrains Mono (SIL OFL 1.1)
```

### 10-1. 전략 핵심: "범위 밖 선언 = 시장 공백"

README 명시:
> "**자동 Mermaid 파싱**, **범용 자동 레이아웃**, **호스팅 공유**, **WYSIWYG 편집**은
> 의도적으로 현재 범위 밖이다."

| 공백 | 기회 | 난이도 |
|---|---|---|
| **호스팅 공유** ❌ | SaaS (링크 공유, 팀 권한, 버전 히스토리) | ⭐⭐⭐⭐ |
| **WYSIWYG 편집** ❌ | React 비주얼 에디터 | ⭐⭐⭐⭐ |
| **자동 Mermaid 파싱** ❌ | 변환 서비스 (기존 자산 마이그레이션) | ⭐⭐⭐ |
| **한국어 로케일** ❌ | 로컬라이징 + 한국 시장 포지션 | ⭐⭐ |
| **한국어 교육 콘텐츠** ≈0 | 콘텐츠·강의·컨설팅 | ⭐⭐ |

### 10-2. 9가지 경로

```
                      투입 노력
                  낮음 ←──────→ 높음
         ┌──────────────┬──────────────┐
    높음 │ ② 콘텐츠      │ ⑦ SaaS       │
         │ ③ 외주       │ ⑧ GitHub App │
   수익  │ ④ 유료강의    │ ⑨ IR 패턴이식 │
   잠재력 ├──────────────┼──────────────┤
         │ ① 한국어기여  │ ⑤ 템플릿팩    │
    낮음 │              │ ⑥ 컨설팅      │
         └──────────────┴──────────────┘
```

**권장 순서:** ① → ② → ③ → ④ → ⑤ → ⑥ → ⑦/⑧ → ⑨
(앞쪽이 뒤쪽의 신뢰 자산을 만들어줌)

---

#### ① 한국어 로케일 기여 — 수익 0원, ROI 최고

**왜 1번인가:** 나머지 8개의 신뢰 기반.
```
"Archify 한국어 지원을 기여한 사람"
  → ② 콘텐츠의 권위 / ③ 외주의 신뢰 / ④ 강의의 설득력 / ⑥ 컨설팅 입찰 자격
```

**정확한 진입점:**
```
archify/renderers/shared/i18n.mjs        → 'en', 'zh-CN' 둘뿐
archify/renderers/shared/generated-validators.mjs  → 로케일 enum
archify/schemas/common.schema.json       → locale enum
archify/test/i18n.test.mjs               → 테스트 케이스
README.md / SKILL.md                     → 지원 언어 목록
```

`SKILL.md`가 기여 지점을 사실상 명시:
> "그 외 모든 언어는 `meta.locale`을 생략하고, 고정 Viewer UI와 `<html lang>`이
> 영어로 폴백됨을 명시적으로 고지하라."

**기술 난점 (CHANGELOG에 이미 적혀있음):**
> "폰트는 표준 HTML/SVG 산출물당 약 96KB를 추가하며... 포함된 문자는 번들 폰트를
> 오프라인으로 유지하지만, **CJK 폴백과 플랫폼 래스터화는 여전히 다를 수 있다.**"

→ 한글은 번들 JetBrains Mono에 없어 OS 폰트로 폴백 (Mac: Apple SD Gothic Neo,
Windows: 맑은 고딕, Linux: Noto Sans KR). 글자 폭이 환경마다 달라 레이아웃이 밀릴 수
있으므로 `text-fit.mjs`의 한글 폭 측정 정확성 확인 필요. 단순 번역이 아니라 실제
기술 문제 해결이라 기여 가치가 높음.

**기여 전 필독:** `CONTRIBUTING.md`, `REVIEWING.md`, `.agents/skills/archify-review/SKILL.md`

리뷰 스킬 3축 (PR 본문을 이 구조로 쓰면 통과율 상승):
> **Value**: 실제 문제가 뭐고, 누가 이득 보고, 얼마나 자주 발생하고, 지금 할 가치가 있나
> **Cost**: 구현·검증 노력 + 미래의 이해·변경·유지 비용. 기존 역량 선호, 해법을 문제에 비례
> **Impact**: 영향받는 동작과 모듈, 유지보수성이 나아지나 결합이 늘어나나
> "판단을 증거에 기반하라: 직접 검증과 기존 증거, 가정을 구별하라."

**투입: 2~4주(파트타임) / 직접 수익: 0원 / 전략 가치: 최상**

---

#### ② 콘텐츠 (유튜브 + 블로그 + 뉴스레터)

**2트랙:** 트랙 A(유입: 조회수·광고) + 트랙 C(차별화: 신뢰·전환)

**수익 경로 5개:**

| 경로 | 모델 | 기대 |
|---|---|---|
| 유튜브 광고 | CPM | 개발 채널 CPM 높음($5~15), 구독자 임계 필요 |
| 제휴(affiliate) | 커미션 | Supercode, Cursor 등 — 스폰서 구조가 이미 있는 프로젝트 |
| 블로그 (velog/티스토리/Medium) | 유입 + 광고 | SEO 롱테일 자산 |
| 뉴스레터 | 유료 구독 | "AI 에이전트 설계" 주제 월 5천~1만원 |
| **유료 강의 유입** | 전환 | **실질 수익의 핵심** |

**블로그 시리즈 (SEO):**

| # | 제목 | 타겟 키워드 |
|---|---|---|
| 1 | Archify 완전 정복 | "아키텍처 다이어그램 자동 생성" |
| 2 | Mermaid vs Archify 실측 비교 | "Mermaid 대안" |
| 3 | AI 에이전트 스킬 설계 — SKILL.md 해부 | "AI 에이전트 설계", "Claude Skill" |
| 4 | AI가 거짓 보고하는 걸 막는 3가지 장치 | "LLM 신뢰성", "할루시네이션 방지" |
| 5 | 수리 영수증 패턴 | "LLM 에러 핸들링" |
| 6 | PR 리뷰에 아키텍처 델타 붙이기 | "코드 리뷰 자동화" |
| 7 | 오픈소스에 한국어 지원 PR 보내기 | "오픈소스 기여 방법" |

3·4·5번이 핵심 (도구 사용법은 수명 짧고 설계 패턴은 길게 읽힘).

**투입: 지속적 / 수익: 월 0~300만원(성장에 따라) / 역할: 전체의 엔진**

---

#### ③ 기술문서 외주 — 가장 빠른 현금화

**패키지 A — 아키텍처 도면 패키지 (80만~150만원)**
- 아키텍처 1 + 시퀀스 1 + 워크플로 1
- HTML(인터랙티브) + PNG + SVG + 공유카드
- JSON 소스 제공 (고객이 나중에 수정 가능)
- **9/9 검증 영수증 + SHA-256**
- 수정 2회, 5~7영업일

**패키지 B — 시스템 문서화 풀세트 (300만~600만원)**
- 5가지 타입 전부 + 딥링크 가이드 문서
- 온보딩용 가이드 스토리 챕터 5개
- 소스 증거(`SRC`) 커밋 고정 + 라인 범위
- 수정 3회, 2~3주

**패키지 C — 아키텍처 리뷰 리포트 (500만~1,000만원)**
- 현재 구조 도면 + 개선안 도면 + `compare` 델타
- 리뷰 리포트 (가치/비용/영향 3축), 3~4주

**차별화 영업 멘트:**
```
❌ 경쟁사: "예쁜 다이어그램 만들어 드립니다"
✅ 당신  : "검증 영수증이 붙은 다이어그램을 만들어 드립니다"
```

| 셀링 포인트 | 근거 |
|---|---|
| "9단계 자동 검증 통과분만 납품" | `checksPassed: 9/9, errors: 0, warnings: 0` |
| "SHA-256으로 입력-출력 대응 증명" | deliver 영수증 |
| "선이 글씨를 뚫거나 가짜 합류가 보이는 그림은 애초에 산출되지 않음" | Clean Label Gate 4px, 공유 복도 8px |
| "소스 코드의 정확한 라인까지 연결 (요청 시)" | `SRC` + 커밋 고정 |
| "JSON 소스 제공 — 벤더 락인 없음" | MIT, 오픈 포맷 |
| "서버·구독 불필요, HTML 파일 하나로 영구 보관" | 자기완결형 |
| **"PR 전후 구조 변경을 기계 영수증으로 비교"** | `compare` ← 경쟁사 불가 |

**영업 채널:** 크몽/숨고(소액·실적) → 위시켓/프리모아(중간) → 링크드인(고단가)
+ 유튜브 설명란(인바운드)

**초기 전략:** 크몽에서 원가 수준 3~5건 → 리뷰·포트폴리오 → 단가 인상

**리스크 관리:**

| 리스크 | 대응 |
|---|---|
| "드래그로 수정하고 싶다" | 계약 전 명시 + JSON 소스 제공 |
| 12노드 상한 | "하나의 명확한 이야기" 철학, 큰 시스템은 여러 장 |
| 한국어 뷰어 UI | 사전 고지, 또는 ①을 먼저 |
| 저장소 접근 (NDA) | NDA + "말로만 설명" 옵션 |
| 개발 버전 | 특정 버전 고정 명시 |

**투입: 건당 1~3주 / 수익: 건당 80만~1,000만원 / 가장 빠른 현금화**

---

#### ④ 유료 강의 — 최고 수익 잠재력

**핵심: "Archify 쓰는 법"이 아니라 "Archify처럼 만드는 법"**

| 주제 | 시장 | 가격 | 수명 |
|---|---|---|---|
| ❌ "Archify 사용법" | 작음 | 3~5만원 | 짧음 |
| ✅ **"AI 에이전트 스킬 설계"** | **큼** | **20~50만원** | **길음** |

**커리큘럼: "신뢰할 수 있는 AI 에이전트 스킬 설계" (약 13시간)**

| Part | 주제 | 시간 |
|---|---|---|
| 0 | 왜 대부분의 AI 스킬이 실패하는가 (7가지 고질병) | 30분 |
| 1 | 산출물 우선 설계 (Artifact First) | 60분 |
| 2 | 컨텍스트 경제학 (점진적 공개 + 읽기 금지 목록) | 60분 |
| 3 | 검증 게이트 설계 (품질 프로필, fail-closed) | 90분 |
| **4** | **수리 영수증 — 강의의 핵심** | **120분** |
| 5 | 루프 제어 (최소값 규칙, 1회 1수정, 2라운드 상한) | 60분 |
| 6 | 정직성 강제 (주장 3분할, 위조 금지 목록) | 90분 |
| 7 | 원자적 납품 (후보→검사→rename, SHA-256) | 60분 |
| 8 | 모델 하한선 엔지니어링 (ordinary-model-floor) | 60분 |
| 9 | 캡스톤: 처음부터 스킬 하나 만들기 | 180분 |
| 10 | 배포와 유지 (버전, 결정론적 패키징, 기여 규율) | 45분 |

**가격 티어:**

| 티어 | 내용 | 가격 |
|---|---|---|
| Basic | 영상 + 코드 | 19만원 |
| Pro | + 디스코드 Q&A + 템플릿 팩 | 39만원 |
| Mentoring | + 1:1 코드 리뷰 3회 | 99만원 |
| Team | 5인 라이선스 + 사내 워크숍 1회 | 300만원 |

**판매 채널:**

| 채널 | 수수료 | 장점 | 단점 |
|---|---|---|---|
| 인프런 | 높음 | 한국 개발자 트래픽 최상 | 가격 통제 약함 |
| 자체 (Gumroad/Lemon Squeezy) | 낮음 | 가격·고객 통제 | 유입 직접 책임 |
| Udemy | 매우 높음 | 글로벌 | 할인 압박 |
| 클래스101 | 높음 | 마케팅 지원 | 개발 카테고리 약함 |

**권장:** 인프런 시작 → 트랙 레코드 후 자체 플랫폼 병행. 영어 버전은 시장 20배.

**투입: 2~3개월(제작) / 수익: 연 2,000만~1억원 / 최고 수익 잠재력**

---

#### ⑤ 산업별 템플릿 팩 — 수동적 수익

Archify는 범용 예시 11개만 제공 → 산업별 특화 템플릿이 공백.

| 팩 | 포함 | 가격 |
|---|---|---|
| 핀테크 | PG 연동, 정산 배치, KYC/AML, 거래 시퀀스, 원장 데이터플로, 결제 상태머신 | 15만원 |
| 이커머스 | 주문-결제-배송, 재고 동기화, 장바구니 세션, 추천 파이프라인, 반품 라이프사이클 | 15만원 |
| 의료/헬스케어 | EMR 연동, HL7/FHIR 데이터플로, **PHI 경계**, 진료 예약 | 20만원 |
| 공공/금융 보안 | 망분리, 전자정부 표준, 감사 로그, 권한 승인 | 25만원 |
| AI/LLM 서비스 | RAG 파이프라인, 에이전트 툴콜, 벡터DB, 프롬프트 캐싱, 스트리밍 | 20만원 |
| MSA | 서비스 메시, Saga, CQRS, 이벤트 소싱, 분산 추적 | 20만원 |
| DevOps | CI/CD 게이트, 블루그린, 롤백, 인시던트 대응, K8s 토폴로지 | 15만원 |
| **전체 번들** | 7팩 + 업데이트 1년 | **99만원** |

**각 팩 구성:**
```
핀테크-팩/
├── README.md                       사용법 + 커스터마이징 가이드
├── LICENSE + THIRD_PARTY_NOTICES   필수
├── templates/*.json                타입별 템플릿
├── rendered/                       미리보기 HTML + PNG
├── receipts/                       9/9 검증 영수증 전부
└── prompts/                        "이 템플릿을 내 시스템에 맞춰줘" AI 프롬프트
```

**왜 팔리는가:** 시간 절약 / 도메인 지식 내장 / 검증 보증 / AI 프롬프트 포함 /
컴플라이언스 각도(PHI·PII 경계가 이미 그려진 템플릿)

**확장:** 템플릿 구독 월 1.5만원 (매월 신규 2~3개 + 업데이트 + 요청 채널)

**투입: 팩당 1~2주 / 수익: 수동적·누적**

---

#### ⑥ 사내 도입 컨설팅 — 최고 단가

**타겟:** 개발자 50~500명, 마이크로서비스 많고, 아키텍처 문서가 낡았거나 없는 회사

| 패키지 | 내용 | 가격 |
|---|---|---|
| 1일 워크숍 | 도입 + 팀별 도면 1장씩 + 사내 가이드 | 200만원 |
| 1개월 도입 | 진단 → 도면 15~20장 → CI 통합 → 전사 교육 → 인수인계 | 1,000만원 |
| 3개월 전사 문서화 | 전체 시스템 맵 + 데이터/보안 경계 + 온보딩 코스 + 유지 체계 | 3,000만~5,000만원 |

**경영진 설득 (숫자로):**

| 메시지 | 근거 |
|---|---|
| "온보딩 3개월 → 1개월" | 도면 + 딥링크 가이드 스토리 |
| "구조 변경을 PR에서 잡음" | `compare` 델타 영수증 |
| "문서 노후화 방지" | CI `validate` 게이트 → 깨지면 PR 차단 |
| "벤더 락인 0" | MIT, JSON 소스 보유, 서버 불필요 |
| "구독비 0원" | 호스팅·라이선스 비용 없음 |
| "컴플라이언스 증거" | PII/PHI 경계 도면 + SHA-256 영수증 |

**보안 심사 통과 카드 (대기업·금융권 필수) — 전부 코드로 검증된 사실:**
```
✅ 렌더러·CLI·델타 엔진에 네트워크 호출 0건 (grep 검증)
✅ API 키·토큰 관련 코드 0건
✅ 런타임 의존성 0개 (node_modules 없음)
✅ 텔레메트리 없음
✅ 유일한 통신: 업데이트 알림 1개 (고정 URL 하드코딩, 끌 수 있음)
   → ARCHIFY_UPDATE_CHECK_DISABLED=1
✅ 업데이트 체크가 보내지 않는 것: 버전, 에이전트, 프로젝트 데이터,
   프롬프트, 계정/기기 ID, ETag
✅ 자동 다운로드·설치·실행 절대 안 함
✅ preview는 127.0.0.1 루프백 전용, 외부 호스트·쓰기 요청 거부
✅ MIT 라이선스 (법무 검토 용이)
✅ 테스트 114개, CI 3개 OS (Ubuntu/macOS/Windows)
```
→ 에어갭 환경, 금융권 폐쇄망에서도 통과 가능. 상업 도구 대비 압도적 강점.

**투입: 건당 1일~3개월 / 수익: 200만~5,000만원 / 최고 단가**

---

#### ⑦ 웹 SaaS 래퍼 — "Archify Cloud"

**근거:** README가 "호스팅 공유는 의도적으로 범위 밖"이라 선언 → 자리가 비어있음

**제품 구성:**
```
[웹 에디터]              [생성]
├ react-flow 캔버스      ├ 자연어 → JSON (AI)
├ 드래그로 노드 배치      ├ GitHub 저장소 연결
├ 실시간 검증 표시        └ Mermaid 붙여넣기 변환
└ supportedFixes 버튼

[공유]                   [협업]
├ 영구 링크              ├ 팀 워크스페이스
├ 비밀 링크 + 만료        ├ 역할 권한
├ 임베드 코드            ├ 코멘트 스레드
└ 딥링크 보존            └ 변경 알림

[버전]                   [통합]
├ 히스토리 타임라인       ├ GitHub App (PR 델타)
├ compare 델타 UI        ├ Slack 알림
└ 롤백                   ├ Notion/Confluence 임베드
                        └ REST API

⚙️ 엔진: Archify CLI (MIT) — 수정 없이 그대로 사용
```

**가격:**

| 플랜 | 가격 | 내용 |
|---|---|---|
| Free | 0원 | 다이어그램 3개, 공개 링크만, 워터마크 |
| Pro | $12/월 | 무제한, 비밀 링크, 버전 히스토리, 임베드 |
| Team | $20/사용자/월 | 워크스페이스, 권한, 코멘트, GitHub App |
| Enterprise | 문의 | SSO/SAML, 온프레미스, 감사 로그, SLA |

**기술 스택:**
```
프론트  : Next.js + react-flow + CodeMirror + Tailwind
백엔드  : Node.js (Archify CLI를 자식 프로세스로)
큐      : BullMQ
스토리지 : S3/R2 (HTML) + Postgres (메타데이터)
인증    : Clerk / Auth.js
결제    : Stripe / Lemon Squeezy (한국은 Paddle)
배포    : Fly.io / Railway / Vercel + 별도 워커
```

렌더링 워커가 의존성 0개라 컨테이너가 가벼움:
```dockerfile
FROM node:22-alpine
COPY archify/ /opt/archify/
# npm install 없음
CMD ["node", "worker.mjs"]
```

**리스크:**

| 리스크 | 심각도 | 대응 |
|---|---|---|
| 원작자가 호스팅 시작 | 높음 | 협업 제안 / 차별화(한국 시장, 특정 산업) |
| 경쟁 (Eraser.io, Multiplayer, ilograph) | 높음 | **"검증 영수증"이 차별점** |
| 운영 부담 (24/7) | 중간 | 초기엔 Free 티어 제한적으로 |
| AI 비용 | 중간 | 저렴 모델 + 캐싱 |
| 라이선스 준수 | 낮음 | MIT 고지만 지키면 됨 (단 반드시) |
| 개발 버전 추적 | 중간 | 특정 버전 핀 + 정기 업그레이드 |

**추천 — 적대가 아닌 협업 경로:**
```
1) ① 한국어 기여로 신뢰 쌓기
2) 이슈/디스커션에서 "호스팅 래퍼 만들어도 되나?" 공개 질문
3) 범위 밖임을 확인받고 진행
4) 제품에 "Powered by Archify" 명시 + 저장소 링크
5) 수익 일부 스폰서로 환원 (이미 스폰서 구조 있음)
```

**투입: 3~6개월 / 수익: MRR $0 → $10k+ / 최고 상한, 최고 리스크**

---

#### ⑧ 아키텍처 리뷰 GitHub App

`compare` 기능을 GitHub App으로 자동화:
```
PR 생성 → base/head 아키텍처 JSON → archify compare --json → PR 자동 코멘트

┌─────────────────────────────────────┐
│ 🏗️ 아키텍처 변경 감지               │
│ ➕ 추가: Redis 캐시, SQS 큐          │
│ ➖ 삭제: (없음)                      │
│ 🔄 변경: API Server 라벨             │
│ ↔️ 이동: Worker (880,440)→(950,440)  │
│ 🔀 재라우팅: api→db 경로              │
│ [Before/Delta/After 보기 ↗]         │
│ 검증: 9/9 ✅  영수증: a7f3...        │
└─────────────────────────────────────┘
```

**의도적으로 하지 말 것 (Archify 정직성 원칙):**
```
❌ "이 PR은 안전합니다"           ← 추론, 거짓일 수 있음
❌ "리스크: 중간"                ← 근거 없음
❌ "머지 가능합니다"              ← 책임 못 짐
✅ "이 5개 사실이 변경되었습니다"   ← 검증 가능
```
이 제약을 지키는 게 오히려 셀링 포인트 ("거짓 확신을 주지 않습니다").

**가격:** 공개 저장소 무료 / 비공개 $8/저장소/월 / Org $40/월 / Enterprise 문의

GitHub Marketplace 등록 시 유입 자동. 아키텍처 도면 자동 생성은 경쟁 거의 없음.

**투입: 1~2개월 / 수익: MRR 성장형 / ⑦보다 작고 빠름 — 먼저 해볼 가치**

---

#### ⑨ "검증된 IR" 패턴을 다른 도메인에 이식 — 최대 상한

**핵심 통찰: Archify의 진짜 발명은 다이어그램이 아니다.**

```
이식 가능한 패턴:
  AI가 타입 있는 IR을 쓴다
    → 결정론적 검증기가 거부/승인한다
    → 실패 시 수리 영수증 (code/subject/evidence/supportedFixes)
    → 통과한 것만 원자적으로 커밋 + 해시 영수증
```

**이식 가능 도메인 10개:**

| # | 도메인 | IR | 검증 규칙 예 | 시장 |
|---|---|---|---|---|
| 1 | **ERD / DB 스키마** | 테이블·관계·제약 | 고아 FK, 순환 참조, 정규화 위반, 인덱스 누락 | 큼 |
| 2 | 조직도 / HR | 사람·직책·보고선 | 순환 보고, 과도한 span, 공석 | 중 |
| 3 | **BPMN / 업무 프로세스** | 태스크·게이트웨이·레인 | 도달 불가 태스크, 데드락, 미종료 경로 | 큼 |
| 4 | 테스트 플랜 | 시나리오·커버리지 | 미커버 요구사항, 중복 케이스 | 중 |
| 5 | API 명세 (OpenAPI) | 엔드포인트·스키마 | 깨진 $ref, 누락 응답, 네이밍 불일치 | 큼 |
| 6 | **IaC 다이어그램** | 리소스·의존성 | 순환 의존, 미사용 리소스, 보안그룹 과대 개방 | 큼 |
| 7 | 네트워크 토폴로지 | 노드·링크·VLAN | IP 충돌, 서브넷 중복, 단일 장애점 | 중 |
| 8 | 학습 커리큘럼 | 모듈·선수관계 | 순환 선수, 도달 불가 모듈 | 중 |
| 9 | 간트/프로젝트 계획 | 태스크·의존·자원 | 순환 의존, 자원 과배정, 임계경로 오류 | 큼 |
| 10 | 법률 문서 구조 | 조항·참조 | 깨진 상호참조, 모순 조항, 미정의 용어 | 중 |

**가장 유망한 3개:**

🥇 **ERD / DB 스키마** — 개발자 전원이 필요. 기존 도구(dbdiagram, DrawSQL)는 검증이 약함
🥈 **IaC 다이어그램** — 클라우드 비용·보안 직결, 수요 폭발 중. 실제 `.tf`에서 IR 추출
🥉 **BPMN** — 비개발 시장(기획·운영·컨설팅), 경쟁 적음, BPMN 2.0 표준 준수 + 자동 검증

**실행 방법:**
```
1) archify를 fork
2) renderers/<새타입>/ 추가
3) schemas/<새타입>.schema.json 작성
4) 검증 규칙 정의 (9개 수준으로)
5) 수리 영수증 코드 체계 설계
6) SKILL.md 작성 (Archify 패턴 그대로)
7) ordinary-model-floor 벤치마크 작성
8) npx skills add <계정>/<이름> 로 배포
```
인프라(CLI 골격, 원자적 납품, 뷰어, 내보내기, 테스트 하네스)를 거의 그대로 재사용 가능.

**수익 모델:** 오픈소스 코어(MIT) + 유료 템플릿 팩 + SaaS + Enterprise 지원 + 컨설팅
= Archify의 9가지 경로를 그대로 복제

**투입: 3~12개월 / 수익: 상한 없음 / 유일하게 "제품"이 되는 경로**

---

### 10-3. 12주 실행 로드맵 (파트타임)

```
Week 1-2   ① 한국어 로케일 조사 + 기여 준비
           ├ CONTRIBUTING.md, REVIEWING.md 숙독
           ├ i18n.mjs 구조 파악
           ├ CJK 폭 측정 이슈 재현
           └ 이슈 등록 ("Korean locale support?")

Week 3-4   ② 콘텐츠 시작 (①과 병행)
           ├ A1 영상 + A6 쇼츠 공개
           ├ 블로그 1~2편
           └ ① PR 제출 (과정을 C11 영상으로 녹화)

Week 5-6   ③ 외주 첫 수주
           ├ 크몽/숨고 프로필 개설
           ├ 포트폴리오 3건 (오픈소스 저장소로 샘플 제작)
           └ 원가 수준 1~2건 수주 → 리뷰 확보

Week 7-8   ② 콘텐츠 가속 + ⑤ 템플릿 팩 1개
           ├ 트랙 C 영상 2~3편 (C1, C3 우선)
           ├ 템플릿 팩 1개 완성
           └ Gumroad 판매 시작

Week 9-10  ④ 강의 기획 + 사전 판매
           ├ 커리큘럼 확정
           ├ Part 0~2 제작
           └ 사전 등록 오픈 (얼리버드)

Week 11-12 ③ 외주 단가 인상 + ⑥ 컨설팅 리드
           ├ 위시켓/링크드인 진출
           ├ 1일 워크숍 상품 패키징
           └ ④ 강의 Part 3~5 제작

이후        ⑦/⑧/⑨ 중 하나 선택 집중
```

**수익 예상 (추정 — 콘텐츠 성장은 예측 어렵고 외주·컨설팅은 영업력에 좌우됨):**

| 시점 | 보수적 | 낙관적 |
|---|---|---|
| 3개월 | 100만원 (외주 1건) | 500만원 (외주 2건 + 템플릿) |
| 6개월 | 500만원 | 2,000만원 (강의 출시) |
| 12개월 | 1,500만원 | 8,000만원 (강의 + 컨설팅 + SaaS 초기) |

**지금 당장 할 3가지:**
```
1️⃣ LICENSE + THIRD_PARTY_NOTICES 처리 방법 숙지 (모든 수익화의 법적 전제)
2️⃣ CONTRIBUTING.md + REVIEWING.md + archify-review/SKILL.md 읽기
3️⃣ A1 영상 기획서 작성 (설치→결과 8분)
```

**가장 확실한 조합: ① + ② + ③ + ④**
서로를 강화하고, 초기 투자가 거의 0이며, 실패해도 손실이 작음.
⑦(SaaS)·⑨(IR 이식)는 상한이 크지만 ①~④로 기반을 만든 다음에.

---

## 11. 한계 (솔직한 정리)

| 한계 | 내용 |
|---|---|
| 🇰🇷 **한국어 뷰어 UI 미지원** | `i18n.mjs`에 `en`, `zh-CN`만. 한국어는 `locale` 생략 + 영어 폴백 명시 고지 필요. 노드 라벨·카드 내용은 한국어 유지됨 |
| 📦 **범위 외 (의도적)** | Mermaid 자동 파싱 ❌ / 범용 자동 레이아웃 ❌ / 호스팅 공유 ❌ / WYSIWYG ❌ |
| 🖱️ **드래그 편집 불가** | 수정은 AI에게 말하기 또는 JSON 직접 편집 |
| 🖥️ **데스크톱 우선** | 모바일은 "안전한 담김 + 사용 가능한 컨트롤"만. 별도 모바일 제품 아님 |
| 🚧 **개발 버전** | `v2.17.0-dev.1` — 정식 릴리스 아님 |
| 📏 **12노드 권장 상한** | 거대 모놀리스 전체를 한 장에 넣는 용도 아님 |
| 🤖 **AI 품질 의존** | Archify는 렌더러. JSON을 쓰는 건 AI이므로 AI가 시스템을 오해하면 예쁘게 틀린 그림이 나옴 |
| 🔤 **CJK 폰트 폴백** | JetBrains Mono 내장(약 96KB). CJK는 플랫폼 폴백 → 한글 렌더링이 환경마다 다를 수 있음 |
| 🌐 **업데이트 체크** | 고정 매니페스트 GET(약 72h ±20%). 버전·프롬프트·계정 정보 미전송. 자동 설치 안 함. `ARCHIFY_UPDATE_CHECK_DISABLED=1`로 차단 |
| 🧪 **visual-check의 한계** | 기계 측정 + 스크린샷. 지각적 품질은 사람이 판단해야 함 (설계상 의도) |

---

## 12. 종합 평가

| 항목 | 평가 |
|---|---|
| 엔지니어링 완성도 | ⭐⭐⭐⭐⭐ 테스트 114개, 9단계 검증, 원자적 커밋, 결정론적 ZIP, 3 OS CI |
| 설치 편의성 | ⭐⭐⭐⭐⭐ 의존성 0, `npx skills add` 한 줄 |
| 산출물 품질 | ⭐⭐⭐⭐⭐ 48라운드 디자인 연구, 4프리셋, Proof Lab 11개 실증 |
| AI 에이전트 친화성 | ⭐⭐⭐⭐⭐ 이 분야 설계 레퍼런스급 |
| 문서화 | ⭐⭐⭐⭐⭐ README 3국어, 4개 심화 계약서, 쿠킅북, 로드맵 |
| 한국어 지원 | ⭐⭐☆☆☆ 뷰어 UI 미지원 (영어 폴백) |
| 자유 편집성 | ⭐⭐☆☆☆ 의도적 제한 (WYSIWYG 아님) |

**결론:**

> Archify는 **"AI의 설계 판단을 받아서, 기계적으로 검사하고, 혼자 돌아가는
> 인터랙티브 문서로 컴파일하는 제면기"** 다. AI는 좌표와 관계를 글자로 쓰고,
> Archify는 9개 검사로 거짓과 추함을 거부하고, 통과한 것만 원자적으로 교체하며,
> 결과물은 서버 없이 열리는 HTML 1개다.
>
> 가장 특이한 점은 **정직성이 코드로 강제**된다는 것 — AI가 하지 않은 검사를
> 했다고 말할 수 없고, 추론하지 않은 영향도를 주장할 수 없고, 라벨을 지워서
> 통과하는 편법을 쓸 수 없다.
>
> 그리고 이 설계 철학 자체가 도구보다 더 값진 학습 자산이다.

---

## 13. 7가지 질문 최종 요약표

| 질문 | 답 |
|---|---|
| **설치/사용법** | `npx skills add tt-a1i/archify -g` → `doctor` 확인 → 자연어로 요청. CLI 14개 명령 |
| **플러그인/스킬/MCP** | **Agent Skill** (+ 번들 Node CLI). MCP ❌, IDE 플러그인 ❌ |
| **API 토큰** | ❌ Archify 자체는 불필요 (네트워크 호출 0건 검증). AI 구독만 필요 |
| **AI 에이전트 구축** | ✅ 매우 유용. 단 "도구"보다 **"설계 교재"** 로서 가치가 압도적 |
| **수익화** | ✅ 9가지 경로. 권장 조합 ① 기여 + ② 콘텐츠 + ③ 외주 + ④ 강의 |
| **React/PHP** | React 래핑 ✅ (**비주얼 에디터가 최고 기회**) / PHP는 Node 호출 오케스트레이터 ✅ / 재구현 ❌ |
| **유튜브** | ✅ 소재 과잉. 트랙 A(유입) + C(차별화) 조합. C11(한국어 기여)가 1타 3피 |

---

## 부록: 자주 쓰는 명령어 치트시트

```bash
# 환경 확인
node bin/archify.mjs doctor
node bin/archify.mjs demo /tmp/archify-demo

# 타입 추천
node bin/archify.mjs guide "Show CI/CD checks, approval, deploy, and rollback"
node bin/archify.mjs guide "Map Kafka topics, consumer groups, replay, and DLQ" --json

# 검증 (showcase = 9검사 전부)
node bin/archify.mjs validate workflow my.workflow.json --quality showcase --json

# 워크플로 v2 기하 진단
node bin/archify.mjs validate workflow my.workflow.json --layout-json

# 최종 납품
node bin/archify.mjs deliver workflow my.workflow.json out.html --quality showcase --json
node bin/archify.mjs deliver workflow my.workflow.json out.html --quality showcase --open --json

# 실시간 미리보기 루프
node bin/archify.mjs preview workflow my.workflow.json out.html --quality showcase

# 브라우저 증거 수집 (납품된 HTML을 수정/재렌더 안 함)
node bin/archify.mjs visual-check out.html --json

# 아키텍처 델타 (PR 리뷰)
node bin/archify.mjs compare architecture base.json head.json delta.html --json

# 브랜드 마크
node bin/archify.mjs brands "postgres" --json
node bin/archify.mjs brands capture "https://example.com" --json

# 워크플로 v1 → v2 마이그레이션
node bin/archify.mjs migrate workflow old.json new.json --to-schema 2 --json

# 업데이트 체크 끄기
export ARCHIFY_UPDATE_CHECK_DISABLED=1
```

---

## 라이선스 및 출처

- **Archify**: MIT — Copyright (c) tt-a1i — https://github.com/tt-a1i/archify
- **기반 프로젝트**: architecture-diagram-generator (MIT v1.0) — Copyright (c) Cocoon AI
  — https://github.com/Cocoon-AI/architecture-diagram-generator
- **JetBrains Mono**: SIL Open Font License 1.1
- 상세 고지는 저장소의 [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md) 참조

이 분석 문서의 인용문은 모두 저장소 내 `README.md`, `archify/SKILL.md`, `PRODUCT.md`,
`ROADMAP.md`, `CHANGELOG.md`, `.agents/skills/archify-review/SKILL.md`에서 가져왔습니다.
실행 결과와 `grep` 검증은 `v2.17.0-dev.1` (커밋 `72aebf6`) 기준입니다.
