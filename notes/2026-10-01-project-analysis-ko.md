# html-video 전수조사 분석 & 활용/수익화 정리 (한국어)

> 작성일: 2026-10-01
> 저장소(fork): <https://github.com/bmshin94/html-video>
> 원본(upstream): <https://github.com/nexu-io/html-video>
> 분석 기준 커밋: `47da58c` (branch `claude/trusting-wozniak-aiop7k`)
> 분석 도구: Claude Code (Opus 5)

이 문서는 저장소 전체를 읽고 수행한 분석 대화를 하나로 정리한 기록이다.
① 프로젝트 정체 ② 쉬운 설명 ③ 설치·분류·토큰·에이전트·React/PHP·유튜브 Q&A
④ 수익화 아이디어 순서로 구성했다.

---

## 1. 프로젝트 정체 — 한 줄 정의

**`html-video` = HTML을 영상(MP4)으로 바꿔주는 "메타 레이어"**

로컬에 설치된 코딩 AI 에이전트(Claude Code · Cursor · Codex · Gemini CLI 등)에게
"이런 영상을 만들어줘"라고 하면, 에이전트가 HTML 애니메이션을 작성하고
→ 헤드리스 Chromium이 녹화 → ffmpeg가 MP4로 인코딩한다. **전 과정이 로컬에서 끝난다.**

| 항목 | 내용 |
|---|---|
| 라이선스 | Apache-2.0 (상업적 사용 허용, 렌더링 과금 없음) |
| 스택 | TypeScript, pnpm 모노레포, Node 20+ |
| 코드 규모 | 핵심 TS 약 1만 줄 + 스튜디오 UI 약 5,100줄 |
| 엔진 상태 | Hyperframes **완성**, Remotion **부분**, Motion Canvas/Revideo/Manim **계획** |
| 템플릿 | 23개 (hyperframes 22 + remotion 1) |
| 지원 에이전트 | 15개 정의 (CLI 14 + Anthropic Messages API) |

### 차별점 (vs Hyperframes 단독 사용)

| 차원 | Hyperframes | html-video |
|---|---|---|
| 렌더 엔진 | 단일 (GSAP + Puppeteer) | 멀티 엔진 pluggable |
| Authoring | 단일 (HTML+CSS+GSAP) | backend에 따름 (HTML / React / TS-generator) |
| 에이전트 | 자체 skill | 15개 에이전트 교차 지원 + skill 주입 |
| 템플릿 | 자체 라이브러리 | 생태계 교차 큐레이션 + 커뮤니티 기여 |

---

## 2. 폴더 전수조사

```
html-video/
├── README.md / README.zh-CN.md   공개 문서 (영문 + 중문) — 한국어 없음
├── CLAUDE.md                     내부 작업 노트 (중문) + 사용자 페르소나 가이드
├── CONTRIBUTING.md               기여 가이드 (영/중 2개국어)
├── ATTRIBUTIONS.md / LICENSE     출처 고지 + Apache-2.0
├── biome.json / tsconfig*.json   린트/빌드 설정
├── packages/                     실제 코드 8개 패키지
├── templates/                    영상 템플릿 23개
├── research/                     RFC 설계 문서 (spec-01 ~ spec-09)
├── notes/                        의사결정 기록 (이 문서 포함)
└── docs/assets/                  README용 미리보기 이미지
```

### 2.1 packages/ — 8개 패키지

