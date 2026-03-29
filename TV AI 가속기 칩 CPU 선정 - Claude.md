---
tags:
  - CPU선정
---

# 파일 : AI_Accelerator_CPU_Selection_Report.docx
# 주제
Tv soc 옆에 pcie로 연결하여 ai 가속기 역할을 할 칩을 설계하려고 해. 가속기 칩에는 32 tops NPU가 내장되는데 이 기속기 칩의 CPU 사양 결정을 위해 고려해야할 사항을 알려줘. 메인 tv soc는 단독으로도 작은 cpu gpu npu등을 포함하고 풀 tv기능이 구현가능하고 가속기칩과 연결하는 구성은 ai를 차별화로 내세우는 플래크십 제품에 사용할거야. Cpu  선택가능범위는 ARM cortex a73, cortex a550,arm cortec m시리즈,  sifive RISC-V P550후속이야. ai기속기 칩에 포함되므로 npu제어, npu와  협동하는 heterogeneous ai acceleration, npu가 큰모델을 처리하고 작고 자주처리하는 모델은npu가 담당, 호스트 tv soc와 인터페이스등이 사용시나리오 범위일때 구체적으로 지원해야하는 사항, 해당항목에서 후보 cpu의 우위를 평가할수 있는 체크리스트, 최종 판단 방법을 칩개발기획전문가이자 시스템 전문가로써 상사에게 보고할 때의 논리적이고 구체적인 방식으로 설명해줘. 관련 SW에코시스템에서는 구체적으로 점검해야할 라이브러리 프레임붜크 기타 필수함수 벤치마크 의 리스트와  평가방법도 포함해줘. 참고로 NPU가 네트워크 모델을 처리할때 ops 시퀀스는 mpu 내부의 작은 프로세스가 미리 컴파일된 스케쥴링 순서에 따라 제어를 하므로 CPU의 개입은 거의 없어. 하지만 트낸스포머 효율화 기술들이 계속 나오면서 모델의 중간 단계에서 CPU가 처리해야할 일들이 생기는 트렌드인듯.    
    
구글의 터보퀀터까지 지원하기위해 필요한 요소를 NPU와 CPU의 SW, HW로 구분해보고  CPU 선정기준에 고려대상이 된다면 포함시켜줘.https://research.google/blog/turboquant-redefining-ai-efficiency-with-extreme-compression/

터보퀀트 관련해서는는 단독 고려사항보다는 다른 카테고리의 예로써 포함시켜주고 부록으로는 자세히설명하면돼

# 조사내용


기술 기획 보고서

**AI** **가속기 칩 CPU 선정 기준 분석**

32 TOPS NPU 탑재 PCIe 연결형 가속기 칩 설계 기획

보고 대상 | 개발 본부장

작성자 | 칩 개발기획 / 시스템 아키텍처팀

기준일 | 2025년

|   |
|---|
|본 문서는 TV 플래그십 제품에 탑재될 PCIe 연결형 AI 가속기 칩의 CPU 선정을 위한 공식 검토 보고서입니다.<br><br>후보군 평가, SW 에코시스템 점검, 최종 판단 프레임워크를 포함합니다.|

  
  

# 1. 보고 개요 및 배경

## 1.1 보고 목적

본 보고서는 TV 플래그십 제품의 AI 차별화 전략을 뒷받침하기 위해 메인 TV SoC와 PCIe로 연결되는 AI 가속기 칩에 탑재할 CPU 코어 선정 기준을 수립하고, 4개 후보군에 대한 체계적 평가 프레임워크를 제공하는 것을 목적으로 한다.

## 1.2 시스템 구성 개요

대상 시스템은 두 개의 칩으로 구성된다. 메인 TV SoC는 독립적으로 완전한 TV 기능을 수행하며 자체 CPU/GPU/NPU를 보유한다. 가속기 칩은 PCIe를 통해 연결되며, 32 TOPS급 NPU를 주 자산으로 하여 AI 추론 성능을 플래그십 수준으로 끌어올린다. 가속기 칩 내 CPU는 NPU의 제어 두뇌이자 이 두 칩 사이의 SW 인터페이스 관리자 역할을 담당한다.

|   |   |
|---|---|
|**구성 요소**|**주요 역할 및 특징**|
|**메인 TV SoC**|독립 TV 기능 완결, 자체 CPU/GPU/NPU 포함, PCIe Host로 동작|
|**AI** **가속기 칩 (본 보고 대상)**|32 TOPS NPU + CPU + 메모리 인터페이스, PCIe EP로 동작, 플래그십 AI 차별화 담당|
|**연결 인터페이스**|PCIe (Gen 3/4), 메모리는 독립 LPDDR 또는 공유 구성|

## 1.3 CPU 후보군

아래 4개 후보에 대해 본 보고서 전반에 걸쳐 비교 평가를 수행한다.

|   |   |   |   |
|---|---|---|---|
|**후보**|**ISA**|**아키텍처 세대**|**주요 포지셔닝**|
|**ARM Cortex-A73**|ARMv8-A|2017년 (구세대)|레거시 고성능 Big 코어, 넓은 SW 검증 이력|
|**ARM Cortex-A550**|ARMv9-A|2022–23년 (최신 효율)|고효율 미들 코어, SVE2/BF16 지원, 에너지 최적화|
|**ARM Cortex-M** **시리즈 (M55/M85)**|ARMv8-M|MCU 급 (저전력)|초저전력 실시간 제어, Helium(MVE) 벡터 내장, RTOS 최적화|
|**SiFive RISC-V P550** **후속 (P650/P870급)**|RISC-V|2023–24년 (고성능)|오픈 ISA, RVV 1.0 벡터, 커스터마이징 가능, 에코 성장 중|

  
  

