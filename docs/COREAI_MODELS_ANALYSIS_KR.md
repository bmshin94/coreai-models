# Core AI Models 전수조사 & 활용 가이드 (한국어)

> 이 문서는 `bmshin94/coreai-models` 레포지토리를 전수조사하고,
> 설치/사용법 · 성격(플러그인/스킬/MCP) · 토큰 필요성 · 에이전트 활용 ·
> 웹(React/PHP) 연동 · 콘텐츠 제작 · 수익화 아이디어까지 정리한 기록입니다.
>
> - 작성 도구: Claude Code (페르소나: 카리나)
> - 작성일: 2026-10-01
> - 작업 브랜치: `claude/loving-pasteur-v7heej`

## 📌 깃허브 주소

| 구분 | URL |
| --- | --- |
| 내 포크 (작업 대상) | https://github.com/bmshin94/coreai-models |
| 원본 (Apple 공식) | https://github.com/apple/coreai-models |
| Core AI 프레임워크 문서 | https://developer.apple.com/documentation/coreai |
| coreai-torch 문서 | https://apple.github.io/coreai-torch/index.html |
| coreai-optimization 문서 | https://apple.github.io/coreai-optimization/introduction/how_to_use_coreaiopt.html |
| coreai-build (AOT 컴파일) | https://developer.apple.com/documentation/coreai/compiling-core-ai-models-ahead-of-time |

---

## 1. 이게 뭐하는 레포인가

**애플이 공식 배포한 "Core AI" 온디바이스 AI 개발 키트.**
PyTorch / HuggingFace 모델을 아이폰·맥에서 인터넷 없이 실행 가능한
`.aimodel` 포맷으로 변환(export)하고, 그것을 Swift 앱에 통합해
실행하기까지의 레시피 + 라이브러리 + 에이전트 스킬 모음집이다.

### 기본 정보

| 항목 | 내용 |
| --- | --- |
| 원본 | `apple/coreai-models` |
| 저작권 | Copyright 2026 Apple Inc. |
| 라이선스 | **BSD 3-Clause** (상업적 사용·수정·재배포 허용) |
| 규모 | Swift 244개 / Python 200개 / 문서 44개, 약 8.5MB |
| 요구사항 | **macOS·iOS 27.0+**, **Xcode 27.0+**, Apple Silicon |
| 기여 정책 | **PR 받지 않음** (Issue만 접수) → 포크 활용이 정석 |
| CI | `if: github.repository == 'apple/coreai-models'` → 포크에서는 자동 비활성 |

### 폴더 4개 = 기능 4개

```
models/  → "레시피"  : 어떤 모델을 어떻게 변환할지 (30종)
python/  → "주방"    : 실제 변환 도구 + 저작용 primitives
swift/   → "식탁"    : 변환 결과물을 앱에서 실행하는 런타임
skills/  → "AI조수" : 코딩 에이전트가 읽는 매뉴얼 (플러그인)
```

#### ① `models/` — 모델 카탈로그 30종

| 분류 | 모델 |
| --- | --- |
| LLM (9) | Gemma 3, GPT-OSS, Mistral, Mixtral, Muse Glimmer, Phi, Qwen2.5, Qwen3, Qwen3-MoE |
| 이미지·영상 생성 | Stable Diffusion 1.5/2.1/3.5, FLUX.2, Wan |
| VLM | Qwen3-VL |
| 비전 (7) | CLIP, Depth Anything v3, EDSR, EfficientSAM, PVT v2, SAM 3 / SAM3-Video, YOLOS |
| 오디오 (4) | CLAP, Parakeet TDT, Wav2Vec 2.0, Whisper |
| 텍스트 (2) | RoBERTa, T5 |

#### ② `python/` — 변환 엔진 (`coreai_models`)

CLI 엔트리포인트 5개:

```bash
uv run coreai.model.registry --list-models     # 지원 모델 조회 (프리셋 41개)
uv run coreai.llm.export <model>               # LLM 변환
uv run coreai.llm.eval                         # 품질 평가
uv run coreai.vlm.export <model>               # 비전+언어 모델 변환
uv run coreai.diffusion.export <model>         # 디퓨전 변환
```