| 패키지 | 줄 수 | 역할 |
|---|---:|---|
| `core` | 2,087 | 타입 정의 + 프로젝트 오케스트레이터. `EngineAdapter` 규약, 템플릿 레지스트리, 에셋 스토어, MiniMax 오디오, ffmpeg 믹싱 |
| `content-graph` | 351 | 멀티프레임 스토리보드 IR. 노드(entity/data/text) + 엣지(sequence/dependency/contrast) → `validate` + `topoSort` |
| `runtime` | 1,326 | 에이전트 런타임. PATH 자동 탐지 → spawn → 출력 스트리밍. 정의 15개 + ACP 클라이언트 |
| `adapter-hyperframes` | 714 | **실동작 엔진 #1.** Playwright Chromium `recordVideo` → ffmpeg `libx264 crf20` |
| `adapter-remotion` | 556 | 엔진 #2. Phase1 = HTML 브릿지, Phase2 = 네이티브 React tsx |
| `cli` | 4,862 | `html-video` 명령어 + 스튜디오 HTTP 서버(3,472줄) + 링크/레포 fetch |
| `project-studio` | 5,132 | 브라우저 스튜디오 UI (바닐라 HTML/JS, 빌드 불필요) + i18n |
| `studio-next` | 52 | 실험용 (hyperframes/studio 임베드) |

### 2.2 templates/ — 23개, 독립 패키지 구조

```
templates/frame-glitch-title/
├── template.html-video.yaml   매니페스트 (에이전트가 읽는 설명서)
├── SKILL.md                   AI용 디자인 스펙 (한/중/영 메타)
├── example.md                 예시 데이터
├── preview.png                썸네일
└── source/index.html          실제 애니메이션 HTML
```

매니페스트가 담는 정보 — AI가 HTML을 열지 않고도 선택할 수 있게 한다.

- `category` / `tags` / `best_for` — 의도 매칭 (`search-templates`가 스코어링)
- `output` — 해상도/비율/fps/길이 범위/알파채널/오디오 지원
- `inputs.schema` — JSON Schema로 채울 슬롯 명시
- `license` — SPDX + `attribution_required` / `redistribution_allowed` / `commercial_use`

**카테고리 분포**: presentation 9 · data-viz 4 · product-demo 2 · explainer 2 ·
social-shorts 2 · marketing 1 · ambient 1 · intro-outro 1

### 2.3 research/ — RFC 문서

| 문서 | 내용 |
|---|---|
| spec-01 | 엔진 어댑터 인터페이스 (단일 `render()` 계약) |
| spec-02 | 템플릿 메타데이터 포맷 |
| spec-03 | 에이전트 스킬 설계 |
| spec-04 / 05 | 스토리보드 → 프로젝트 중심 워크플로 |
| spec-06 | content-graph (멀티프레임 IR) |
| spec-07 | PPT→템플릿 변환 + 3층 저작권 표기 규범 |
| spec-08 | Remotion 어댑터 |
| spec-09 | 멀티엔진 UX (hyperframes = 기반, Remotion = 사용자가 켜는 증강) |

---

## 3. 파이프라인

```
프롬프트 / 기사 링크 / GitHub 레포 주소
        ↓
① 소스 fetch     서버사이드로 URL·레포를 긁어 Markdown으로 평탄화
                 (WeChat 공중호 기사 지원, GitHub는 public API)
        ↓
② 에이전트 루프  로컬 AI CLI가 자료 + 템플릿 스타일을 읽고
                 content-graph(스토리보드) + 프레임별 HTML 생성
        ↓
③ content-graph  노드/엣지 위상정렬 → 장면 순서·타이밍 확정
        ↓
④ 프레임 HTML    각 노드 = 독립 애니메이션 HTML 파일
        ↓
⑤ 렌더           헤드리스 Chromium 로드 + 녹화 → webm
        ↓
⑥ ffmpeg         webm→mp4(libx264) → concat 결합
                 (+ MiniMax 음악·나레이션 믹싱)
        ↓
      output.mp4
```

단일 프레임은 content-graph를 건너뛰는 fast path를 쓴다.
②~④가 "메타 레이어"의 본질 — 에이전트는 스토리보드를, 엔진은 그리는 방법을 담당하며 서로 침범하지 않는다.
⑤만 엔진 고유 영역이므로 Remotion/Motion Canvas로 교체하면 그 박스만 바뀐다.

---

## 4. 쉬운 설명 (비유 + 핵심 개념)

