# 03. Evals — 평가 및 개선 프롬프트

> 목적: 02번이 만든 프롬프트(또는 사람이 쓴 프롬프트)를 **객관적으로 채점하고 자동 개선**하는 평가 프롬프트.
> 핵심 규칙: 최고점 대비 **−5점 이내 도달이 목표**, **최소 3회 평가** 후에야 합격 가능, **최대 10회 개선 반복**, 10회에도 미달이면 **원인 분석 + 해결책 강제 출력**.

---

## 한 줄 요약

이미지 프롬프트 전용으로 **새로 정의한 7개 평가축**을 1~5점으로 채점합니다(가중 합산, 만점 35점). 채점 전 반드시 근거를 먼저 쓰게 하고(chain-of-thought), 합격선은 `만점 − 5 = 30점` 이상으로 두되 **3회 이상 평가**를 통과해야 합니다. 개선은 평가→수정→재평가 루프로 최대 10회 돌고, 그래도 30점을 못 넘기면 루프를 멈추고 "왜 안 되는가 + 어떻게 해결하는가"를 구조화해 보고합니다.

---

## 1. 평가 방법론 배경 (왜 이렇게 설계했나)

- **LLM-as-a-judge**는 사람 평가(비싸다)와 자동 지표(얕다) 사이의 확장 가능한 중간 지점입니다. 2025~2026 베스트 프랙티스의 합의:
  - **분석형 루브릭**(항목별 개별 채점)을 쓴다 — 어디서 왜 실패했는지 드러나 원인 추적이 쉽다.
  - **1~5점 + 단계별 앵커 설명**, 채점 전 **근거 서술 먼저**, **temperature 0**.
  - 네 가지 판정 편향을 설계 단계에서 막는다: **위치(position)·장황함(verbosity)·자기선호(self-preference)·표면유창성(authority)**. → 순서 무작위화, 명확한 루브릭, 가능하면 다중 심사.
  - 소규모 **정답셋(golden set)** 으로 보정하고 주기적으로 점검해 드리프트를 막는다.
- **반복 개선 루프**는 검증된 계보 위에 있습니다:
  - **Self-Refine** (Madaan et al., NeurIPS 2023, arXiv:2303.17651): 한 모델이 생성→비평→수정을 반복. 정지 조건이 "더 개선할 게 없거나 최대 반복 도달" — 우리의 3회 최소/10회 최대 설계와 동일.
  - **Reflexion** (Shinn et al., NeurIPS 2023, arXiv:2303.11366): Actor·Evaluator·Self-Reflection으로 분리하고 언어적 피드백을 **에피소드 메모리**에 누적. → 10회 실패 시 이 누적 메모리가 원인 보고서의 재료가 됨.
  - **APE / OPRO / DSPy**: 프롬프트를 자동 최적화하는 산업·학계 표준 (각각 arXiv:2211.01910 / 2309.03409 / Stanford DSPy).
  - **Anthropic Evaluator-Optimizer 워크플로우**: "한 LLM이 응답을 만들고 다른 LLM이 평가·피드백을 루프로 제공" — 명확한 평가 기준이 있고 반복 개선이 측정 가능한 가치를 줄 때 효과적.

---

## 2. 새로 정의한 7개 평가축

> 기존 범용 7축(Clarity/Specificity/Structure…)을 재사용하지 않고, **이 워크플로우(플라스틱 3D 아이콘 · Nano Banana 2 · 세트 일관성)에 맞춰 새로 정의**했습니다.

| # | 평가축 | 보는 것 | 가중치 |
|---|---|---|---|
| 1 | **주제 정합성** (Subject Fidelity) | Subject가 단수·명확하고, 아이콘으로 즉시 인식되는가 | 15% |
| 2 | **스타일 고정성** (Style Lock Integrity) | 잠긴 플라스틱 3D 스타일 블록이 글자 그대로 보존됐는가 | 20% |
| 3 | **슬롯 완전성** (Slot Completeness) | 7개 슬롯이 모두 있고 올바른 자리에 채워졌는가 | 15% |
| 4 | **제약 준수** (Constraint Compliance) | 긍정 표현·1:1·plain bg·no text·단일 객체가 지켜졌는가 | 15% |
| 5 | **세트 재현성** (Set Reproducibility) | 주제만 바꿔도 같은 세트로 보일 만큼 앵글·조명·팔레트가 고정됐는가 | 15% |
| 6 | **모델 적합성** (Model Fit) | 키워드 나열이 아닌 서술형이고, 모순이 없으며, Nano Banana 강점을 활용하는가 | 10% |
| 7 | **재질 구체성** (Material Specificity) | 막연한 형용사 대신 구체적 재질·마감·색이 들어갔는가 | 10% |

### 점수 앵커 (각 축 공통)

- **5** = 완벽. 흠잡을 데 없음
- **4** = 양호. 사소한 개선 여지
- **3** = 보통. 명확한 결함 1개
- **2** = 미흡. 결함 다수 또는 핵심 누락
- **1** = 실패. 해당 축이 사실상 충족되지 않음

### 점수 계산

```
가중점수 = Σ (각 축 점수 × 가중치)         # 1~5 스케일 유지
환산점수(35점 만점) = 가중점수 × 7         # 보고용
합격선 = 만점 − 5 = 30점 이상 (≈ 가중평균 4.3 이상)
```

---

## 3. 평가·개선 프롬프트 (복사해서 사용)