핵심 구조 — **primitives가 타겟별로 분리**되어 있다:

| | `primitives/macos/` | `primitives/ios/` |
| --- | --- | --- |
| 타겟 | **GPU** (성능 우선) | **Neural Engine** (전력 효율 우선) |
| 구성 | sdpa, rope, rms_norm, cache, cache_scatter, mlp, switch(MoE) | sdpa, bidirectional_sdpa, rope, rms_norm, cache, mlp, gelu, layer_norm, embedding, quantization |

기타: `export/` (ios.py, macos.py, presets.py, compression.py, externalize.py,
mlir_ops.py, bundle.py, compiler.py, metadata.py, pipeline.py),
`model_registry.py` (ModelPreset 41개).

주요 핀 버전: `torch==2.9.0`, `coreai-torch==0.4.2`, `coreai-opt==0.2.1`,
`coreai-core==1.0.0b2`, `transformers>=5.5`.

#### ③ `swift/` — 런타임 + CLI 도구

라이브러리 6개: `CoreAILM`, `CoreAIDiffusion`, `CoreAIVideoDiffusion`,
`CoreAISegmentation`, `CoreAISpeech`, `CoreAIObjectDetection`

CLI 도구 8개:

| 도구 | 역할 |
| --- | --- |
| `llm-runner` | 터미널 LLM 대화 |
| **`llm-server`** | **OpenAI 호환 HTTP 서버** |
| `llm-benchmark` | 속도 측정 (prompt 512 / gen 1024 / 5 trials) |
| `diffusion-runner`, `videodiffusion-runner` | 이미지·영상 생성 |
| `image-segmenter`, `object-detector` | 분할 / 객체 탐지 |
| `speech-recognizer` | 음성 인식 |

`llm-server` 라우트 (`swift/Sources/Tools/llm-server/ChatHandler.swift`):

```
GET  /health      GET  /ready       GET  /v1/stats
GET  /v1/models   POST /v1/chat/completions   POST /v1/completions   POST /v1
```

추가 기능: 스트리밍(SSE), `supportsToolCalling`, `supportsLogprobs`,
`GuidedGeneration/` (XGrammar 기반 제약 디코딩 = JSON 스키마 강제 출력).

앱 통합은 `FoundationModels` 프레임워크와 결합되어 3줄로 끝난다:

```swift
import FoundationModels
import CoreAILanguageModels

let model = try await CoreAILanguageModel(resourcesAt: modelURL)
let session = LanguageModelSession(model: model)
let response = try await session.respond(to: "What is quantum computing?")
```

#### ④ `skills/` — 코딩 에이전트 플러그인

| 스킬 | 내용 |
| --- | --- |
| `working-with-coreai` | export → compile → run 전체 워크플로우 |
| `model-authoring` | NE/GPU별 저작 규칙 (BC1S 레이아웃, KV캐시 패턴, PSNR 검증) |
| `model-compression-exploration` | 양자화/팔레타이제이션 스윕 + 리포트 (`compression_metrics.py`, `quality_metrics.py`) |

매니페스트가 3종 에이전트를 모두 지원: `.claude-plugin/`, `.codex-plugin/`,
`gemini-extension.json`.

### 압축(Compression) 프리셋

| 플랫폼 | 프리셋 | 설명 |
| --- | --- | --- |
| macOS | `4bit` (기본) | INT4 weight-only, block size 32 |
| macOS | `4bit_weights_8bit_kv_cache` | INT4 + INT8 per-tensor KV cache |
| macOS | `none` | 전정밀도 |
| iOS | `4bit_weight_palettized_group32` (기본) | 4-bit 팔레타이제이션, 채널 그룹 32 |
| iOS | `4bit_weight_palettized_group8` | 채널 그룹 8 |
| iOS | `none` | 전정밀도 |

- iOS 팔레타이제이션 프리셋은 Embedding을 기본 8-bit per-tensor로 양자화
- KV 캐시 양자화는 `coreai-opt` graph 실행 모드 필요
- 커스텀 레시피는 `--compression-config <yaml>` 로 지정

### 컨텍스트 길이

- macOS: 동적 KV 캐시 → 생략 가능 (모델 최대값)
- iOS: **정적 shape이므로 `--max-context-length` 필수**