### 햄버거 가게 비유

| 가게 요소 | html-video | 하는 일 |
|---|---|---|
| 요리사 | AI 에이전트 | 주문 듣고 요리 |
| 레시피북 | `templates/` 23개 | "글리치 타이틀은 이렇게" |
| 주문서 | content-graph | "1→2→3 장면" |
| 오븐 | Chromium + ffmpeg | HTML을 구워 MP4로 |
| 카운터 | 스튜디오 UI | 주문 접수 + 결과 표시 |
| 오븐 교체 | 엔진 어댑터 | 오븐 바꿔도 레시피 그대로 |

핵심은 **"오븐을 바꿔 끼울 수 있는 규격"을 정의한 것**이다.

### 실제 사용 순서

1. `pnpm install && pnpm -r build`
2. `node packages/cli/dist/bin.js studio`
3. 브라우저에서 `localhost:3071`
4. 새 프로젝트 → 채팅창에 요청 또는 기사 링크 붙여넣기
5. AI가 질문 (스타일 / 길이 / 장면 수) → 카드 선택
6. HTML 생성 → 미리보기 → 프레임별 텍스트 수정
7. MP4 내보내기

### 핵심 개념 3개

**① `EngineAdapter` (어댑터 패턴)**

```typescript
interface EngineAdapter {
  render(input, ctx): Promise<RenderOutput>;
  validate(template): ValidationResult;
  preview?(template, ctx): Promise<PreviewHandle>;
  renderToHtml?(input, ctx): Promise<HtmlSceneOutput>;
}
```

멀티탭 콘센트와 같다. 플러그 모양만 맞으면 어떤 엔진도 꽂힌다.
`capabilities`에 `bestFor`뿐 아니라 **`weaknesses`까지 선언**하는 것이 특징 —
AI가 "이 엔진이 못하는 것"을 알고 선택할 수 있다.

**② `content-graph` (스토리보드 그래프)**

```
[인트로] --sequence--> [데이터1] --contrast--> [데이터2] --sequence--> [아웃트로]
```

- `dependency` — 반드시 순서를 지킴 (하드 제약)
- `sequence` — 되도록 순서를 지킴 (소프트 정렬)
- `contrast` — 정렬에 참여하지 않음 (의미적 대비만 표현)

AI에게 최종 결과물을 바로 만들라고 하지 않고 **구조를 먼저 출력**시킨 뒤
코드가 검증·정렬하고 그 다음 렌더한다 → AI 출력 신뢰성을 올리는 핵심 패턴.

**③ 에이전트 런타임 추상화**

각 에이전트는 `{ bin, versionArgs, buildArgs, streamFormat, promptViaStdin }` 선언만으로 추가된다.
claude / cursor-agent / codex / gemini / grok / qwen / opencode / copilot / aider /
hermes / trae-cli / qoder / amr(vela) / anthropic-api 등 15개.
AI를 교체해도 상위 코드 수정이 필요 없다.

---

## 5. Q&A

### Q1. 설치 및 사용법

**필수 준비물**

| 항목 | 버전 | 확인 |
|---|---|---|
| Node.js | 20+ | `node --version` |
| pnpm | 9+ | `pnpm --version` |
| ffmpeg | 최신 | `ffmpeg -version` |
| Chromium | — | `npx playwright install chromium` |

**설치**

```bash
git clone https://github.com/bmshin94/html-video.git
cd html-video
pnpm install
pnpm -r build
node packages/cli/dist/bin.js doctor      # 환경 점검 (가장 먼저)
npx playwright install chromium           # 시스템 Chromium 없을 때
node packages/cli/dist/bin.js studio --port 3071
```

**CLI 전체 명령어**