# 2. 사용 시나리오별 CPU 요구사항

AI 가속기 칩 내 CPU의 역할은 단순한 NPU 제어에 그치지 않는다. 아래에 제시된 4대 시나리오는 CPU가 수행해야 할 기능의 범위를 정의하며, 각각은 독립된 성능·기능 요구사항을 갖는다. 특히 Transformer 최적화 기술의 빠른 발전으로 인해, 시나리오 2와 3의 비중이 지속적으로 증가하는 추세임을 주목해야 한다.

## 2.1 시나리오 A — NPU 제어 (Control Plane)

NPU가 네트워크 연산을 수행하는 동안 CPU의 직접적 개입은 최소화된다. 그러나 CPU는 NPU가 연산을 시작하기 전후, 즉 작업 큐 제출·완료 통지·오류 처리·전력 상태 전환 구간에서 핵심적 역할을 수행한다.

|   |
|---|
|**핵심 요구사항 — NPU 제어**|
|•     인터럽트 레이턴시: NPU 완료 인터럽트 수신 후 1μs 이내 응답 보장<br><br>•     MMIO 접근: 32/64-bit 레지스터 읽기/쓰기, Non-cacheable 메모리 맵 관리<br><br>•     DMA 디스크립터 관리: 다중 채널 DMA 링 버퍼 설정 및 완료 처리<br><br>•     멀티 스트림 큐 관리: 복수의 AI 작업을 우선순위 기반으로 스케줄링<br><br>•     실시간성: 25ms 이내 end-to-end 제어 루프 보장 (1080p@60fps 기준 프레임 기간 이내)<br><br>•     전력 상태 기계: NPU idle/active 전환에 따른 클럭/전압 제어 명령 발행|

## 2.2 시나리오 B — CPU-NPU 이종 협동 가속 (Heterogeneous AI)

Transformer 모델의 효율화 기술이 빠르게 발전하면서, 모델 추론의 중간 단계에 CPU 처리가 요구되는 사례가 증가하고 있다. 이는 본 CPU 선정에서 가장 주목해야 할 트렌드이다.

대표적인 협동 처리 시나리오는 다음과 같다.

•     Speculative Decoding: Draft 모델(소형)은 CPU에서, Verify 단계는 NPU에서 수행

•     Mixture of Experts (MoE) 라우팅: Expert 선택 로직은 CPU soft-max + top-k 연산으로 처리

•     Dynamic KV Cache 관리: 캐시 eviction 정책, 양자화 파라미터 갱신을 CPU가 담당

•     KV Cache 압축 전처리: 예) TurboQuant의 PolarQuant 단계에서 random rotation matrix 연산 (부록 A 참조)

•     Tensor reshape/transpose: NPU 미지원 연산을 CPU 폴백으로 처리

•     Token Sampling: Temperature/Top-p 샘플링, beam search 등 CPU 실행

|   |
|---|
|**핵심 요구사항 — Heterogeneous AI**|
|•     SIMD 폭: 최소 128-bit, 권장 256-bit 이상 (INT8 dot product, FP16/BF16 연산)<br><br>•     L1/L2 캐시: KV cache 버퍼 재사용성 확보 — L2 1MB 이상 권장<br><br>•     메모리 대역폭: NPU와 동일 메모리 풀 공유 시 최소 25 GB/s 이상<br><br>•     Cache Coherency: CPU-NPU 간 zero-copy 텐서 공유를 위한 coherent 메모리 인터페이스<br><br>•     BF16/FP16 네이티브 지원: Transformer attention score 계산 시 정밀도 보장<br><br>•     INT4/INT8 역양자화 커널 실행 능력: 극압축 모델(3-4bit) 지원 시 필수|

## 2.3 시나리오 C — CPU 직접 추론 (Hybrid Execution)

NPU는 대형 모델 처리에 집중하고, CPU는 빈번하게 호출되는 소형 모델을 직접 처리하는 역할 분담 구조를 취한다. 해당 모델들은 NPU 오버헤드(작업 스케줄링, 컨텍스트 스위칭)보다 CPU 직접 실행이 낮은 레이턴시를 제공할 때 적합하다.

CPU 직접 추론 대상 모델 예시:

•     Wake-word detection (< 1M params, always-on)

•     Voice Activity Detection (VAD)

•     의도 분류 / 개체명 인식 (경량 NLP, BERT-tiny급)

•     영상 노이즈 측정·분류 전처리 (소형 CNN)

•     이상 감지 모델 (정형 데이터 기반 경량 모델)

|   |
|---|
|**핵심 요구사항 — CPU 직접 추론**|
|•     INT8 추론 성능: TFLite/ONNX Runtime 기준 Mobilenet-V2 INT8 < 5ms (1코어)<br><br>•     벡터 연산 처리량: CMSIS-NN 또는 XNNPACK 기반 convolution 커널 최적화 가능 여부<br><br>•     Always-on 모드: < 10mW 대기 전력 상태에서도 wake-word 탐지 가능 (M 시리즈 유리)<br><br>•     메모리 공간: 모델 가중치 + 활성화 버퍼 — 최소 2MB SRAM 또는 256MB LPDDR 접근<br><br>•     레이턴시: end-to-end 추론 < 20ms (사용자 인터랙션 응답성 기준)|

## 2.4 시나리오 D — Host SoC 인터페이스 관리

