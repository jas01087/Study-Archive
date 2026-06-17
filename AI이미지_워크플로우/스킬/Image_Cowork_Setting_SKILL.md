---
name: image-cowork-setting
description: 브리프(재질·개수·용도)를 받아 Cowork이 ① 사용자와 채팅으로 아이콘 주제를 상의·확정하고 ② 승인된 주제로 프롬프트를 만든 뒤 ③ 먼저 샘플 3개를 생성해 스타일을 확인받고 ④ OK가 나면 Claude in Chrome으로 Gemini 채팅창을 구동해 나머지를 일괄 생성하고 ⑤ 사용자가 지정한 폴더에 저장·파일명 정리까지 수행하도록 안내하는 절차다. "메탈 아이콘 50개 뽑아줘", "프로스티드 글라스 2레이어로 카드사 아이콘 20개", "제미나이로 3D 아이콘 세트 배치 생성해줘", "OO용 아이콘 세트 만들어줘"처럼 재질·개수·용도를 주며 3D 아이콘을 여러 개 만들려는 요청에는 반드시 이 스킬을 사용한다. 재질과 개수는 자유롭게 지정 가능하다. 아이콘 1개만 원하거나 모션·영상 요청에는 사용하지 않는다.
---

# Image Cowork Setting

브리프를 받아 **주제 상의·확정 → 샘플 3개 검수 → (승인 후) 나머지 일괄 생성 → 사용자 지정 폴더 저장·파일명 정리**까지 수행하는 절차다.

지켜야 할 원칙:

- **재질·개수·용도는 모두 런타임 입력**이다. 특정 재질이나 개수를 미리 가정하지 않는다. "메탈 50개"든 "프로스티드 글라스 2레이어 20개"든 동일하게 처리한다.
- 생성은 **API가 아니라 Claude in Chrome으로 Gemini 채팅창을 조작**해서 한다. 이유는 비용이다 — 채팅창은 구독(정액) 한도 안에서 추가 과금 없이 생성되지만 API는 장당 과금이 붙는다. 사람이 채팅창에서 반복하던 일을 자동화하는 것이지 유료 API로 갈아타는 게 아니다.
- **두 개의 승인 게이트**(주제 확정, 샘플 3개 확인)를 통과하기 전에는 다음 단계로 넘어가지 않는다. 특히 사용자 승인 없이 대량 생성을 시작하지 않는다.

## 입력 (브리프에서 추출. 빠진 필수값은 시작 전에 사용자에게 질문)

- **material (재질)** — 필수. 예: `frosted glass 2-layer`, `plastic`, `clay`, `glossy metal`, `ceramic` 등. 아래 "재질 프리셋"에 있으면 그 블록을 쓰고, 없으면 7-슬롯 패턴대로 합리적인 LOCKED STYLE을 추론해 만든다.
- **count (개수)** — 필수. 임의의 N(예: 20, 50). 가정하지 않는다.
- **domain (용도/맥락)** — 주제 제안에 사용. 예: 카드사, 보험, 핀테크, 헬스케어.
- **out_dir (저장 폴더)** — 필수. **사용자가 지정한 경로**에 저장한다. 지정이 없으면 반드시 물어본다.
- **aspect** — 종횡비. 기본값 `1:1`.

## 1단계 — 주제 상의·확정 (승인 게이트 ①)

주제는 이 절차에 미리 정해져 있지 않다. 매번 domain·count에 맞게 새로 제안하고 사용자와 확정한다.

1. domain·count에 맞춰 아이콘 주제 후보 리스트(번호 매김)를 제안한다. 각 주제는 **단수·구체 객체**, 서로 중복 없이, 그 도메인 사용자가 실제로 떠올릴 만한 개념으로. 추상 개념은 대표 사물로 환원한다(예: "혜택" → 선물상자).
2. 사용자와 상의한다 — 추가/삭제/교체/순서 변경 요청을 반영해 리스트를 다듬는다. 필요하면 몇 차례 왕복한다.
3. 사용자가 **명시적으로 "이대로 진행"이라고 승인할 때까지** 다음 단계로 넘어가지 않는다.

