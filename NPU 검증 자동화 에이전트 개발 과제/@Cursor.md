---
tags:
  - 온톨로지
  - 지식그래프
  - ontology
  - knowledgegraph
  - KG
  - KnowledgeCore
  - 검증자동화
  - Cursor
---
## User Initial Prompt

1. 핵심미션 (MVP): NPU Verilog RTL 검증 자동화 에이전트 개발
2. 자연어로 작성된 PDF형태의 NPU Architeture 문서(기본), 기존의 verilog test bench(옵션), 레퍼런스 C코드 시뮬레이터(옵션), 유저가이드 문서(옵션), micro-architecture문서(옵션), 기타 설계자가 작성한 PPT형태의 모듈 Design Review 문서(옵션)로 부터 검증 자동화(핵심응용)과 NPU IP 관련 문서 작성, 자연어로 개발자와 대화하며 NPU IP 정보를 줄 수 있는 챗봇 등의 여러 응용에 공통적으로 활용되는 지식기반 형태로 저장하고, 각 응용별로 별도의 에이전트를 제작할 것임. 입력에서 (옵션)은 정보가 주어지지 않을 수도 있음을 의미함.
3. 이 과제에 참여하는 개발자는 하드웨어 설계 지식은 있지만 타겟 NPU IP에 대해서는 제한된 지식을 갖고 있고, 온톨로지, 지식그래프, API/Agent기반 AI응용 개발등의 소프트웨어 개발 지식은 개념 정도만 이해하는 초보적인 수준이어서 방대한(400페이지 이상) NPU 사양 문서를 직접 모두 읽어서 이해하고 검증을 위한 내용을 정리하거나 온톨로지와 지식그래프, API 설계를 모두 직접할 수는 없어. 그런 부분은 LLM을 활용하고 단계적 진행할 때는 LLM이 설명을 해주고 개발자가 이해한 하 수락하면 LLM이 코딩을 하는 방식으로 하나씩 시스템을 만들어 갈거야. 
4. "사람이 문서를 읽고 정리해서 소프트웨어 구조 생성, 코딩등등의 노가다를 하지 않겠다"는 것임. 이를 위해 가장 먼저 구축해야 할 것은 **Knowledge Core(지식 코어)**임. 방대한 PDF나 C 코드를 LLM이 스스로 분석하여 **온톨로지(Ontology)** 형태의 지식 그래프로 정형화하고, 이를 다양한 에이전트들이 가져다 쓰는 '지식 허브' 구조가 필요함.

## AI

plan: npu_knowledge_core_and_verification_7005c84b.plan.md

## User

*첫번째 단계는 온톨로지의 초안이 필요한 것 같은데. 그 다음에 pdf를 파싱해서 온톨로지 구조에 내용을 채우고 의존성등까지 파악해서 지식그래프로 만들어야하고. 맞지? 현재 폴더에는 예시로서 사용할 ARM의 NPU 사양 문서가 있어(pdf). 이걸 바탕으로 온톨로지 초안을 작성하되 IP벤더와 버젼은 바뀔 수 있으므로 벤더/버전 중립적인 형태로 만들어 볼 수 있어?*

## AI

ontology_dfraft.md

## User

*초안에 따라 json/json-ld 기반으로 스키마를 만드는 게 온톨로지 설계 단계가 완성되는 거야? 다음 단계 작업을 이번 단계에서 만단 json기반으로 진행하더라도, 나는 protege툴을 맛보기 위해 동일한 내용을 protege호환 포맷으로도 저장해주면 좋겠어.*

## AI

plan: ontology_schema_json_and_protege_owl_8c7a2ba4.plan

결과: ontology_schema.json, ontology_context.jsonld, npu_ontology.ttl

## User

*좋아 우선 청킹 단계의 스크립트를 만들어주고, 실제 LLM은 233~244 페이지의 청크에 대해서만 요청해서 knowledge graph를 만들어줘. 너가 진행하는 전체 과정은 스크립트를 통해 내가 확인할 수 있도록 해줘. 청킹된 데이터들은 메타 데이터로 컨텍스트를 달아놓을거야? 냐중에 다른 페이지를 테스트해보고나 여러 부분(레지스터부분, 명령어부분)을 선택해서 LLM과 대화하려면 매번 다시 청크를 하는 것도 이상하잖아*

## AI

결과

`python scripts/chunk_pdf.py "arm_ethos-u85_npu_technical_reference_manual_102685_0000_05_en.pdf`

`python scripts/extract_kg_from_chunks.py --chunks output/chunks_*.json --page-start 233 --page-end 244`

`python scripts/run_pipeline.py arm_ethos-u85_npu_technical_reference_manual_102685_0000_05_en.pdf --page-start 233 --page-end 244`

| 파일                                  | 역할                                                                                |
| ----------------------------------- | --------------------------------------------------------------------------------- |
| scripts/chunk_pdf.py                | PDF → 페이지 단위 청킹, 메타데이터 부여, JSON 저장.                                               |
| scripts/extract_kg_from_chunks.py   | chunks JSON 로드 → 페이지 범위(233–244 등) 필터 → 스키마 로드 → 청크별 LLM 호출 → 결과 병합 → KG JSON 저장. |
| output/chunks_*.json                | 청킹 결과 (한 번만 생성 후, 페이지/섹션 필터로 재사용).                                                |
| output/knowledge_graph_233_244.json | 233–244 페이지만 LLM 돌린 결과 지식 그래프.                                                    |