---

## 2. 쉬운 비유 정리

- **"AI 모델 수입·통관 업체"**: 해외 공장(HuggingFace)의 좋은 물건(모델)을
  한국 콘센트(애플 기기)에 맞게 변압(변환)해서 설치해주는 곳.
- **양자화**: 숫자를 대충 적기. `3.14159265` → `3.14`
- **팔레타이제이션**: 색연필 16색으로 그림 그리기. 비슷한 값을 묶어 번호만 저장.
- 결과: `16GB → 4GB` 수준으로 줄어들어 아이폰에 탑재 가능.
- **칩 안의 일꾼 3명**: CPU(만능·느림) / GPU(힘셈·배터리 소모) /
  Neural Engine(AI 전용·초저전력, 단 BC1S 등 특정 데이터 모양 요구).
  → 그래서 primitives가 iOS용 / macOS용으로 분리됨.

### 전체 흐름 5단계

```
1. 모델 고르기   uv run coreai.model.registry --list-models
2. 변환하기      uv run coreai.llm.export Qwen/Qwen3-0.6B --platform iOS
3. 압축 튜닝     model-compression-exploration 스킬로 최적점 탐색
4. 테스트        swift run llm-runner --model ./model --prompt "안녕"
5. 앱에 심기     CoreAILanguageModel(resourcesAt:)
```

---

## 3. 질문별 답변

### Q1. 설치 및 사용법

준비물: Apple Silicon 맥 / macOS 27.0+ / Xcode 27.0+ / `uv` / 디스크 50GB+

```bash
# 설치
brew install uv            # 또는 curl -LsSf https://astral.sh/uv/install.sh | sh
git clone https://github.com/bmshin94/coreai-models.git
cd coreai-models
# 의존성은 uv run 이 pyproject.toml + uv.lock 기준으로 자동 처리

# 지원 모델 확인
uv run coreai.model.registry --list-models
uv run coreai.model.registry --list-models --type llm --platform macOS
uv run coreai.model.registry --list-models --type diffusion

# 변환
uv run coreai.llm.export Qwen/Qwen3-0.6B                                   # macOS 기본
uv run coreai.llm.export Qwen/Qwen3-0.6B --platform iOS --max-context-length 4096
uv run coreai.llm.export Qwen/Qwen3-4B --compression 4bit_weights_8bit_kv_cache
uv run coreai.vlm.export qwen3-vl
uv run coreai.diffusion.export stabilityai/stable-diffusion-3.5-medium
uv run models/whisper/export.py                                            # 개별 레시피

# 유용한 옵션
--dry-run            # 설정만 미리보기 (추천)
--num-layers 1       # 레이어 1개만 (디버깅)
--compression none   # 무압축
--output-dir ./out/
--include-debug-info # 디버그 정보 포함
-v                   # 로그 상세

# 실행 / 측정 / 서버
swift run -c release llm-runner --model ./model --prompt "Hello"
swift run -c release llm-benchmark --model ./model
swift run -c release llm-server --model ./model --port 8080
```

스킬(플러그인) 설치:

```bash
# Claude Code
/plugin marketplace add git@github.com:bmshin94/coreai-models.git
/plugin install coreai-skills@coreai-models

# Codex CLI
codex plugin marketplace add https://github.com/bmshin94/coreai-models
# 이후 /plugins → coreai-skills → Install

# Gemini CLI
gemini extensions install /path/to/coreai-models/skills
```

Xcode 통합: Package Dependencies에 레포 URL 추가 → 필요한 라이브러리 선택.
큰 모델은 iOS 메모리 한도 초과 가능 →
`com.apple.developer.kernel.increased-memory-limit` entitlement 필요.

### Q2. 플러그인? 스킬? MCP?

**전부 아니고 "오픈소스 SDK/툴킷"이며, 그 중 `skills/`만 플러그인이다.**

| 구분 | 해당 | 설명 |
| --- | --- | --- |
| 플러그인 | 부분 ⭕ | `.claude-plugin/marketplace.json` 의 `coreai-skills` |
| 스킬 | 부분 ⭕ | 그 안의 Agent Skill 3개 |
| MCP | ❌ **없음** | MCP 서버/설정 파일 전무 |