```text
# ROLE
You are a strict evaluator-and-optimizer for Nano Banana 2 image prompts that
follow the fixed plastic-3D-icon 7-slot structure. You score a prompt, then
improve it in a loop until it is near-perfect or until you must stop and explain
why.

# INPUT
- TARGET_PROMPT: the image prompt to evaluate (7-slot format).
- LOCKED_STYLE: the canonical locked style block (for comparing slot integrity).

# 7 EVALUATION AXES (score each 1-5, with the anchors below)
1. Subject Fidelity     (weight 0.15) - subject singular, concrete, instantly readable as an icon
2. Style Lock Integrity (weight 0.20) - locked plastic-3D style block preserved verbatim
3. Slot Completeness    (weight 0.15) - all 7 slots present and correctly placed
4. Constraint Compliance(weight 0.15) - positive phrasing, 1:1, plain bg, no text, single object
5. Set Reproducibility  (weight 0.15) - camera angle / lighting / palette locked so it stays set-consistent
6. Model Fit            (weight 0.10) - narrative (not keyword salad), no contradictions, plays to Nano Banana strengths
7. Material Specificity (weight 0.10) - concrete material/finish/color, not vague adjectives

Anchors: 5 perfect / 4 minor issue / 3 one clear flaw / 2 multiple flaws / 1 fails.

# SCORING
weighted = sum(axis_score * weight)          # 1-5 scale
scaled35 = round(weighted * 7, 1)            # report out of 35
PASS_THRESHOLD = 30   (i.e. max 35 minus 5)

# LOOP RULES (strict)
- MIN_EVAL_ROUNDS = 3   : never accept before at least 3 full evaluation rounds,
  even if the score already clears 30. (Guards against a lucky first draft.)
- MAX_ITERATIONS  = 10  : at most 10 improve-then-reevaluate cycles.
- On each round: (a) reason BEFORE scoring, per axis; (b) give scaled35;
  (c) if not accepted, list the 1-3 weakest axes and REWRITE the prompt to fix
  exactly those, keeping the locked style block verbatim; (d) carry a short
  "reflection memory" noting recurring failure patterns across rounds.
- ACCEPT only when (rounds >= 3) AND (scaled35 >= 30).
- If iteration 10 finishes still below 30 -> STOP and output the FAILURE REPORT.

# OUTPUT PER ROUND
Round N
  - per-axis reasoning + score (1-5)
  - scaled35: XX.X / 35
  - verdict: CONTINUE or ACCEPT
  - if CONTINUE: weakest axes + the rewritten prompt
  - reflection memory: <recurring issues so far>

# ON ACCEPT
Output:
  - FINAL PROMPT (7-slot SLOTS + woven PROMPT paragraph)
  - final scaled35 and per-axis scores
  - rounds used

# ON FAILURE (10 iterations, still < 30)
Output a structured FAILURE REPORT:
  1. Final score and per-axis scores
  2. Root cause: which axes never reached 4+, and the recurring pattern from
     reflection memory (e.g. "Subject too abstract to render as a single icon",
     "Style lock keeps drifting because the subject implies a non-plastic material")
  3. Why iteration could not fix it (structural reason, not just symptoms)
  4. 2-3 concrete solutions (e.g. "split into two simpler subjects",
     "switch material preset to glass for transparent subjects",
     "relax the 1:1 constraint to 4:5 for tall subjects")
  5. Recommendation: retry with which change, or escalate to a human.

# STYLE
Be terse and honest. Do not inflate scores. Randomize nothing about the locked
style — it must stay identical. Temperature 0.
```

---

## 4. 운영 팁

- **golden set 보정:** 사람이 직접 채점한 프롬프트 8~10개를 만들어 두고, 이 평가기가 비슷하게 매기는지 가끔 대조하세요. 너무 후하면 앵커 설명을 더 빡빡하게 조정합니다.
- **편향 차단:** 같은 프롬프트를 두 번 평가해 점수가 출렁이면 temperature가 0이 아니거나 루브릭이 모호한 신호입니다.
- **"−5점 목표"의 의미:** 만점(35)에 집착하면 루프가 무한정 돌 수 있습니다. `30점(만점−5)` 합격선은 "실무에서 충분히 좋은" 지점을 노린 의도적 설계입니다.
- **이미지까지 평가하려면(권장 확장):** 생성된 *이미지*를 다시 멀티모달 심사(Gemini/Claude)에 넣어 7축으로 채점하면, 프롬프트가 아니라 **결과물 기준의 진짜 evaluator-optimizer 루프**가 됩니다. → 자세한 내용은 `04`와 전체 가이드의 "더 나은 파이프라인" 참고.

---

## 참고

- Madaan et al., *Self-Refine: Iterative Refinement with Self-Feedback*, NeurIPS 2023 — arXiv:2303.17651
- Shinn et al., *Reflexion: Language Agents with Verbal Reinforcement Learning*, NeurIPS 2023 — arXiv:2303.11366
- Zhou et al., *Large Language Models Are Human-Level Prompt Engineers* (APE) — arXiv:2211.01910
- Yang et al., *Large Language Models as Optimizers* (OPRO), Google DeepMind — arXiv:2309.03409
- Anthropic, *Building Effective Agents* — Evaluator-Optimizer 워크플로우

*다음 문서: `04_생성_자동화.md` — Claude in Chrome(Cowork) + Gemini 자동화 및 오케스트레이션 판단*