```bash
# 진단 / 조회
html-video doctor
html-video list-engines
html-video search-templates --intent "github stars race" --top 3
html-video inspect-template frame-glitch-title

# 프로젝트 라이프사이클
html-video project-create --name "데모" --intent "..." --aspect 16:9 --commercial
html-video project-list
html-video project-show <id>
html-video project-delete <id>

# 소재 / 설정
html-video project-add-asset <id> --file ./logo.png --caption "로고"
html-video project-add-asset <id> --inline-data-file ./data.json
html-video project-remove-asset <id> --asset <assetId>
html-video project-set-template <id> --template frame-nyt-graph
html-video project-set-var <id> --key title --value '"안녕"'
html-video project-set-vars <id> --vars-file ./vars.json

# 출력
html-video project-preview <id>
html-video project-render <id> --output out.mp4 --stream-progress
html-video studio --port 3071
```

모든 명령이 `--json` 기본 ON, `--stream-progress`는 NDJSON 스트림 → 자동화에 유리하다.

**스튜디오 대화 흐름 (5단계 상태머신)**
`템플릿 → 내용 → 스타일 → 포맷 → 확인` → 생성 → 생성 후 반복 수정
(pin한 프레임은 단일 편집, 그 외는 메뉴로 재확인)

### Q2. 플러그인 / 스킬 / MCP 중 무엇인가

**정답: 셋 다 아니다. "독립 실행 애플리케이션 + 모노레포 라이브러리"다.**

| 분류 | 해당 | 설명 |
|---|:---:|---|
| 독립 CLI 앱 | O | `bin: { "html-video": "./dist/bin.js" }` |
| 로컬 웹 앱 | O | Node HTTP 서버 + 브라우저 UI (3071) |
| npm 라이브러리 | O | `@html-video/core` 등 import 가능 |
| Claude 플러그인 | X | `.claude-plugin/` 없음. 반대로 Claude Code를 부품으로 사용 |
| Skill | 부분 | 템플릿마다 `SKILL.md` 존재. 단 Claude Skill 규격이 아닌 자체 규격 |
| MCP 서버 | X | MCP 코드 없음. CLI를 직접 spawn하는 방식 |

**관계도**

```
사용자 → html-video 스튜디오 (지휘자)
            ↓ child_process.spawn
     Claude Code / Cursor / Codex ... (부품)
            ↓
         HTML 생성
            ↓
     Chromium + ffmpeg → MP4
```

html-video가 AI를 호출하는 구조다. 역방향(AI가 html-video를 호출)이 비어 있으므로
**MCP 서버 래퍼가 가장 가치 있는 기여 포인트**다.

### Q3. API 토큰이 필요한가

| 용도 | 토큰 | 상세 |
|---|:---:|---|
| 렌더링(MP4 생성) | 불필요 | 100% 로컬 (Chromium + ffmpeg), 네트워크 0 |
| 템플릿 / CLI / 스튜디오 | 불필요 | 전부 로컬 |
| AI 생성 (CLI 방식) | CLI 자체 인증 | `claude` CLI가 로그인돼 있으면 추가 키 불필요 |
| AI 생성 (API 방식) | 필요 | `ANTHROPIC_API_KEY` 또는 `ANTHROPIC_AUTH_TOKEN` |
| 음악 / 나레이션 | 필요 | MiniMax API 키 (Settings → Audio) |
| 링크 / 레포 가져오기 | 불필요 | 공개 fetch + GitHub public API |

**환경변수**

```bash
ANTHROPIC_API_KEY=sk-ant-...                   # 1순위
ANTHROPIC_AUTH_TOKEN=...                       # 2순위 (OpenRouter 라우팅)
ANTHROPIC_BASE_URL=https://openrouter.ai/api   # OpenRouter 사용 시
HV_AGENT_MODEL=claude-opus-5                   # 모델 오버라이드
```

주의: 코드의 기본 모델은 `claude-sonnet-4-6`이지만 현재 최신은 Claude 5 패밀리
(`claude-opus-5`, `claude-sonnet-5-5`, `claude-fable-5-1`)이므로 `HV_AGENT_MODEL`로 지정하는 것이 좋다.