## 2단계 — 승인된 주제로 7-슬롯 프롬프트 생성

승인된 각 주제를 선택된 material의 **LOCKED STYLE**에 끼워 7-슬롯 분해형 + 서술형 합본을 만든다.

- **Subject(+주제 종속 디테일 1줄)만 변하고 나머지 슬롯은 배치 내 전 주제에서 글자 그대로 고정**한다(세트 일관성).
- Subject는 단수·구체. 긍정 표현만(유일한 예외 `no text`). 프롬프트는 **영어**로 작성.
- material 특성에 맞는 accent(예: 색)를 Subject에 은은하게 넣는다.

7-슬롯 템플릿 (material 프리셋이 슬롯을 채움):

```text
--- SLOTS ---
[Subject]      <subject + subtle accent suited to the material>
[Style]        <from material preset>
[Composition]  single object centered, slight 3/4 top-down isometric angle, generous even padding around the object
[Lighting]     <from material preset>
[Rendering]    <from material preset>
[Detail]       <from material preset> (+ one subject-specific detail)
[Constraint]   single object only, plain solid white background, no text, no extra props, {aspect} aspect ratio, centered

--- PROMPT ---
<all slots woven into ONE natural-language English paragraph, in slot order,
ready to paste into the Gemini chat>
```

## 3단계 — 샘플 3개 먼저 생성·검수 (승인 게이트 ②)

대량 생성 전에 **재질·스타일 톤을 먼저 확정**한다. 잘못된 톤으로 50개를 뽑는 사고를 막는 단계다.

1. 승인된 주제 중 **대표 3개**(서로 형태가 다른 것으로 고르면 좋다)를 선택한다.
2. 3단계의 생성 방식(4단계와 동일하게 Claude in Chrome → Gemini 채팅)으로 **샘플 3장만** 생성·다운로드한다.
3. 사용자에게 3장을 보여주고 스타일을 확인받는다 — 색감, 마감(광/매트), 조명, inner core 느낌, 여백 등.
4. 수정 요청이 있으면 **LOCKED STYLE을 조정**하고 샘플 3장을 다시 뽑아 재확인한다(OK 나올 때까지 반복).
5. 사용자가 **샘플을 승인하면** 다음 단계로 진행한다. 승인된 3장은 최종본으로 유지한다(스타일이 바뀌지 않는 한 재생성하지 않는다).

## 4단계 — 나머지 일괄 생성 (Claude in Chrome으로 Gemini 채팅)

샘플이 승인되면 **나머지 (N−3)개**를 같은 LOCKED STYLE로 생성한다.

```text
준비: 이미지 생성이 가능한 Gemini 채팅 탭을 연다. 신뢰 도메인만. 다른 사이트 이동 금지.

For each remaining subtopic:
  1. Gemini 채팅 입력창에 그 주제의 PROMPT 문단을 붙여넣고 전송한다.
  2. 이미지가 실제로 렌더될 때까지 기다린다(완료 확인 — 빈 응답/오류면 실패 처리).
  3. 생성된 이미지를 다운로드한다.
  4. 스타일 표류가 보이면 새 채팅으로 리셋 후 계속.
  5. 실패 시(이미지 없음/다운로드 실패): 백오프 두고 최대 2회 재시도, 그래도 실패면
     로그에 남기고 다음 주제로 넘어간다 — 배치 전체를 멈추지 않는다.
```

## 5단계 — 사용자 지정 폴더에 저장 + 파일명 정리

생성된 모든 이미지(샘플 3개 포함)를 **out_dir(사용자가 지정한 폴더)** 로 옮기고 규칙대로 이름을 바꾼다.