개념 정리:

- **스킬** = AI가 읽는 지침서(SKILL.md). 새 도구가 아니라 "지식".
- **플러그인** = 스킬/커맨드/MCP설정을 묶은 설치 패키지(배포 포장지).
- **MCP** = AI에게 새 도구(함수)를 제공하는 서버 프로토콜. → 이 레포엔 없음.

즉 MCP는 "손"을, 스킬은 "지식"을 준다. 이 레포는 지식만 제공.
**빈 칸 = "Core AI MCP 서버"는 아직 아무도 안 만들었다 (기회).**

### Q3. API 토큰 필요?

**Core AI 자체는 토큰 0개 (완전 로컬·무료).**

| 상황 | 토큰 | 비고 |
| --- | --- | --- |
| 모델 실행(추론) | ❌ | 기기에서 동작, 과금 개념 없음 |
| `.aimodel` 변환 | ❌ | 로컬 작업 |
| 공개 모델 다운로드 | ❌ | Qwen2.5/3, SmolLM2, OLMo2, Whisper, YOLOS, CLIP |
| **게이트 모델 다운로드** | ✅ HF 토큰 | Gemma 3, Mistral/Mixtral, FLUX.2, SD 일부, Llama 계열 |
| 스킬 사용 | ❌ | 문서 읽기뿐 |
| `llm-server` 호출 | ❌ | `api_key`는 아무 문자열 가능 |
| 앱스토어 배포 | 💳 | 애플 개발자 계정 (연 $99) |

게이트 모델 처리 (`python/src/coreai_models/diffusion/pipeline.py`):

```
PermissionError: Access denied: {model_id} is a gated model.
Accept the license at https://huggingface.co/{model_id} and run: hf auth login
```

```bash
hf auth login            # 또는 export HF_TOKEN=hf_xxxx
```

> 첫 연습은 토큰이 필요 없는 **Qwen3-0.6B** 추천.

### Q4. AI 에이전트 구축에 도움이 되는가 → YES

| 에이전트 요소 | 지원 | 근거 |
| --- | --- | --- |
| LLM 추론 | ⭕ | LLM 9종 + MoE + VLM |
| OpenAI 호환 API | ⭕⭕ | `llm-server` `/v1/chat/completions` |
| 스트리밍 | ⭕ | `ChatHandler.swift` |
| Tool Calling | ⭕ | `ServerState.swift` `supportsToolCalling` / `toolCallDetection` |
| JSON 강제 출력 | ⭕⭕ | `GuidedGeneration/` + XGrammar |
| logprobs | ⭕ | `supportsLogprobs` (신뢰도 측정) |
| 멀티모달 | ⭕ | Qwen3-VL |
| 음성 I/O | ⭕ | Whisper, Parakeet |
| 운영 모니터링 | ⭕ | `/v1/stats`, `/health`, `/ready` |

XGrammar의 가치: 토큰 단위로 문법 위반 후보를 차단 → **JSON 파싱 에러가
구조적으로 0%**. 작은 모델(0.6B)로도 안정적인 에이전트 구성 가능.

추천 하이브리드 구조:

```
오케스트레이터(Claude/GPT)   ← 복잡한 추론, 소량 호출
        ↓ 위임
로컬 워커(Core AI llm-server) ← 분류/요약/추출/임베딩/음성, 대량 호출 = 비용 0
```

한계: 애플 하드웨어 전용 / 작은 모델의 추론력 / iOS 고정 컨텍스트 /
macOS·iOS 27+ / 동시 요청 처리량 한계(1~2 사용자 규모).

### Q5. React / PHP로 만들 수 있는가

- **직접 실행은 불가**: `.aimodel`은 Apple Silicon + Core AI 런타임 전용
  (브라우저 JS·PHP·리눅스 불가, Swift/Python 바인딩만 존재).
- **활용은 100% 가능**: `llm-server`를 HTTP 백엔드로 두면 된다.

```
React / PHP  ──HTTP──▶  llm-server (맥, :8080/v1/...)  ──JSON──▶  응답
```

React:

```jsx
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "http://localhost:8080/v1",
  apiKey: "dummy",
  dangerouslyAllowBrowser: true, // 로컬 개발용
});

const stream = await client.chat.completions.create({
  model: "local",
  messages: [{ role: "user", content: prompt }],
  stream: true,
});
for await (const chunk of stream) {
  setText((t) => t + (chunk.choices[0]?.delta?.content ?? ""));
}
```

PHP:

```php
<?php
function askLocalLLM(string $prompt): string {
    $ch = curl_init('http://localhost:8080/v1/chat/completions');
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER     => ['Content-Type: application/json'],
        CURLOPT_POST           => true,
        CURLOPT_POSTFIELDS     => json_encode([
            'model'    => 'local',
            'messages' => [['role' => 'user', 'content' => $prompt]],
        ]),
    ]);
    $res = json_decode(curl_exec($ch), true);
    curl_close($ch);
    return $res['choices'][0]['message']['content'] ?? '';
}
```

보안 주의: `llm-server`에는 인증이 없다. 브라우저에서 직접 호출하지 말고
PHP/Node 프록시에서 인증·레이트리밋을 처리하고, 외부 노출 시 CORS +
HTTPS 리버스프록시(nginx/Caddy)를 둔다.

웹 레이어로 만들 수 있는 것: Core AI 웹 대시보드, 변환 작업 큐 매니저,
압축 실험 비교 뷰어(PSNR vs 용량), 셀프호스팅 챗 UI, 모델 카탈로그 사이트.

### Q6. 유튜브 강의 제작 가능성 → 가능 (블루오션)

- 라이선스 BSD-3 → 코드 설명·화면 노출·수익화 모두 허용 (저작권 고지 유지)
- 주의: "Apple 공식"처럼 오인되게 표기하지 말 것 / 베타 소프트웨어는 NDA 확인 /
  모델별 라이선스(Gemma 등) 영상에서 안내
- 한국어 Core AI 강의가 거의 없음 + 애플 개발자 모수 큼 + 구매력 높은 시청자

추천 커리큘럼 10부작:

| EP | 제목 | 비고 |
| --- | --- | --- |
| 0 | "아이폰이 인터넷 없이 ChatGPT를 돌린다" | 훅/숏츠 |
| 1 | Core AI 전체 지도 + 레포 투어 | 본 문서 기반 |
| 2 | 환경 세팅 (uv/Xcode) + 실패담 | 에러 해결 = 검색 유입 |
| 3 | 첫 변환: Qwen3-0.6B | 성취감 → 구독 전환 |
| 4 | 양자화/팔레타이제이션 완전정복 | 품질 비교 |
| 5 | Neural Engine vs GPU 벤치마크 | 숫자 썸네일 |
| 6 | SwiftUI 챗앱 30분 완성 | 최고 인기 예상 |
| 7 | `llm-server`로 무료 ChatGPT API + React | |
| 8 | Whisper 음성 받아쓰기 앱 | 실용성 |
| 9 | SD / FLUX 온디바이스 이미지 생성 | 비주얼 임팩트 |
| 10 | AI 에이전트 + XGrammar JSON 강제 | 차별화 |

제작 팁: 숫자 썸네일("16GB → 4GB", "토큰비 0원"), 숏츠 분할, 에러 해결 영상,
설명란에 깃허브 링크, 영어 자막, 수익 단계(애드센스 → 멤버십 → 유료강의 → 출강).

---

## 4. 수익화 아이디어 (상세)

### Tier 1 — 즉시 (투자 0원)

| # | 아이디어 | 수익 모델 | 난이도 |
| --- | --- | --- | --- |
| ① | 기술 블로그 / 뉴스레터 (한국어 Core AI 자료 선점) | 애드센스, 제휴, 포트폴리오 | ⭐ |
| ② | 유튜브 강의 (위 커리큘럼) | 광고 + 멤버십 + 강의 유입 | ⭐⭐ |
| ③ | 포크 가치 올리기 | 깃허브 스폰서, 평판 → 수주 | ⭐⭐ |

③ 구체안 — **애플이 PR을 안 받으므로 "생태계 보완 레포"는 포크가 독점 가능**:

```
awesome-coreai/        큐레이션
examples/react-chat/   React 연동 예제 (원본에 없음)
examples/php-proxy/    PHP 프록시 예제 (없음)
docs/ko/               한국어 문서 (없음)
docker/                개발환경 세팅 자동화
mcp-server/            Core AI MCP 서버
```

### Tier 2 — 중기 (1~3개월)

④ **유료 강의 / 전자책**

| 상품 | 가격 |
| --- | --- |
| 전자책 "온디바이스 AI 실전" | 2~3만원 |
| 인프런·유데미 강의 | 8~15만원 (100명 ≈ 1,000만원) |
| 주말 2일 라이브 부트캠프 | 30~50만원 (10명 ≈ 400만원) |

차별화: 공식 문서는 영어 + Swift 중심 → 한국어 + 웹개발자 친화 설명.

⑤ **템플릿 / 보일러플레이트 판매 ($49~99)**

```
"온디바이스 AI 챗앱 스타터킷"
├─ SwiftUI 챗 UI (스트리밍, 마크다운, 히스토리)
├─ 모델 다운로드 매니저 (진행률, 재시도)
├─ 메모리 가드 (entitlement 설정 포함)
├─ React 웹 콘솔
└─ 설치 영상 + 30일 지원
```

판로: Gumroad, LemonSqueezy, CodeCanyon.
가치 근거: 메모리 한도 / 토크나이저 번들링 / iOS 고정 컨텍스트 등 함정 선해결.

⑥ **Core AI MCP 서버 (임팩트 최상)**

```
mcp__coreai__list_models          # 프리셋 41개 조회
mcp__coreai__estimate_size        # 변환 전 용량 예측
mcp__coreai__export               # 변환 실행 (진행률 스트리밍)
mcp__coreai__benchmark            # 속도/메모리 측정
mcp__coreai__compare_compression  # 압축 옵션 비교표
mcp__coreai__run_prompt           # 로컬 추론
```

애플 스킬은 "지식"만 제공 → "실행 도구"가 빈 칸.
오픈소스로 평판 확보 후 Pro(클라우드 변환 큐, 팀 대시보드) 월 $19 모델.

### Tier 3 — 본게임 (3~12개월)

⑦ **앱스토어 앱** — 온디바이스는 "프라이버시"가 곧 마케팅 문구

| 앱 | 모델 | 수익 | 근거 |
| --- | --- | --- | --- |
| 비밀 AI 일기 | Qwen3-1.7B | 월 4,900원 | "서버에 안 올라감" |
| 오프라인 통역기 | Qwen3 + Whisper | 평생 29,000원 | 로밍 없이, 여행객 |
| 회의록 받아쓰기 | Whisper + LLM 요약 | 월 9,900원 | 기밀 회의 |
| 계약서 분석기 | Qwen3-4B | 건당/구독 | 법무·의료 업로드 금지 |
| 사진 정리 AI | CLIP + SAM3 | 9,900원 | 대량 사진 업로드 부담 |
| 오프라인 이미지 생성 | SD3.5 / FLUX.2 | 구독 | 생성 무제한, 서버비 0 |
| 시각보조 앱 | Qwen3-VL | 무료+기부/B2G | 사회적 가치, 피처드 |
| 수능/자격증 과외 | Qwen3-4B | 월 구독 | 교육 시장, 오프라인 학습 |

핵심: 클라우드 AI 앱은 사용자가 늘면 서버비가 폭발하지만,
온디바이스는 **사용자 수와 무관하게 서버비 0원 → 마진 90%+**.

⑧ **B2B 컨설팅 / SI (단가 최고)**

```
타겟: 의료·금융·법률·국방·제조 (데이터 외부 전송 금지 업종)
· 사내 모델 온디바이스 최적화             500만 ~ 3,000만원
· 압축 튜닝 (품질↔용량 최적점 탐색)       300만 ~ 1,000만원
· 맥 스튜디오/미니 기반 사내 AI 서버 구축  1,000만원+
· 기업 교육 (2일)                         300만 ~ 800만원
```

근거: `model-compression-exploration` 스킬 + PSNR 검증 노하우.
영업 루트: 블로그·유튜브로 신뢰 축적 → 인바운드.