**최저 비용 조합**: 기존 Claude Code 구독 + 로컬 렌더 + 음악 off = 추가 비용 0원

### Q4. AI 에이전트 구축에 도움이 되는가

도움이 된다. 바로 재사용 가능한 자산이 다섯 가지 있다.

1. **`packages/runtime` — 멀티 에이전트 추상화 레이어**
   `detect.ts`(PATH 탐지 + 버전) / `spawn.ts`(프로세스 + 스트리밍) /
   `acp-client.ts`(Agent Client Protocol) / `defs/*.ts`(선언형 정의 15개).
   떼어내 다른 프로젝트에 넣으면 즉시 "멀티 AI 지원"이 된다.

2. **5단계 대화 상태머신 (`detectPhase`)**
   의도 판정에 화이트리스트 정규식을 쓰지 말고 **기본값을 반전**시킨다 —
   기본은 대화/재생성, 명시적 소수 케이스(pin된 프레임)만 단일 편집.

3. **content-graph = 구조화된 중간 표현(IR)**
   최종 산출물을 바로 만들게 하지 않고 구조를 먼저 출력시킨 뒤 검증·정렬한다.

4. **블록 기반 출력 프로토콜**
   ` ```json#content-graph ` + ` ```html#<nodeId> ` — 펜스 코드블록에 식별자를 붙여
   하나의 응답에서 여러 산출물을 안전하게 추출한다. JSON mode 없이 구조화 출력 가능.

5. **어댑터 패턴 + capability 선언** (`weaknesses` 포함)

**실전 교훈 (CLAUDE.md 기록)**

- "타입체크 통과 + 로직이 맞아 보임"을 신뢰하지 말고 반드시 엔드투엔드 실행으로 검증한다.
- ffmpeg **concat demuxer + 재인코딩**은 timebase/PTS가 다른 구간을 섞으면 타임스탬프를
  잘못 누적한다(8초 → 35초). 혼합 출력은 **concat filter**를 써서 타임라인을 재구성해야 한다.
  단일 엔진은 demuxer `-c copy`가 빠르고 무손실이다.
- 폰트 FOUT은 export한 MP4에서만 재현된다(iframe 프리뷰는 폰트 캐시 때문에 안 보임).
  해결: `addInitScript`로 `animation-play-state: paused`를 문서 파싱 전에 주입 →
  stylesheet + 각 face `fonts.load()` + `fonts.ready` 대기 → 해제하고 동결 구간을 lead-in으로 잘라낸다.
- PR에 커밋을 추가하기 전에 PR이 아직 OPEN인지 확인한다 (merge된 뒤 추가하면 커밋이 유실된다).

### Q5. React / PHP로 만들 수 있는가

**React — 이미 사용 중이며 확장을 권한다.**

현재 상태: `adapter-remotion`(Remotion = React 기반 렌더러),
`templates/frame-data-rollup/DataRollup.tsx`(`spring()`, `interpolate()` 사용),
`studio-next`(React 19 + Vite + Tailwind + Zustand, 실험 중).
메인 스튜디오 UI는 바닐라 JS 3,257줄이다.

| 할 일 | 난이도 | 가치 |
|---|:---:|:---:|
| 스튜디오 UI를 React로 재작성 | 중 | 최상 |
| React/Remotion 템플릿 추가 | 하 | 최상 |
| Next.js 템플릿 갤러리 (쇼케이스) | 하 | 상 |
| `@html-video/react` SDK (훅/컴포넌트) | 중 | 상 |
| Electron/Tauri 데스크톱 앱 | 상 | 중 |

```jsx
export const MyFrame = ({ data, width, height }) => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();
  const h = spring({ frame, fps, config: { damping: 200 } });
  const n = interpolate(frame, [0, 60], [0, data.value], { extrapolateRight: 'clamp' });
  return (
    <div style={{ width, height }}>
      <div style={{ height: `${h * 100}%`, background: '#3ce6ac' }} />
      <h1>{Math.round(n).toLocaleString()}</h1>
    </div>
  );
};
```