가속기 칩은 PCIe EP(Endpoint)로 동작하며, 메인 TV SoC의 PCIe RC(Root Complex)와 통신한다. CPU는 이 인터페이스 계층 전체를 관리하는 펌웨어/드라이버의 실행 주체이다.

|   |
|---|
|**핵심 요구사항 — Host Interface**|
|•     PCIe EP 드라이버: BAR 공간 관리, MSI/MSI-X 인터럽트 처리, TLP 생성/수신<br><br>•     명령 프로토콜: Host SoC↔가속기 간 추론 요청/응답 메시지 큐 처리 (<1ms 명령 파싱)<br><br>•     DMA 엔진 제어: Host 메모리→NPU 메모리 대용량 가중치 전송 (초기 로딩 / 모델 교체)<br><br>•     보안 격리: IOMMU/SMMU를 통한 메모리 접근 보호, TrustZone 또는 RISC-V PMP 활용<br><br>•     펌웨어 업데이트: 보안 부트, OTA 업데이트 검증 (SHA-256, RSA/ECDSA)<br><br>•     전원 관리: D-state 전환 (D0/D3hot), 가속기 칩 전체 전력 상태 조율|

  
  

# 3. 후보 CPU 아키텍처 특성 분석

## 3.1 ARM Cortex-A73

ARMv8-A 기반의 Out-of-Order 슈퍼스칼라 코어로, 3-wide decode, 2.5GHz 이상 동작이 가능하다. 2017년에 설계된 IP이나 다수 양산 SoC에 채택된 이력이 있어 SW 검증 기반이 넓다. NEON SIMD(128-bit)를 지원하나 SVE/SVE2, BF16, INT8 dot product 명령은 부재하다. Linux 메인라인 지원은 완벽하나, 최신 AI 연산 가속 명령어 부재로 인해 시나리오 B(Heterogeneous AI)에서 성능 상 불리하다. 에너지 효율 측면에서도 최신 ARMv9 대비 열세이다. 후보군 중 가장 성숙한 에코시스템을 보유하지만 미래 지향성이 가장 낮다.

## 3.2 ARM Cortex-A550

ARMv9-A 기반의 효율 지향 코어로, A55 계열의 설계 철학을 계승하면서 SVE2(Scalable Vector Extension 2), BF16, INT8 dot product를 지원한다. 에너지 효율이 대폭 개선되었으며, ARM의 최신 TrustZone 확장인 RME(Realm Management Extension)를 포함하여 보안 격리 구조도 강화되었다. AI 가속기 칩처럼 전력 예산이 제한된 환경에서 성능과 전력의 균형이 가장 우수한 후보이다. TFLite, ONNX Runtime, ArmNN 등 주요 ML 프레임워크에서 최적화 커널이 지속적으로 공급된다는 점도 중요한 강점이다.

## 3.3 ARM Cortex-M55 / M85

ARMv8.1-M ISA 기반의 MCU 급 코어로, Helium(MVE: M-Profile Vector Extension)을 통해 INT8/FP16 벡터 연산을 하드웨어에서 처리한다. 마이크로와트 수준의 전력 소비로 always-on 동작이 가능하며, RTOS 친화적 설계(우선순위 인터럽트, MPU, 결정론적 레이턴시)가 특징이다. 그러나 가상 메모리(MMU) 미지원으로 인해 Linux 구동이 불가하며, 대용량 ML 프레임워크 스택 실행에 제약이 있다. NPU 제어 전용(Control Plane only) 또는 always-on 소형 모델 전담 코어로는 최적이나, 복잡한 이종 협동 처리 시나리오에서는 능력 한계에 직면한다.

## 3.4 SiFive RISC-V P550 후속 (P650 / P870급)

P550은 ARM Cortex-A55 수준의 성능을 목표로 설계된 고성능 RISC-V 코어이며, 그 후속인 P650/P870급은 더 높은 IPC와 RVV 1.0(RISC-V Vector Extension) 풀 지원을 제공한다. 오픈 ISA 특성상 맞춤형 명령어 확장(Custom Extension)이 가능하며, 이는 NPU와의 긴밀한 인터페이스 설계에서 유리하게 작용할 수 있다. 그러나 ARM 대비 SW 에코시스템의 성숙도가 현저히 낮다. TFLite, ONNX Runtime의 RISC-V 최적화 백엔드는 아직 ARM NEON/SVE 수준에 미치지 못하며, 상용 ML 프레임워크의 검증 이력도 제한적이다. 공급망 독립성과 로열티 절감을 전략적으로 중시하는 경우 고려 가치가 있다.

  
  

# 4. 평가 체크리스트 — 후보 CPU 비교

아래 체크리스트는 각 시나리오별 요구사항을 평가 항목으로 분해하고 4개 후보의 상대적 우위를 나타낸다. 범례: ◎ 최우수 / ○ 우수 / △ 보통 / ✕ 미흡

### 4.1 NPU 제어 / Control Plane 역량

|   |   |   |   |   |
|---|---|---|---|---|
|**평가 항목**|**A73**|**A550**|**M55/85**|**RV P550+**|
|인터럽트 레이턴시 (<1μs 보장)|○ 우수|◎ 최우수|◎ 최우수|○ 우수|
|MMIO/레지스터 접근 성능|○ 우수|○ 우수|◎ 최우수|○ 우수|
|DMA 디스크립터 처리 능력|○ 우수|○ 우수|○ 우수|○ 우수|
|RTOS 지원 / 결정론적 스케줄링|△ 보통|△ 보통|◎ 최우수|△ 보통|
|전력 상태 제어 (저전력 idle 진입)|△ 보통|○ 우수|◎ 최우수|△ 보통|
|멀티 큐 NPU 작업 스케줄링 (OS 필요)|○ 우수|○ 우수|✕ 미흡|○ 우수|