## User
*지식 허브 API(JSON 파일 읽어서 조회하는 CLI/라이브러리 또는 FastAPI) 까지 만들고 간단히 테스트해보자*

## AI

knowledge_hub_api_and_test_4e679763.plan.md

|         |                                              |
| ------- | -------------------------------------------- |
| 항목      | 경로                                           |
| 라이브러리   | `kg_hub/graph.py` 또는 `scripts/kg_hub_lib.py` |
| CLI     | `scripts/kg_hub_cli.py`                      |
| FastAPI | `api/app.py` 또는 `scripts/kg_hub_api.py`      |
| 의존성     | `requirements.txt`에 fastapi, uvicorn 추가      |
| 테스트     | 위 CLI 3회 + FastAPI 3회 호출로 수동 검증              |

## User

*지금까지 완료한 내용에 대해 다른 옵션이 있었는 지 알려줘.
예를들어 지식그래프를 파일로 저장하지 않고 다른 방법은 있었는지 장단점은 무엇인 지. 비슷한 리뷰로 청킹 방식과 청킹 결과 양식, 온톨로지를 입력문서에서 찾는 것외에 설계검증을 위한 일반적으로 정의하는 리스트는 없는 지 (예, AssertLLM, UVM2 등 LLM을 활용한 Verilog design verification 논문이 많은데 개념적으로는 우리와 비슷한 방식을 갖고 있을 것 같은데 그들의 지식기반생성 방법 또는 생성한 온톨로지 또는 유사한 결과물로부터 우리의 온톨로지를 보완할 것은 없는지) 등*

## AI

.cursor\plans\npu_knowledge_core_advanced_plan.md


## User
`지금 RTL이 없는데 다음 단계로 진행가능한 내용을 알려줘`

## AI

RTL이 없어도 Phase 2를 상당 부분 진행할 수 있습니다. 다만 “실제 시뮬레이션·커버리지 측정”은 RTL이 있을 때만 가능합니다.

---

#### RTL 없이 할 수 있는 것

| 항목          | 가능한 작업                                                                                                                                                                            |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 테스트 시나리오 생성 | 지식 그래프(Instruction, Register, usesRegister, 문서 출처)를 읽어 무엇을 테스트할지 정의. 예: “NPU_OP_RESIZE는 이 레지스터들을 설정한 뒤 실행”, “테스트 시나리오: 1) 레지스터 A,B 설정 2) 인스트럭션 X 실행 3) 결과 확인”. 시나리오 명세/리스트 생성 가능. |
| 테스트벤치 보조    | 실제 DUT(RTL) 없이 TB 골격·템플릿 생성. 예: “이 인스트럭션에 필요한 레지스터 프로그래밍 시퀀스”, APB/AXI 트랜잭션 시퀀스 초안, UVM 시퀀스/테스트 클래스 스켈레톤. 나중에 RTL이 생기면 DUT만 연결하면 됨.                                               |
| 커버리지/체크리스트  | 커버리지 계획(verification plan) 또는 체크리스트 작성. 예: “Instruction 타입별 1개 이상 테스트”, “Register 타입별 1개 이상 접근”, “문서 233–244페이지 인스트럭션 전부 체크리스트”. RTL이 없어도 “무엇을 커버할지” 목록은 만들 수 있음.                 |

정리하면, “무엇을·어떤 순서로·어떤 레지스터/인스트럭션으로 테스트할지”와 TB/체크리스트 초안은 지식 허브만으로도 가능합니다.

---

### RTL이 있어야 하는 것

- 실제 시뮬레이션 실행 (RTL + TB 돌려서 pass/fail)

- RTL 기준 코드/펑션 커버리지 측정

- 커버리지 피드백으로 TB/시나리오 수정하는 완전한 루프

---

#### 진행 방식 제안

- 지금 단계: RTL 없이 검증 자동화 에이전트를 “지식 허브 기반”으로 설계·구현.

- 입력: 지식 허브 API(entities, entity/{id}, graph).

- 출력: 테스트 시나리오 명세(JSON/문서), TB 스켈레톤/템플릿, 커버리지 계획/체크리스트.

- 나중에 RTL이 생기면: 같은 에이전트에 “RTL 경로·시뮬 결과·커버리지 리포트”를 입력으로 넣어, 시나리오·TB·체크리스트를 실제 결과와 연동하는 단계만 추가하면 됨.

즉, RTL 없이도 Phase 2(시나리오·TB 보조·체크리스트/커버리지 계획)는 진행 가능하고, RTL이 있으면 그때 “실행·커버리지 측정·피드백”을 붙이면 됩니다.

1단계에서 스키마를 정의하던 때와와 과정이라고 보면 됩니다.

---

##### 1단계(스키마·지식 코어)에서 한 일

1. 스키마를 먼저 정함