- 네이밍: `YYYYMMDD_{domain-slug}_{subject-slug}_{material-slug}_v###.png`
  - 예) `20260617_creditcard_credit-card_metal_v001.png`
- 영문·숫자·하이픈/언더스코어만. 공백·특수문자 금지. ISO 날짜. 3자리 제로패딩.
- `out_dir/{subject-slug}/` 아래에 주제별로 배치.
- `out_dir/README.txt`에 재질·개수·날짜·주제 리스트·네이밍 규칙을 기록.

```
{out_dir}/
├── README.txt
├── credit-card/
│   └── 20260617_creditcard_credit-card_metal_v001.png
├── reward-points/
│   └── 20260617_creditcard_reward-points_metal_v001.png
└── ...
```

## 재질 프리셋 라이브러리

`Composition`·`Constraint`는 모든 재질 공통(위 템플릿). 재질별로 `Style`·`Lighting`·`Rendering`·`Detail`을 아래로 채운다. 목록에 없는 재질은 같은 형식으로 합리적으로 추론한다.

**frosted glass 2-layer**
- Style: frosted glass 3D icon, two-layer construction — a semi-translucent frosted outer shell over a solid colored inner core, soft satin finish, glassmorphism
- Lighting: soft studio lighting, gentle key from the upper-left, soft ambient fill, a single soft contact shadow beneath, subtle rim light passing through the glass
- Rendering: high-quality Octane/Cinema4D-style PBR render, frosted glass material with subsurface scattering and a soft internal glow, ambient occlusion
- Detail: rounded beveled edges, soft specular highlights on the frosted surface, the colored inner core clearly visible through the translucent outer shell, clean seams

**plastic**
- Style: glossy matte plastic 3D icon, toy-like, soft rounded forms
- Lighting: soft studio lighting, gentle key from the upper-left, soft ambient fill, a single soft contact shadow beneath
- Rendering: high-quality Octane/Cinema4D-style PBR render, smooth plastic material with subtle subsurface softness, ambient occlusion
- Detail: smooth beveled edges, subtle specular highlights, clean seams

**clay**
- Style: soft clay / plasticine 3D icon, matte, hand-molded look
- Lighting: soft diffuse studio lighting, soft ambient fill, a single soft contact shadow beneath
- Rendering: high-quality PBR render, matte clay material, gentle ambient occlusion
- Detail: fingerprint-soft surface, rounded soft edges, no hard speculars

**glossy metal**
- Style: glossy metallic 3D icon, polished
- Lighting: soft studio lighting with crisp controlled highlights, soft contact shadow beneath
- Rendering: high-quality PBR render, polished metal with sharp reflections and anisotropic highlights, ambient occlusion
- Detail: clean machined edges, bright specular highlights, subtle reflection of a neutral studio environment

**ceramic**
- Style: smooth glazed ceramic 3D icon
- Lighting: soft studio lighting, soft ambient fill, a single soft contact shadow beneath
- Rendering: high-quality PBR render, glazed ceramic with a subtle glossy coat, ambient occlusion
- Detail: smooth rounded edges, soft glaze speculars, clean seams

## 안전·비용

- **두 승인 게이트 준수**: 주제 확정(①)과 샘플 3개 확인(②)을 통과하기 전에는 대량 생성을 시작하지 않는다.
- 신뢰 도메인(Gemini)만, "실행 전 확인" 유지, 민감한 탭은 닫고 실행.
- **대량 삭제·덮어쓰기 전 반드시 사람 확인.** 임의 삭제 금지.
- 끝나면 성공 N건 / 실패 M건과 실패 사유, 저장 위치를 요약 보고.
- 생성은 **Gemini 채팅창(구독 정액 한도 내)** 에서 — 한계비용 0. API는 채팅 한도를 명백히 초과하거나 결정론적 대량 처리가 꼭 필요할 때만(장당 과금, Batch ~50% 저렴) 검토.