### 4.2 Heterogeneous AI 협동 처리 역량

|   |   |   |   |   |
|---|---|---|---|---|
|**평가 항목**|**A73**|**A550**|**M55/85**|**RV P550+**|
|SIMD 폭 / 처리량 (128-bit 이상)|○ 우수(NEON)|◎ 최우수(SVE2)|○ 우수(MVE)|○ 우수(RVV1.0)|
|BF16 네이티브 지원|✕ 미흡|◎ 최우수|✕ 미흡|○ 우수|
|INT8 dot product 명령어|✕ 미흡|◎ 최우수|○ 우수|○ 우수|
|INT4 역양자화 연산 지원|△ 보통|○ 우수|△ 보통|△ 보통|
|Cache Coherency (CPU-NPU 공유)|○ 우수|○ 우수|△ 보통|○ 우수|
|KV Cache 압축 전처리 (rotation 등)|△ 보통|○ 우수|△ 보통|△ 보통|
|Dynamic routing / MoE 분기 처리|○ 우수|○ 우수|✕ 미흡|○ 우수|
|Token Sampling (softmax, top-k)|○ 우수|○ 우수|△ 보통|○ 우수|

### 4.3 CPU 직접 추론 역량

|   |   |   |   |   |
|---|---|---|---|---|
|**평가 항목**|**A73**|**A550**|**M55/85**|**RV P550+**|
|INT8 추론 (MobileNet-V2 기준)|△ 보통|○ 우수|○ 우수|△ 보통|
|경량 모델 레이턴시 (<5ms 목표)|△ 보통|○ 우수|◎ 최우수|△ 보통|
|Always-on 대기 전력 (<10mW)|✕ 미흡|○ 우수|◎ 최우수|✕ 미흡|
|TFLite/ONNX RT 최적화 커널 가용성|○ 우수|◎ 최우수|○ 우수|△ 보통|
|CMSIS-NN / XNNPACK 지원|○ 우수|◎ 최우수|◎ 최우수|△ 보통|

### 4.4 Host Interface / 시스템 역량

|   |   |   |   |   |
|---|---|---|---|---|
|**평가 항목**|**A73**|**A550**|**M55/85**|**RV P550+**|
|Linux 메인라인 지원 (드라이버 개발)|◎ 최우수|◎ 최우수|✕ 미흡|○ 우수|
|PCIe EP 펌웨어 작성 용이성|◎ 최우수|◎ 최우수|△ 보통|△ 보통|
|TrustZone / 보안 격리|○ 우수|◎ 최우수(RME)|○ 우수|○ 우수(PMP)|
|OTA 보안 부트 지원|○ 우수|○ 우수|○ 우수|△ 보통|
|IOMMU/SMMU 지원|○ 우수|○ 우수|✕ 미흡|△ 보통|

### 4.5 전력 / 면적 / 공정 효율 (PPA)

|   |   |   |   |   |
|---|---|---|---|---|
|**평가 항목**|**A73**|**A550**|**M55/85**|**RV P550+**|
|에너지 효율 (DMIPS/mW)|✕ 미흡|◎ 최우수|◎ 최우수|○ 우수|
|면적 효율 (성능/mm²)|✕ 미흡|○ 우수|◎ 최우수|○ 우수|
|최신 공정 적용 가능성|✕ 미흡|◎ 최우수|◎ 최우수|○ 우수|
|발열 관리 (TDP 여유)|✕ 미흡|○ 우수|◎ 최우수|○ 우수|

### 4.6 SW 에코시스템 성숙도

|   |   |   |   |   |
|---|---|---|---|---|
|**평가 항목**|**A73**|**A550**|**M55/85**|**RV P550+**|
|ML 프레임워크 최적화 커버리지|○ 우수|◎ 최우수|○ 우수|△ 보통|
|NPU 드라이버 개발 레퍼런스|○ 우수|◎ 최우수|△ 보통|△ 보통|
|커뮤니티·상업 지원 수준|○ 우수|◎ 최우수|○ 우수|△ 보통|
|공급망·라이선스 독립성|✕ 미흡|✕ 미흡|✕ 미흡|◎ 최우수|
|사내 팀 축적 기술 연계 가능성|○ 우수|◎ 최우수|○ 우수|△ 보통|

  
  

# 5. SW 에코시스템 점검 — 라이브러리·프레임워크·벤치마크

CPU 하드웨어의 성능이 실제 제품 가치로 이어지려면 SW 에코시스템의 지원이 필수적이다. 아래 항목들은 설계 착수 전 반드시 점검해야 할 체크포인트이며, 각 항목에 대해 4개 후보의 지원 현황과 평가 방법을 명시한다.

## 5.1 OS 및 런타임 환경

|   |   |   |
|---|---|---|
|**항목**|**점검 내용**|**평가 방법**|
|**Linux Kernel**|메인라인 커널(6.x) 상의 타겟 CPU 지원 여부, BSP 제공 여부, 드라이버 포팅 난이도|kernel.org defconfig 빌드 + boot test (QEMU 포함)|
|**FreeRTOS / Zephyr**|NPU 제어 RTOS 구동 가능성, 태스크 우선순위/인터럽트 지연 측정|IRQ latency 측정 (cyclictest 또는 custom timer ISR)|
|**Bare-metal** **펌웨어**|PCIe EP 초기화, NPU 부트 시퀀스 구현 용이성, HAL 지원|시동 시간 측정, 코드 크기 분석|