**PHP — 전체 재작성은 비권장, 특정 레이어는 적합하다.**

불가 사유: Playwright/Puppeteer는 Node 전용, Remotion은 React/Node 필수,
장시간 프로세스 스트리밍 제어가 약함, 영상 툴체인 생태계가 Node 중심.

적합한 역할:

- Laravel/PHP 백엔드가 html-video CLI를 호출하는 래퍼 (Queue + Job)
- 워드프레스 플러그인 (글 → 홍보 영상 자동 생성)
- 템플릿 갤러리 / 관리자 패널
- 결제·유저 관리 등 SaaS 비즈니스 로직

```php
class RenderVideoJob implements ShouldQueue {
    public function handle() {
        $out = storage_path("videos/{$this->id}.mp4");
        Process::timeout(600)->run([
            'node', base_path('html-video/packages/cli/dist/bin.js'),
            'project-render', $this->id, '--output', $out, '--json',
        ]);
    }
}
```

결론: 영상 엔진은 Node에 두고 PHP는 지휘 + 비즈니스 레이어로 쓴다.

### Q6. 유튜브 강의 영상으로 제작 가능한가

가능하며 소재가 좋다.

좋은 이유: ① "영상 만드는 툴 강의를 그 툴로 만든 영상으로" 라는 메타 구성이 성립한다
② 시각적 결과물이 즉시 나온다 ③ 한국어 콘텐츠가 없다 ④ AI·자동화·영상 3대 키워드 교집합
⑤ 난이도 스펙트럼이 입문(설치)부터 고급(어댑터 구현)까지 넓다.

**추천 시리즈 12편**

| # | 제목 | 길이 | 난이도 |
|---|---|---|:---:|
| 1 | "AI가 HTML로 영상을 만든다" 10분 첫 영상 | 10분 | 1 |
| 2 | 설치 완전정복 — Node·pnpm·ffmpeg·Chromium | 15분 | 1 |
| 3 | 템플릿 23종 리뷰 + 선택 기준 | 20분 | 1 |
| 4 | 기사 링크 하나로 해설 영상 만들기 | 12분 | 2 |
| 5 | GitHub 레포 → 프로젝트 소개 영상 | 12분 | 2 |
| 6 | AI 음악 + 나레이션 붙이기 (MiniMax) | 15분 | 2 |
| 7 | CLI 자동화 — 영상 100개 자동 생산 | 20분 | 3 |
| 8 | 내 템플릿 만들기 (매니페스트 + SKILL.md) | 25분 | 3 |
| 9 | 코드 해부 ① 에이전트 런타임 15개 추상화 | 25분 | 4 |
| 10 | 코드 해부 ② content-graph & 위상정렬 | 20분 | 4 |
| 11 | 엔진 어댑터 직접 구현 | 30분 | 5 |
| 12 | React/Next.js 앱에 통합 | 25분 | 4 |

**쇼츠 소재**: 링크 붙여넣고 10초 뒤 영상 완성(타임랩스) / 무료 vs 유료 영상툴 비교 /
AI 15개 중 누가 영상을 제일 잘 만드나 / ffmpeg PTS 버그로 8초가 35초 된 사연.

**제작 체크리스트**

| 항목 | 주의 |
|---|---|
| 라이선스 | Apache-2.0 — 강의 제작·수익화 자유, 단 출처 표기 필수 |
| 원본 명시 | `nexu-io/html-video` 크레딧 |
| API 키 | 화면 녹화 시 절대 노출 금지 |
| 상태 명시 | "Hyperframes만 완성, Remotion 부분"을 정확히 고지 |
| 버전 고정 | 커밋 해시 명시 (업데이트로 영상이 무효화되는 것 방지) |
| 중문/개인정보 | `CLAUDE.md`에 중국어 + 로컬 경로가 있으므로 화면 노출 회피 |

---