⑨ **SaaS "Core AI 변환 서비스"**

```
HF 모델 ID 입력 → 맥 서버가 변환 → .aimodel 다운로드
Free : 0.6B, 월 3회
Pro  : $29/월 — 8B까지, 압축 옵션 전체, 벤치마크 리포트
Team : $99/월 — API, 비공개 모델, 우선순위 큐
```

초기 인프라: 맥미니 M4 Pro 2~3대(약 500만원).
타겟: Apple Silicon 맥이 없거나 변환 환경 세팅이 번거로운 개발자.

### 로드맵

```
1개월  블로그 5편 + 유튜브 EP0~3 + 포크에 한국어문서/React예제   → 스타 50
3개월  유튜브 EP4~10 + MCP 서버 오픈소스 공개                   → 구독 1천, 첫 수익
6개월  전자책/인프런 강의 + 스타터킷 판매                        → 월 100~300만원
12개월 앱스토어 앱 1개 + B2B 컨설팅 1~2건                        → 월 500만원+
```

### 리스크 & 대응

| 리스크 | 대응 |
| --- | --- |
| API 변경 (`coreai-core==1.0.0b2` 등 베타 단계) | 버전 명시, 업데이트 영상을 콘텐츠로 |
| 니치 시장 | 영어 콘텐츠로 확장 |
| 애플이 직접 제공 | 시장 확대 = 콘텐츠 수요 증가 |
| 모델 라이선스(Gemma 등) | 상업 앱은 Apache 2.0 Qwen 계열 권장 |

**현실적인 첫걸음**: ① 블로그 + ② 유튜브로 신뢰 축적 → ⑥ MCP 서버로 기술력 증명
→ ⑦ 앱 또는 ⑧ 컨설팅으로 수익 확대.

---

## 부록 A. 자주 쓰는 명령어 치트시트

```bash
# 조회
uv run coreai.model.registry --list-models [--type llm|diffusion] [--platform macOS|iOS]
uv run coreai.vlm.export --list-models

# 변환
uv run coreai.llm.export <hf_id|short_name> [--platform iOS] [--max-context-length N]
                                           [--compression <preset>|--compression-config <yaml>]
                                           [--dry-run] [--num-layers 1] [--include-debug-info] [-v]
uv run coreai.diffusion.export <hf_id>
uv run coreai.vlm.export qwen3-vl [--skip-vision]
uv run models/<name>/export.py

# 실행
swift run -c release llm-runner  --model <dir> --prompt "..."
swift run -c release llm-server  --model <dir> --port 8080
swift run -c release llm-benchmark --model <dir> [-p 512 -g 1024 -n 5]
swift run -c release speech-recognizer --model <whisper_dir>
swift run -c release diffusion-runner --model <sd_dir> --prompt "..."

# 컴파일 (선택)
xcrun coreai-build compile --help
```

## 부록 B. 체크리스트 (시작 전)

- [ ] Apple Silicon 맥 보유
- [ ] macOS 27.0+ / Xcode 27.0+ 설치
- [ ] `uv` 설치 (`brew install uv`)
- [ ] 디스크 여유 50GB+
- [ ] (게이트 모델 사용 시) `hf auth login`
- [ ] 첫 연습은 `Qwen/Qwen3-0.6B` + `--dry-run`
- [ ] 큰 모델 iOS 탑재 시 increased-memory-limit entitlement

## 부록 C. 참고 경로

| 목적 | 경로 |
| --- | --- |
| 모델 레지스트리 | `python/src/coreai_models/model_registry.py` |
| 압축 프리셋 | `python/src/coreai_models/export/presets.py` |
| iOS 저작 primitives | `python/src/coreai_models/primitives/ios/` |
| macOS 저작 primitives | `python/src/coreai_models/primitives/macos/` |
| OpenAI 호환 서버 | `swift/Sources/Tools/llm-server/ChatHandler.swift` |
| 제약 디코딩 | `swift/Sources/CoreAILanguageModels/GuidedGeneration/` |
| 에이전트 스킬 | `skills/skills/*/SKILL.md` |
| 플러그인 마켓플레이스 | `.claude-plugin/marketplace.json` |
