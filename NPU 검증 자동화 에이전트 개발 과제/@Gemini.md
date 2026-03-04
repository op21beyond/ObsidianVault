---
tags:
  - 검증자동화
  - 온톨로지
  - 지식그래프
  - KG
  - KnowledgeCore
  - knowledgegraph
  - Gemini
---

NPU(Neural Processing Unit)의 복잡한 명령어 체계와 데이터 흐름을 고려할 때, 사람이 일일이 테스트 케이스를 짜는 방식으로는 커버리지를 채우기 매우 어렵습니다. LLM을 활용해 **Spec-to-Testbench** 과정을 자동화하고, **C-Sim과 RTL 간의 Equivalence Check**를 수행하는 에이전트 아키텍처를 제안해 드립니다.

---

## 🤖 NPU 검증 자동화 에이전트: "NPU-V-Agent" 아키텍처

이 시스템은 크게 **지식 추출(RAG)**, **테스트 생성(Generation)**, **실행 및 피드백(Execution Loop)**의 3단계로 구성됩니다.

### 1. 지식 엔진 및 컨텍스트 구축 (RAG & Code Parsing)

먼저 LLM이 NPU의 사양을 완벽히 이해하도록 해야 합니다.

- **ISA Parser:** 사양 문서(PDF/MD)에서 명령어 포맷, 오퍼코드, 레지스터 맵을 추출하여 Vector DB에 저장합니다.
    
- **C-Model Analysis:** 레퍼런스 시뮬레이터(C코드)를 정적 분석하여, 특정 명령어가 입력되었을 때 기대되는 상태 변화(Golden Value) 로직을 파악합니다.
    
- **Knowledge Graph:** 명령어 간의 의존성(예: Load 후 Compute 가능)을 그래프 형태로 구축하여 유효한 시퀀스 생성을 돕습니다.
    

### 2. 하이브리드 검증 프레임워크 (Cocotb 활용 추천)

전통적인 UVM(SystemVerilog)보다는 **Cocotb(Python 기반)**를 추천합니다. LLM과의 연동이 쉽고, C-model(Reference)을 직접 Python에서 호출(ctypes/pybind11)하여 비교하기 매우 유리하기 때문입니다.

### 3. 에이전트 워크플로우 (Multi-Agent System)

에이전트를 역할별로 분리하여 협업하게 합니다.

|**역할**|**기능 설명**|
|---|---|
|**Spec Architect**|ISA 문서를 참조하여 유효한 명령어 시퀀스 및 제약 조건(Constraints) 정의|
|**Test Generator**|LLM을 이용해 `Random Stimulus` 및 `Edge Case` 테스트 코드 생성|
|**Sim Runner**|Verilator 또는 VCS/Questasim을 통해 RTL 시뮬레이션 및 C-Sim 결과 비교|
|**Coverage Analyzer**|시뮬레이션 후 도달하지 못한 코드/기능 커버리지를 분석하여 다음 테스트 방향 제시|

---

## 🛠️ 상세 구현 단계

### Step 1: LLM 기반 테스트 코드 생성 (Prompt Engineering)

LLM에게 단순 코딩이 아닌, **"ISA 규칙을 준수하는 시퀀스 생성기"** 역할을 부여합니다.

> **Prompt Example:** "NPU ISA v1.2의 CONV2D 명령어를 테스트하기 위한 Cocotb 테스트 케이스를 작성해줘. 입력 텐서 크기는 3x3~16x16 사이여야 하며, C-model의 `ref_conv2d()` 함수와 결과를 비교하는 Assert문을 포함해."

### Step 2: Coverage-Driven 피드백 루프 (The Core)

1. **Initial Run:** LLM이 생성한 테스트를 실행합니다.
    
2. **Coverage Report:** RTL 시뮬레이터에서 추출된 `.ucdb` 또는 `.vdb` 파일을 분석하여 미충족 커버리지(예: "특정 레지스터의 Overflow 상황 미발생")를 파악합니다.
    
3. **Self-Refinement:** 에이전트가 "현재 커버리지가 60%입니다. 나머지 40%를 채우기 위해 '가중치 메모리 충돌' 시나리오를 추가하세요"라는 피드백을 받아 다시 코드를 수정 생성합니다.
    

### Step 3: 버그 검출 및 리포팅

- **Mismatch Trace:** RTL 출력과 C-model 출력이 다를 경우, LLM이 시뮬레이션 로그와 Waveform(VCD)의 텍스트 요약을 분석하여 **"명령어 파이프라인의 3단계에서 데이터 포워딩 오류가 의심됨"**과 같은 가설을 제시합니다.
    

---

## 💡 구현 시 핵심 팁

- **Small chunks, Big results:** NPU 전체를 한 번에 검증하려 하지 말고, `MAC Unit` -> `DMA` -> `Controller` 순으로 에이전트가 단계를 밟도록 설계하세요.
    
- **C-Model Wrapper:** C-model을 Python 라이브러리화(Shared Library)해두면, 에이전트가 실시간으로 "정답"을 확인하며 학습할 수 있습니다.
    
- **Sanity Checker:** LLM이 생성한 Verilog/Python 코드가 문법적으로 맞는지 먼저 확인하는 `Linter 에이전트`를 중간에 두어 시뮬레이션 비용을 절감하세요.
    

**구체적으로 어떤 단계부터 시작해볼까요?** 우선 **ISA 문서를 기반으로 텍스트 기반 테스트 시퀀스를 생성하는 프롬프트 구조**부터 잡아드릴 수 있습니다. 원하신다면 말씀해 주세요!