## 5.2 ML 추론 프레임워크

|   |   |   |   |
|---|---|---|---|
|**프레임워크**|**타겟 시나리오**|**핵심 점검 항목**|**측정 지표**|
|**TensorFlow Lite (TFLite)**|CPU 직접 추론, 협동 처리 fallback|NEON/SVE2/RVV 델리게이트 활성화 여부, Flex ops 지원|MobileNet-V2 INT8 레이턴시, 메모리 풋프린트|
|**ONNX Runtime**|범용 추론, 서버급 모델 CPU 실행|EP(Execution Provider) 플러그인 구조, 커스텀 NPU EP 작성 가능성|Resnet-50 FP32/INT8 throughput|
|**ExecuTorch (PyTorch Mobile)**|LLM 추론, 이종 실행 분기|Custom backend 등록, RISC-V 포팅 현황|LLaMA-3B 첫 토큰 레이턴시|
|**NCNN / MNN**|임베디드 경량 추론|ARM/RISC-V 어셈블리 커널 가용성|YOLOv5s 처리 시간|

## 5.3 연산 라이브러리 (Kernel-level)

|   |   |   |
|---|---|---|
|**라이브러리**|**역할 및 지원 범위**|**후보별 지원 현황**|
|**CMSIS-NN**|ARM M/A 시리즈용 NN 커널 (conv, depthwise, LSTM). Helium 최적화 포함|A73/A550/M55 지원 ◎  \|  RISC-V ✕|
|**XNNPACK**|Google 개발 포터블 NN 커널. NEON/SSE/WASM 지원. TFLite 기본 백엔드|A73/A550 ◎  \|  M시리즈 △  \|  RISC-V ○(부분)|
|**ArmNN**|ARM HW 가속기 추상화 레이어. NPU delegate로 활용 가능|ARM 계열 전용 ◎  \|  RISC-V ✕|
|**OpenBLAS / libflat**|GEMM 연산 기반 Transformer 처리. CPU 행렬 연산 백본|A73/A550 ◎  \|  RISC-V ○  \|  M시리즈 ✕|
|**ggml / llama.cpp**|LLM 추론용 경량 C 라이브러리. INT4/INT8 양자화 내장. CPU 추론 기준점|A73/A550 ◎  \|  RISC-V ○(성장중)  \|  M ✕|
|**RVVN (RISC-V** **커널)**|RVV 1.0 기반 NN 커널. T-Head CSI-NN, RVNN 등|RISC-V 전용. 성숙도 성장 중 △|

## 5.4 NPU 드라이버 / 컴파일러 연동

가속기 칩의 NPU는 자체 설계(in-house)이므로, CPU-NPU 드라이버 스택은 100% 자체 개발이 필요하다. 이 때 CPU 아키텍처 선택은 드라이버 개발 생산성에 직결된다.

|   |   |   |
|---|---|---|
|**점검 항목**|**중요성 및 내용**|**평가 방법**|
|**Linux Kernel DMA-buf / ION**|NPU↔CPU 간 zero-copy 텐서 공유. 최신 DMA-heap API 지원 여부|Linux v5.10+ 기준 DMA-heap driver 포팅 테스트|
|**OpenCL / Vulkan Compute**|표준 이종 컴퓨팅 API 지원. CPU fallback 커널 작성 표준화 기반|clpeak / VkFFT 벤치마크로 컴퓨팅 처리량 측정|
|**LLVM/Clang** **툴체인 지원**|NPU 컴파일러 백엔드와 동일 툴체인 공유 가능 여부. MLIR 통합 용이성|MLIR CPU dialect 빌드·실행 검증|
|**TVM / Apache TVM**|CPU+NPU 이종 실행 그래프 분할. 각 CPU 타겟 백엔드 지원 성숙도|ResNet-50 TVM auto-tuning 시간·성능 비교|

## 5.5 성능 벤치마크 — 필수 측정 항목

|   |   |   |   |
|---|---|---|---|
|**벤치마크**|**측정 대상 시나리오**|**핵심 지표**|**판단 기준값 (예시)**|
|**MLPerf Tiny v1.1**|CPU 직접 추론 (시나리오 C)|IC, AD, KWS, VWW 점수|KWS >95% 정확도, <2ms|
|**EEMBC MLMark**|CPU+NPU 이종 처리 처리량|Images/sec, Energy/inference|전력 대비 처리량 최대화|
|**Geekbench ML 5**|CPU 단독 ML 처리 능력|Inference 점수|A55 기준 +30% 이상|
|**llama.cpp benchmark**|LLM token 처리 (시나리오 B)|Token/sec (prompt/gen)|≥15 token/s (7B INT4 기준)|
|**cyclictest (RT-Preempt)**|NPU 제어 실시간성 (시나리오 A)|Max latency (μs)|Max jitter <50μs|
|**STREAM bandwidth**|메모리 대역폭 (이종 처리 병목)|GB/s (copy/scale/add/triad)|>20 GB/s (공유 LPDDR5 기준)|
|**CoreMark / SPEC CPU2017**|기본 CPU 처리 능력|CoreMark/MHz|>5 CoreMark/MHz (A55=3.68)|

  
  

# 6. 극한 양자화 기술 트렌드 대응 — TurboQuant을 사례로

본 섹션은 Transformer 효율화 기술의 진화가 CPU 선정에 미치는 영향을 구체적 기술 사례를 통해 설명한다. Google Research가 2026년 ICLR에 발표 예정인 TurboQuant는 KV Cache를 3-bit 수준으로 압축하면서도 정확도 손실을 제로(zero-loss)로 유지하는 알고리즘으로, 이 계열의 기술이 가속기 칩의 CPU 설계 방향에 미치는 시사점을 점검한다.