## 6. 수익화 아이디어

### Tier 1 — 즉시 시작 (저비용·고효과)

**#1. 한국어 생태계 선점 (최우선)**

현황: README가 영문 + 중문만, 한국어 자료 0개.

할 일: `README.ko.md` 작성 후 upstream PR(영구 크레딧) → `CONTRIBUTING.ko.md` →
스튜디오 UI 한국어 i18n(`packages/project-studio/public/i18n.js`) →
템플릿 23개 한국어 SKILL.md → 블로그 시리즈로 검색 선점.

수익: 블로그 광고·제휴 월 10~50만 / "한국 유일 전문가" 포지션 → 강의·컨설팅 건당 50~300만 /
upstream 컨트리뷰터 이력 → 프리랜스 단가 상승.
투자 2~3주, 비용 0원, 난이도 하.

**#2. MCP 서버 래퍼 (`html-video-mcp`) — 기술적 최고 가치**

```typescript
tools: [
  { name: "search_video_templates", desc: "의도로 템플릿 검색" },
  { name: "create_video_project",   desc: "프로젝트 생성" },
  { name: "render_video",           desc: "MP4 렌더링" },
  { name: "url_to_video",           desc: "링크 → 영상" },
]
```

CLI가 이미 `--json` 기본 + `--stream-progress` NDJSON이므로 MCP 매핑이 거의 1:1이다.
로드맵의 "Agent skill packages"와 방향이 일치한다.
수익: 오픈소스 명성 → 컨설팅/스폰서십/채용. 투자 1~2주, 난이도 중상.

**#3. 프리미엄 템플릿 팩 판매**

현황: 23개가 전부 서양/중국 스타일, K-스타일 없음.

| 팩 | 내용 | 가격 |
|---|---|---|
| K-Content | 예능 자막, 썸네일 인트로, 쇼츠 10종 | $49 |
| K-Finance | 주식/코인 차트, 수익률 카드, 뉴스 티커 | $79 |
| K-Commerce | 쿠팡/스마트스토어 상품 홍보 | $59 |
| K-Edu | 강의 인트로, 퀴즈, 단어 카드 | $39 |
| K-Corporate | 사내 공지, IR, 채용 공고 | $99 |

채널: Gumroad / Lemon Squeezy / 크몽 / 자체 사이트.
Apache-2.0은 파생물 유료 판매를 허용한다(자체 디자인이면 완전 자유).
예상 월 20~100만, 난이도 중.

### Tier 2 — 서비스화 (규모 큼)

**#4. 영상 자동 생산 대행**

타겟: 쇼핑몰 / 부동산 / 병원 / 학원 / 유튜버.
흐름: 크몽·숨고 주문 → CSV·링크 입력 → CLI 배치 → MP4 10~50개 자동 생성 → 납품.
가격 편당 2~5만원 또는 월 정액 30~150만원. 로컬 렌더라 원가가 거의 0이므로 마진 90% 이상.
경쟁사는 사람이 편집하지만 이쪽은 자동화로 훨씬 빠르다. 예상 월 50~300만, 난이도 중상.

**#5. SaaS 래핑 (클라우드 렌더)**

```
브라우저 → Next.js 프론트 → API → 큐(BullMQ) → 워커(html-video CLI) × N
                                                      ↓
                                            S3/R2 → CDN 다운로드
```

| 플랜 | 가격 | 영상 수 |
|---|---|---|
| Free | 0원 | 월 3개 (워터마크) |
| Starter | $19/월 | 월 30개 |
| Pro | $49/월 | 월 150개 + API |
| Team | $149/월 | 무제한 + 브랜드 템플릿 |

Apache-2.0은 SaaS 제공을 허용한다(NOTICE 유지 필요).
주의: Remotion은 4인 이상 조직에서 유료이므로 Hyperframes 엔진만 쓰면 회피 가능.
예상 6개월 내 MRR $1,000~5,000, 난이도 최상, 투자 2~3개월 + 서버비.