온톨로지 초안(클래스·관계) → ontology_schema.json / npu_ontology.ttl

→ “무엇을 담을지 구조를 정의”

1. 그 다음 채움/사용

PDF 추출 → 지식 그래프(노드·엣지) → 지식 허브 API

→ “실제 데이터는 나중에, 구조는 이미 있음”

---

##### 2단계(검증 에이전트, RTL 없이)도 같은 패턴

1. 출력물의 스키마/형식을 먼저 정함

- 테스트 시나리오: 어떤 필드(id, instruction, registers, steps, expected 등)로 정의할지

- TB 스켈레톤: 어떤 구조(시퀀스, 트랜잭션, 레지스터 설정 블록 등)로 둘지

- 커버리지 계획/체크리스트: 어떤 항목(instruction_id, register_id, status 등)으로 둘지

1. 그 다음 에이전트가 채움

지식 허브 API로 Instruction/Register/관계 조회 → 위 스키마에 맞춰 시나리오·TB 초안·체크리스트 생성

→ “실제 시뮬/커버리지는 RTL 있을 때, 출력 형식은 미리 고정”

---

##### 정리

- 1단계: “지식의 구조(온톨로지 스키마)” 정의 → 그다음 PDF로 지식 그래프 채움.

- 2단계: “검증 산출물의 구조(시나리오/TB/체크리스트 스키마)” 정의 → 그다음 지식 그래프로 검증 산출물 채움.

그래서 “스키마(형식)를 먼저 만들고, 실제 데이터(또는 RTL)는 나중에 채운다”는 점에서 1단계와 같은 종류의 과정이라고 할 수 있습니다.

#### 결과
##### 1. 검증 산출물 스키마

- 파일: schema/verification_schemas.md

- 테스트 시나리오: scenario_id, instruction_id, description, steps(set_register / execute_instruction), registers_involved, source

- 커버리지 체크리스트: item_id, type, entity_id, description, status(pending), source

##### 2. 검증 에이전트 모듈

- verification_agent/scenarios.py: KG에서 Instruction과 usesRegister 관계를 읽어, “레지스터 설정 → 인스트럭션 실행” 순서의 시나리오 생성

- verification_agent/checklist.py: Instruction·Register 노드별로 체크리스트 항목 생성 (기본 status: pending)

##### 3. CLI

- scripts/verification_agent_cli.py

- scenarios → output/test_scenarios.json (현재 11개 시나리오)

- checklist → output/coverage_checklist.json (현재 33개 항목)

- all → 위 두 파일 모두 생성

- 기본 KG: output/knowledge_graph_233_244.json (옵션: --kg, -o)

##### 4. 실행 결과

- python scripts/verification_agent_cli.py all 실행으로

output/test_scenarios.json, output/coverage_checklist.json 생성 확인

- 시나리오 예: inst_NPU_OP_RESIZE에 대해 레지스터 11개 set_register 단계 + execute_instruction 1단계로 구성

RTL 없이도 아래처럼 이어서 진행할 수 있습니다.

---

#### 1. 지금 바로 할 수 있는 것

|항목|내용|
|---|---|
|TB 스켈레톤 생성|KG의 Instruction·Register를 이용해 테스트벤치 골격 생성. 예: 레지스터 설정용 task, 인스트럭션 실행용 task, APB/AXI 트랜잭션 뼈대. DUT는 비워 두고, 나중에 RTL만 연결.|
|검증 계획서 문서화|시나리오·체크리스트를 한데 모아 검증 계획서(Verification Plan) 마크다운/문서로 출력. 목적, 시나리오 목록, 체크리스트, 출처(문서·페이지) 정리.|
|추적성 매트릭스|시나리오 ID ↔ Instruction ↔ Register ↔ Source(문서, 페이지) 매트릭스(CSV/표) 생성. 요구사항·명세 추적용.|
|KG 확장|233–244페이지만이 아니라 전체 PDF 또는 레지스터/인스트럭션 섹션까지 추출해 KG를 키우면, 시나리오·체크리스트가 더 촘촘해짐.|

---

#### 2. 조금 더 넓히면 가능한 것

|항목|내용|
|---|---|
|어설션/프로퍼티 초안|Advanced Plan처럼 Assertion/Property를 온톨로지에 넣고, 명세 문장이나 KG에서 후보 어설션을 자연어 또는 SVA 스켈레톤으로 생성. RTL 없으므로 검증은 못 하고 “이런 걸 나중에 검증할 수 있다” 수준의 초안.|
|검증 리포트 템플릿|“KG 기반 검증 계획 생성됨. 시나리오 N개, 체크리스트 M개. RTL 미도입으로 모든 항목 pending.” 같은 상태 요약 리포트 자동 생성.|

---

#### 3. 정리

- TB 스켈레톤, 검증 계획서, 추적성 매트릭스, KG 확장은 RTL 없이도 바로 진행 가능하고, 지금 만든 시나리오·체크리스트와 자연스럽게 이어집니다.

- 어설션 초안·리포트 템플릿은 선택적으로 추가할 수 있는 단계입니다.



### AI

.cursor\plans\phase2_remaining_rtl-free_outputs_b2e0a58d.plan.md