(TurboQuant 알고리즘 상세 설명은 부록 A를 참조. 본 섹션에서는 CPU 선정 관점의 요구사항 도출에 집중한다.)

## 6.1 Heterogeneous AI 시나리오에서의 CPU 역할 증대

TurboQuant와 같은 기술들은 모델 추론 파이프라인을 단순한 'NPU가 전부 처리하는' 구조에서 벗어나게 만든다. 구체적으로 CPU가 처리해야 하는 중간 단계가 발생하는 지점은 다음과 같다.

|   |   |
|---|---|
|**처리 단계**|**CPU** **요구사항**|
|**① PolarQuant: Random Rotation Matrix** **곱셈**|고차원 벡터 rotation — BF16/FP16 SIMD 연산 필수. SVE2 벡터 레지스터가 처리 효율에 직결됨. 이 단계가 CPU 병목이 되면 전체 KV 압축 파이프라인이 정체됨|
|**② QJL: 1-bit** **잔차 벡터 생성 (sign bit 추출)**|bit-manipulation 집약적 연산. 빠른 sign bit 추출 명령어(NEON/MVE vshrn 등) 지원 여부가 성능 결정|
|**③** **압축 KV Cache 역양자화 (디코딩 단계)**|Attention score 계산 전 3-bit→FP16 역양자화. INT4/INT8 역양자화 커널 성능이 중요. 이를 NPU에 내리지 못하는 경우 CPU가 전담|
|**④ Quantization** **상수 관리 (online 갱신)**|PolarQuant의 radius(크기) 값 저장·로딩. 캐시 효율성이 중요 — L2 캐시 크기와 정책에 민감|

## 6.2 이 사례가 CPU 선정에 주는 시사점

TurboQuant 사례는 단독으로 새로운 선정 기준을 만드는 것이 아니라, 기존 기준(4.2절 Heterogeneous AI 역량)의 중요도를 확인하고 강화하는 증거로 작용한다. 특히 다음 항목들의 우선순위가 상향되어야 함을 시사한다.

•     BF16/FP16 네이티브 연산 지원: PolarQuant rotation 단계에서의 성능 결정 요소

•     L2 캐시 용량 및 정책: KV cache 압축 파라미터 재활용률이 전체 대역폭 효율을 좌우

•     INT4 역양자화 처리: 3-4bit 양자화 모델의 CPU 처리 병목 방지

•     빠른 컨텍스트 스위칭: NPU↔CPU 간 제어 전달 횟수 증가에 대응

|   |
|---|
|**결론: CPU 선정 기준에 반영할 내용**|
|TurboQuant 계열 기술 지원을 위해 Heterogeneous AI 시나리오(4.2절) 평가 가중치를 상향 조정하되, 특히 ① BF16/SIMD 처리 폭, ② L2 캐시 1MB 이상 여부, ③ INT4 역양자화 커널 성능을 핵심 차별 요소로 설정하는 것을 권고한다. 이 세 항목에서 Cortex-A550이 후보군 중 유일하게 모두 충족하는 것으로 평가된다.|

  
  

# 7. 최종 판단 방법론 및 권고

## 7.1 가중치 기반 평가 매트릭스