**#6. 워드프레스 플러그인 (PHP) — 틈새**

블로그 글 발행 → 자동으로 홍보 영상 생성 → SNS 자동 포스팅.
Freemium(무료 3개/월) → Pro $79/년. 워드프레스는 전 세계 웹사이트의 약 43%를 차지한다.
예상 연 $5,000~30,000, 난이도 상.

### Tier 3 — 콘텐츠·교육 (안정적)

**#7. 유튜브 + 온라인 강의**

| 채널 | 수익 |
|---|---|
| 유튜브 광고 (구독 1만 목표) | 월 30~100만 |
| 인프런/클래스101 강의 | 1회 제작 → 월 50~200만 |
| 멤버십 (템플릿 선공개) | 월 20~50만 |
| 기업 출강 워크샵 | 회당 100~300만 |

번들 전략: 유튜브(무료 유입) → 강의(유료) → 템플릿팩(업셀) → 컨설팅(고가).

**#8. 뉴스레터 / 유료 커뮤니티**

"AI 영상 자동화 위클리" 무료 구독자 확보 → 스폰서십.
유료 디스코드(월 9,900원): 템플릿 선공개 + 코드 리뷰 + Q&A. 예상 월 20~80만, 난이도 중.

### 추천 실행 순서

```
1~3주   ① 한국어 로컬라이제이션 + upstream PR   (신뢰 자산)
          └ 동시에 블로그 시리즈 (SEO 선점)
4~6주   ② MCP 서버 래퍼 공개                   (기술 명성)
7~10주  ③ K-스타일 템플릿 팩 1호 출시           (첫 현금)
동시     ④ 유튜브 시리즈 연재                   (유입 깔때기)
3개월+  ⑤ 대행 서비스 또는 SaaS                 (규모 확장)
```

### 법적 체크리스트

| 항목 | 내용 |
|---|---|
| Apache-2.0 | 상업 이용·수정·재배포·SaaS 전부 허용 |
| NOTICE/LICENSE 유지 | 파생물에 원본 라이선스 + 저작권 고지 필수 |
| 상표 | "html-video" / "nexu-io" 이름으로 오인될 마케팅 금지 → 자체 브랜드 사용 |
| Remotion 라이선스 | 4인 이상 조직은 유료 → Hyperframes 엔진만 쓰면 안전 |
| 템플릿 출처 | `ATTRIBUTIONS.md`의 3층 표기(origin / via_skill / transformation) 승계 |
| 폰트 | Google Fonts 등 웹폰트 라이선스 별도 확인 |

---

## 7. 솔직한 한계 (분석 시점 기준)

- Hyperframes 엔진만 완성. Remotion은 부분, Motion Canvas/Revideo/Manim은 계획뿐이다.
- `ffmpeg` + `chromium`을 별도로 설치해야 하고, `pnpm install` 없이는 아무것도 동작하지 않는다
  (분석 시점에 `node_modules`가 없는 상태였다).
- 스튜디오 서버가 3,472줄 단일 파일이라 유지보수 난이도가 높다.
- `CLAUDE.md`가 내부 노트라 중국어와 개인 정보(개발자 이름, 로컬 경로)가 그대로 노출되어 있다.
- 템플릿 23개가 전부 서양/중국 디자인 계열이라 한국 시장용 자산이 없다.

---

## 참고 링크

- 이 저장소(fork): <https://github.com/bmshin94/html-video>
- 원본(upstream): <https://github.com/nexu-io/html-video>
- Hyperframes (엔진): <https://github.com/heygen-com/hyperframes>
- Remotion: <https://www.remotion.dev/>
- Motion Canvas: <https://github.com/motion-canvas/motion-canvas>
- Revideo: <https://github.com/redotvideo/revideo>
- Open Design (자매 프로젝트): <https://github.com/nexu-io/open-design>
- HTML Anything (자매 프로젝트): <https://github.com/nexu-io/html-anything>