아래 매트릭스는 각 평가 카테고리에 사업 및 기술 중요도에 따른 가중치를 부여하고, 5점 척도(◎=5, ○=4, △=2, ✕=1)로 환산한 총합 점수를 산출한다. 가중치는 본 제품의 전략 방향(AI 차별화 플래그십)에 근거하여 설정하였다.

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|**평가 카테고리**|**가중치**|**A73**|**A550**|**M55/85**|**RV P550+**|
|**NPU** **제어 / Control Plane**|15%|3.5|4.2|4.5|3.5|
|**Heterogeneous AI** **협동 처리 ★**|**30%**|2.5|4.5|2.8|3.5|
|**CPU** **직접 추론 (Hybrid)**|20%|2.8|4.2|4.0|3.0|
|**Host Interface /** **시스템**|15%|4.0|4.5|2.5|3.2|
|**PPA (****전력/면적/공정)**|10%|1.5|4.5|5.0|3.8|
|**SW** **에코시스템 성숙도**|10%|3.8|5.0|3.5|2.5|
|**가중 합산 점수 (최대 5.0)**|100%|**2.82**|**4.42 ★**|**3.28**|**3.18**|

## 7.2 후보별 최종 포지셔닝

|   |   |   |   |
|---|---|---|---|
|**후보**|**종합 판정**|**선택 시 적합 조건**|**선택 시 리스크**|
|**A73**|**❌** **비권고**|단기 재고 활용, 레거시 SoC 계열 재사용 시에만 고려|노후 아키텍처, BF16 미지원으로 AI 기능 확장 불가, 에너지 열세|
|**A550**|**✅** **최우선 권고**|AI 차별화 플래그십 전략 전면 부합. 양산 일정·에코시스템 위험 최소|ARM IP 로열티 의존도 유지. 공급망 단일화 위험은 이중 소싱으로 관리 가능|
|**M55/M85**|**⚠️** **조건부 권고**|전력 예산이 극히 제한적이고, 복잡한 이종 처리가 불필요한 구성 (Control Plane 전용)에 적합|Linux 미지원으로 복잡한 SW 스택 구동 불가. 향후 기능 확장에 병목 발생 가능|
|**RISC-V P650+**|**🔍** **전략 검토**|로열티·공급망 독립 전략, 차세대 제품군 장기 로드맵, 자체 커스터마이징 필요 시 검토|ML 에코시스템 미성숙, 드라이버 개발 공수 2~3배, 양산 검증 이력 부족|

## 7.3 최종 판단 프로세스

정량적 매트릭스 점수는 출발점이며, 최종 결정은 아래 3단계 검증 게이트를 순차적으로 통과해야 한다.

1.  【Gate 1 — 기능 필수 요건 Pass/Fail】 Linux 메인라인 지원, BF16 SIMD, PCIe EP 드라이버 구현 가능성, NPU Cache Coherency 인터페이스 중 하나라도 'Fail'인 후보는 즉시 제외. 이 기준에서 A73(BF16 미지원)과 M시리즈(Linux 미지원)는 플래그십 AI 가속기 칩에서 탈락 조건에 해당한다.

2.  【Gate 2 — 벤치마크 목표값 달성 검증】 5.5절의 벤치마크를 수행하여 목표값(cyclictest <50μs jitter, MLPerf KWS <2ms, STREAM >20GB/s, llama.cpp ≥15 token/s)을 모두 달성하는지 prototype 보드에서 실측한다. 매트릭스 점수가 높더라도 실측에서 미달하면 재검토가 필요하다.

3.  【Gate 3 — 사업·개발 리스크 심의】 IP 라이선스 비용, 공급업체 납기·지원 수준, 내부 팀 숙련도, 양산 일정과의 정합성을 종합하여 최종 승인한다. RISC-V 계열은 이 단계에서 차세대 제품 로드맵 편입 여부를 결정한다.

## 7.4 결론 및 실행 권고

|   |
|---|
|**최종 권고 — Cortex-A550 채택 (단일 A 클러스터 or Dual-core 구성)**|
|•     Cortex-A550은 플래그십 AI 가속기 칩에 요구되는 NPU 제어, 이종 협동 처리, CPU 직접 추론, Host 인터페이스 관리의 4대 시나리오를 모두 충족하는 유일한 후보이다.<br><br>•     SVE2, BF16, INT8 dot product 지원은 TurboQuant 계열로 대표되는 Transformer 효율화 기술의 미래 요구사항에도 선제적으로 대응한다.<br><br>•     ARM의 최신 ML 에코시스템(ArmNN, XNNPACK, TFLite 최적화 백엔드)과의 연계로 드라이버/SW 개발 공수를 최소화할 수 있다.<br><br>•     단기 실행: Cortex-A550 기반 Prototype 보드 제작 → 7.3절 Gate 2 벤치마크 실측 → Q2 설계 확정<br><br>•     중장기 전략: 차세대(2~3세대 후) 제품에서 RISC-V 기반 커스텀 코어 전환 가능성을 병행 탐색하는 투-트랙 로드맵을 권고한다.|

  
  

# 부록 A — TurboQuant 상세 분석 및 NPU/CPU 구현 요건

본 부록은 Google Research가 발표한 TurboQuant 알고리즘의 기술 원리와, 이를 지원하기 위해 NPU 및 CPU 각각에 필요한 하드웨어·소프트웨어 요소를 상세히 기술한다. (원문 출처: research.google/blog/turboquant-redefining-ai-efficiency-with-extreme-compression, ICLR 2026 발표 예정)

## A.1 알고리즘 개요

TurboQuant는 Transformer 모델의 KV(Key-Value) Cache를 3-bit 수준으로 극압축하면서도 정확도 손실이 제로인 알고리즘이다. 두 가지 핵심 구성요소인 PolarQuant와 QJL의 조합으로 동작한다.

### A.1.1 QJL (Quantized Johnson-Lindenstrauss)

Johnson-Lindenstrauss Transform을 이용하여 고차원 벡터를 차원 축소하면서 벡터 간 거리(내적)를 보존한다. 각 결과 요소를 부호 비트(+1/-1) 1비트로 표현하므로 메모리 오버헤드가 제로이다. 이때 특수 추정기(estimator)가 고정밀 쿼리와 저정밀 압축 데이터를 전략적으로 균형 잡아 attention score의 정확도를 유지한다.

### A.1.2 PolarQuant

벡터를 직교 좌표(Cartesian) 대신 극좌표(Polar Coordinate)로 변환하여 양자화한다. 핵심 아이디어는 벡터를 먼저 무작위 회전(random rotation)한 뒤, 각도 성분에 표준 양자화기를 적용하는 것이다. 각도 분포가 예측 가능한 패턴(circular grid)을 가지므로 정규화 단계 없이 메모리 오버헤드를 제거할 수 있다. 대부분의 비트(주 압축)를 이 단계에서 소모한다.

### A.1.3 TurboQuant 파이프라인

PolarQuant로 주요 압축(대부분의 비트) → QJL로 잔차 오차를 1-bit로 보정하는 2단계 구조이다. H100 GPU에서 4-bit TurboQuant 기준 32-bit 대비 최대 8배 attention logit 계산 속도를 달성하였으며, KV memory를 최소 6배 축소하면서 LongBench, RULER 등 벤치마크에서 풀 정밀도와 동등한 성능을 기록하였다.

## A.2 NPU 구현 요건

### A.2.1 하드웨어 (NPU HW)

|   |   |
|---|---|
|**요소**|**상세 내용**|
|**혼합 정밀도 데이터패스**|3-bit, 4-bit, 8-bit, FP16 혼용 처리를 단일 MAC 어레이 또는 전용 혼합 정밀도 경로로 지원해야 함. 표준 INT8/FP16 NPU에 3-bit 디코더 회로를 추가하는 방식이 현실적|
|**벡터 회전 보조 유닛 (권장)**|PolarQuant의 random rotation matrix 연산(GEMV)을 NPU 내부에서 처리하면 CPU 오프로드 불필요. Hadamard Transform 하드웨어 유닛 또는 고속 GEMV 경로 추가 권장|
|**압축 KV Cache 전용 버퍼**|3-bit 압축 KV 데이터를 저장하는 on-chip SRAM 영역 및 압축/복원 DMA 엔진. 압축 캐시는 FP16 대비 ~5x 작으므로 on-chip 적재율 대폭 향상|
|**Lookup Table (LUT)** **지원**|극소 비트 역양자화 시 LUT 기반 매핑이 필요. NPU 내 소형 LUT SRAM(4-8KB) 및 해당 데이터패스 연결|
|**Long-context** **처리 (시퀀스 길이)**|TurboQuant의 효과는 긴 컨텍스트(32K~128K 토큰)에서 극대화됨. NPU attention 엔진이 이 범위를 지원할 수 있도록 메모리 주소 공간 및 버퍼 크기 설계 필요|

### A.2.2 소프트웨어 (NPU SW)

•     NPU 컴파일러 커스텀 Op 지원: TurboQuant 압축 attention 연산을 컴파일러 레벨에서 단일 Op으로 fusion하는 최적화 패스 필요

•     Mixed-precision 스케줄러: 3-bit KV 캐시 연산과 FP16 연산이 혼재하는 그래프를 올바르게 분할·스케줄링하는 컴파일러 단계 추가

•     Python/C++ 양자화 라이브러리: 모델 배포 전 PolarQuant 사전 양자화 적용을 위한 도구 체인 (TFLite/ONNX 변환 파이프라인에 통합)

•     드라이버 레벨 KV 버퍼 관리: CPU 드라이버가 압축 KV 캐시 버퍼 생명주기(할당/해제/교체)를 NPU와 협력하여 관리하는 API 설계

## A.3 CPU 구현 요건

### A.3.1 하드웨어 (CPU HW)

|   |   |
|---|---|
|**요소**|**상세 내용 및 후보 적합성**|
|**SIMD** **폭 및 FP16/BF16 지원**|PolarQuant random rotation GEMV는 FP16/BF16 SIMD 연산 집약적. A550 SVE2(가변폭 벡터) ◎ / A73 NEON(FP16 제한) △ / RVV1.0 ○|
|**Bit-manipulation** **명령어**|QJL의 sign bit 추출, 1-bit 벡터 pack/unpack 연산. ARMv9 NEON/SVE2 bitfield 명령어 ○ / Helium MVE ○ / RVV1.0 ○|
|**L2** **캐시 크기 (1MB 이상 권장)**|PolarQuant quantization 상수 및 rotation 행렬은 반복 재사용됨. 캐시 히트율이 처리 효율 결정. A550 ○ / A73 △ / M시리즈 ✕(L2 없음)|
|**INT4** **역양자화 처리 경로**|3-bit→FP16 역양자화를 CPU가 담당하는 경우, 빠른 table lookup + SIMD scatter/gather 명령이 성능 결정 요소. A550 SVE2 gather ◎|
|**메모리 대역폭 (CPU↔DRAM)**|압축 KV 캐시가 줄었더라도, 긴 시퀀스 처리 시 다수의 캐시 블록을 고속 로딩해야 함. LPDDR5 채널 독립 접근이 가능한 CPU 인터페이스 구조 필요|

### A.3.2 소프트웨어 (CPU SW)

•     rotation_matrix_apply() 커널: 고차원 벡터 × rotation matrix GEMV. NEON/SVE2 intrinsics 또는 RVV 어셈블리로 구현. 참조 구현: scipy/numpy 기반 Python 프로토타입 → C intrinsics 이식

•     sign_bit_pack() / unpack() 커널: 부동소수점 벡터 → 1-bit sign 벡터 pack, 역 unpack. SIMD shift+and 명령 활용

•     dequantize_kvcache_3bit(): 3-bit 압축 KV Cache → FP16 복원. LUT 기반 또는 산술 기반 구현, XNNPACK/GGML 양자화 커널 확장 형태로 구현

•     KV Cache 메모리 관리자: 압축 캐시 블록의 할당·해제·LRU eviction 정책. Linux user-space 또는 kernel driver 형태로 구현

•     벤치마크: LongBench / RULER 기반 end-to-end 정확도 측정 + 처리 시간 프로파일링 (CPU/NPU 각 단계 분리 측정 필수)

## A.4 TurboQuant 지원 구현 로드맵 (제안)

|             |                     |                                                              |                 |
| ----------- | ------------------- | ------------------------------------------------------------ | --------------- |
| **단계**      | **시점**              | **목표**                                                       | **담당**          |
| **1****단계** | Prototype 보드 단계     | CPU FP16 SIMD로 PolarQuant/QJL 커널 구현, 성능 측정                   | SW 아키텍처팀        |
| **2****단계** | NPU 설계 RTL Freeze 전 | NPU 내 Hadamard/rotation 유닛 추가 여부 결정, 혼합 정밀도 경로 설계            | NPU HW팀 + 컴파일러팀 |
| **3****단계** | SW 통합 검증 단계         | TFLite/ONNX RT 플러그인으로 TurboQuant Op 등록, LongBench 정확도 기준값 달성 | ML SW팀          |
| **4****단계** | 양산 SW 릴리즈 전         | TurboQuant 지원 모델 라인업 확정, TV AI 기능과 연계한 시연 완료                 | 제품 기획 + ML SW팀  |