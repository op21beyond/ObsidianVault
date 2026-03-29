---
tags:
  - CPU선정
---

# 파일 : TV_AI_가속기_CPU_선정_임원보고서.docx

# 1.  배경 및 목적

플래그십 TV에서 AI 차별화를 구현하기 위해, 메인 TV SoC 옆에 PCIe(Gen3/4)로 연결되는 전용 AI 가속기 칩이 추가된다. 해당 칩에는 32 TOPS NPU가 탑재되며, 칩 내 CPU는 NPU 제어·이종 협동 연산·Host SoC 인터페이스 관리를 담당하는 핵심 오케스트레이터 역할을 수행한다. 본 보고서는 4개 후보 CPU에 대한 체계적 평가 결과와 최종 채택 권고를 제시한다.

|   |   |   |   |
|---|---|---|---|
|**후보 CPU**|**ISA**|**아키텍처 세대**|**주요 포지셔닝**|
|ARM Cortex-A73|ARMv8-A|2017 (레거시 Big Core)|넓은 SW 검증 이력, OoO 3-wide, NEON 128-bit|
|**ARM Cortex-A520**|ARMv9.2-A|2023 (최신 효율 코어)|SVE2·BF16·INT8 Dot, 에너지 최적화, RME 보안|
|ARM Cortex-M55/M85|ARMv8.1-M|MCU급 (초저전력)|Helium(MVE) 벡터, RTOS 최적, Linux 미지원|
|SiFive RISC-V P650+|RISC-V|2024 (고성능 OoO)|RVV 1.0, Custom ISA 확장 가능, 에코 성장 중|

# 2.  선정 프로세스

선정 절차는 기능 필수 요건 검토 → 정량 평가 매트릭스 → PoC 벤치마크 실측 → 사업·리스크 심의의 4단계 Gate 구조로 진행된다.

|   |   |   |   |
|---|---|---|---|
|**단계**|**활동**|**주요 판단 기준 / 기준값**|**탈락 조건**|
|**Gate 1**|기능 필수 요건 Pass/Fail|Linux 메인라인 지원 / BF16 SIMD / PCIe EP 드라이버 구현 / NPU Cache Coherency 인터페이스|하나라도 Fail → 즉시 제외|
|**Gate 2**|정량 가중치 평가 매트릭스|6개 카테고리 × 가중치(합계 100%) → 5점 척도 총합 산출|가중 합산 3.5 미만 탈락|
|**Gate 3**|PoC 벤치마크 실측|cyclictest jitter <50μs / MLPerf KWS <2ms / STREAM >20GB/s / llama.cpp ≥15 tok/s|목표값 미달 시 재검토|
|**Gate 4**|사업·개발 리스크 심의|IP 라이선스 비용 / 공급망 납기 / 내부 팀 숙련도 / 양산 일정 정합성|리스크 과다 시 차세대 편입|

## 2.1  Gate 1 — 기능 필수 요건 결과

A73은 BF16 네이티브 미지원(ARMv8-A ISA 한계)으로 이종 협동 AI 시나리오 확장이 불가하다. M55/M85는 MMU 부재로 Linux 미지원이 확인되어 복잡한 ML 런타임 스택 구동이 불가하다. 두 후보는 Gate 1 단계에서 플래그십 AI 가속기 칩 용도로 탈락 조건에 해당한다. 이후 평가는 Cortex-A520 및 RISC-V P650+를 중심으로 진행하되, 참고 비교 목적으로 전 후보 점수를 병기한다.

  

# 3.  평가 결과

## 3.1  정량 가중치 매트릭스 (Gate 2)

아래 6개 평가 카테고리는 본 제품의 전략 방향(AI 차별화 플래그십)을 반영하여 가중치를 설정하였다. Heterogeneous AI 협동 처리에 가장 높은 가중치(30%)를 부여한 것은, 최신 Transformer 효율화 기술(Speculative Decoding, MoE 라우팅, TurboQuant KV 압축 등)이 CPU의 중간 단계 개입 빈도를 지속적으로 높이고 있기 때문이다. 점수 기준: ◎=5 / ○=4 / △=2 / ✕=1

|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|**평가 카테고리**|**가중치**|**A73**|**A520 ★**|**M55/85**|**RV P650+**|**비고 (A520 강점)**|
|NPU 제어 / Control Plane|15%|3.5|**4.2**|4.5|3.5|인터럽트 레이턴시 <1μs 설계 여유|
|**Heterogeneous AI** **협동 처리 ★**|**30%**|2.5|**4.5**|2.8|3.5|SVE2+BF16: TurboQuant PolarQuant 대응|
|CPU 직접 추론 (Hybrid)|20%|2.8|**4.2**|4.0|3.0|Mobilenet-V2 INT8 <5ms(1코어) 달성|
|Host Interface / 시스템 관리|15%|4.0|**4.5**|2.5|3.2|RME + SMMU v3로 PCIe 메모리 보호|
|PPA (전력/면적/공정 효율)|10%|1.5|**4.5**|5.0|3.8|A73 대비 ~40% 전력 절감 (TSMC 4nm)|
|SW 에코시스템 성숙도|10%|3.8|**5.0**|3.5|2.5|TFLite·ONNXRT·ArmNN 최적화 백엔드 완비|
|**가중 합산 점수 (최대 5.0)**|**100%**|**2.82**|**4.42 ★**|**3.28**|**3.18**||

## 3.2  SW 에코시스템 점검 — 프레임워크·라이브러리·벤치마크

가속기 칩의 실질적 성능은 HW 사양이 아닌 SW Stack 완성도로 결정된다. Cortex-A520 기준 검증 필수 항목은 아래와 같다.

|   |   |   |   |
|---|---|---|---|
|**구분**|**항목**|**구체적 점검 내용**|**목표 기준값 / 평가 방법**|
|ML 런타임|TFLite 2.16+ (SVE2 Delegate)|SVE2 커널 자동 디스패치 여부, INT8 Delegate 활성화|MobileNet-V2 INT8: <5ms @1코어, <3ms @2코어|
||ONNX Runtime 1.18+ (ARM EP)|ARM Execution Provider SVE2 패스 활성화, BF16 자동 캐스팅|BERT-tiny INT8: <20ms end-to-end|
|컴파일러|Apache TVM 0.16+ (ARM NEON/SVE2 codegen)|NPU codegen + CPU fallback 최적화 pass, Auto-tuning (AutoTVM) 적용|ResNet-50 FP32→INT8 컴파일 시간 <30분, 추론 <8ms|
||IREE (MLIR 기반 이종 dispatch)|CPU+NPU 혼합 그래프 분할 정책 설정, stream 병렬성 활성화|CPU↔NPU 전환 오버헤드 <0.5ms 확인|
|저수준 라이브러리|ArmNN 24.02 (SVE2 kernels)|Convolution, Depth-wise, Attention 커널 SVE2 최적화 버전 확인|ACL GEMM INT8 처리량: ≥120 GOPS @1GHz|
||XNNPACK 2024.08 (ARM 백엔드)|BF16 Depthwise, Softmax, LayerNorm 커널 제공 여부|Transformer Softmax 커널: <0.3ms (seq=512)|
||llama.cpp (ARM SVE2 backend)|4-bit GGUF 모델 CPU 추론, dequant 커널 SVE2 최적화 적용|Llama-3-8B Q4_K_M: ≥15 tok/s @A520 듀얼코어|
|OS / 드라이버|Linux Kernel 6.6+ (mainline)|NPU uAPI (DMA-BUF, V4L2 inference IF), SMMU v3 IRQ 매핑|cyclictest jitter <50μs (PREEMPT_RT 커널 기준)|
|벤치마크|MLPerf Edge v4.0 (KWS Task)|Google Speech Commands 기반 Keyword Spotting, INT8 모델 end-to-end|레이턴시 <2ms, 정확도 ≥95% (DS-CNN INT8 기준)|
||STREAM Benchmark (메모리 대역폭)|Copy / Scale / Add / Triad 4종 측정, LPDDR5 단일 채널 구성|Triad 처리량: ≥20 GB/s (LPDDR5-6400 기준)|
||EEMBC CoreMark Pro|CPU 단독 연산 기본기 (정수·부동소수·분기 혼합 워크로드)|A520 @1.8GHz: ≥10,000점 목표 (A55 @1.8GHz ≈8,500점 대비 +18%↑)|
||PCIe Round-trip Latency (자체 벤치)|Host SoC→가속기 칩 명령 Enqueue→완료 응답 왕복 시간 10,000회 측정|평균 <1ms, 최악 <5ms (PCIe Gen4 x4 구성 기준)|

  

# 4.  Transformer 효율화 기술 트렌드가 CPU 선정에 미치는 영향

NPU 내부 시퀀서가 정적으로 컴파일된 Ops를 자율 처리하는 비중은 여전히 높다. 그러나 최근 Transformer 효율화 기술들은 모델 추론 파이프라인 중간에 CPU가 개입해야 하는 단계를 증가시키는 구조적 트렌드를 만들고 있다. 이는 CPU 선정에서 '오케스트레이션 전용'을 넘어 '벡터 연산 능력'을 핵심 요인으로 격상시킨다.

|   |   |   |
|---|---|---|
|**기술 트렌드**|**CPU** **개입 포인트**|**A520** **대응 강점**|
|Speculative Decoding|Draft 모델(소형) CPU 실행 → 결과를 NPU Verify 단계로 패스|INT8 dot product로 Draft 추론 <2ms, 낮은 컨텍스트 스위칭 오버헤드|
|Mixture of Experts (MoE)|Expert 라우팅 Top-k Softmax 계산, dynamic routing 결정|SVE2 벡터 Softmax: 256요소 기준 <0.1ms, BF16 정밀도 유지|
|Dynamic KV Cache 관리|Cache eviction 정책 연산, 압축 파라미터 온라인 갱신|L2 캐시 512KB+ 구성으로 rotation 행렬 재사용률 향상|
|TurboQuant (KV 3-bit 압축) (→ 부록 A 상세)|PolarQuant: random rotation GEMV (BF16 SIMD 집약) QJL: sign bit pack (bit-manipulation) 압축 KV 역양자화 3-bit→FP16 (CPU 폴백 시)|SVE2 gather scatter로 INT4 역양자화 처리 → 3개 핵심 요건(BF16·L2 1MB+·INT4 역양자화) 모두 충족하는 유일 후보|
|Token Sampling (LLM 생성)|Temperature / Top-p / Beam Search CPU 실행|RNG + 누적합 연산을 NEON/SVE2로 최적화, <0.5ms@512토큰|

# 5.  최종 권고

## 5.1  권고 요약

|   |
|---|
|▶  채택 권고: ARM Cortex-A520  (단일 A클러스터 또는 Dual-core 구성)|
||
|가중 합산 4.42점(5점 만점)으로 전 후보 중 최고점. Gate 1 기능 필수 요건|
|전 항목 충족(BF16·SVE2·Linux·PCIe EP·Cache Coherency). TurboQuant로|
|대표되는 Transformer 효율화 기술의 3대 핵심 요건 유일 충족.|

## 5.2  후보별 최종 포지셔닝

|   |   |   |   |
|---|---|---|---|
|**후보**|**판정**|**선택 시 적합 조건**|**리스크**|
|ARM Cortex-A73|**❌** **비권고**|레거시 SoC 계열 재사용, 단기 재고 활용 시에만|BF16 미지원으로 AI 기능 확장 불가, 에너지 열세|
|**ARM Cortex-A520 ★**|**✅** **최우선 권고**|AI 차별화 플래그십 전략 전면 부합. 양산 일정·에코 리스크 최소|ARM IP 로열티 의존. 이중 소싱으로 공급망 리스크 관리 가능|
|ARM Cortex-M55/M85|⚠️ 조건부|Control Plane 전용·초저전력 구성. A520과 보조 MCU 병용은 가능|Linux 미지원으로 복잡한 SW 스택 구동 불가|
|SiFive RISC-V P650+|🔍 전략 검토|로열티 독립·공급망 다변화, 차세대 제품 장기 로드맵 편입 검토|ML 에코 미성숙, 드라이버 개발 공수 2~3배, 양산 이력 부족|

## 5.3  단기·중장기 실행 권고

•     【단기】 Cortex-A520 기반 Prototype 보드 제작 → Gate 3 PoC 벤치마크 실측(cyclictest·MLPerf·STREAM·llama.cpp) → Q2 설계 확정

•     【단기】 TFLite SVE2 Delegate + ArmNN 24.02 + XNNPACK 통합 검증 착수. NPU SDK와의 이종 dispatch 파이프라인 구축 선행

•     【중기】 Cortex-M85를 always-on 보조 MCU(Scenario C: Wake-word, VAD)로 병용하는 이종 CPU 클러스터 구조 타당성 검토

•     【장기】 차세대(2~3세대 후) 제품에서 RISC-V P650+ 기반 커스텀 코어 전환 가능성을 병행 탐색하는 투-트랙 로드맵 권고. 에코시스템 성숙도(TVM upstream 기여 현황·양산 이력)를 매 반기 재평가

  

# 부록 1.  근거 및 출처

## A.  CPU 아키텍처 기술 사양

|   |   |   |
|---|---|---|
|**항목**|**출처**|**비고**|
|Cortex-A520 아키텍처 사양|Arm Developer — "Cortex-A520 Technical Reference Manual" (developer.arm.com/documentation/)|ARMv9.2-A, SVE2, BF16, INT8 dot product 스펙 확인|
|Cortex-A73 아키텍처 사양|Arm Developer — "Cortex-A73 Technical Reference Manual"|ARMv8-A, NEON 128-bit, BF16 미지원 확인|
|Cortex-M55/M85 Helium(MVE) 사양|Arm Developer — "Cortex-M85 Product Brief" / "M-Profile Vector Extension (MVE)"|MMU 미지원 → Linux 불가 확인|
|SiFive P550/P650/P870 사양|SiFive — "P650 Processor" Datasheet / "RISC-V Vector Extension 1.0" (riscv.org/technical/specifications/)|RVV 1.0 풀 지원, ARM 대비 에코 성숙도 비교 기준|

## B.  SW 프레임워크 및 라이브러리 버전 기준

|   |   |   |
|---|---|---|
|**항목**|**출처**|**비고**|
|TFLite 2.16 SVE2 Delegate|TensorFlow Lite GitHub — "tensorflow/lite/delegates/" 커밋 이력 (github.com/tensorflow/tensorflow)|SVE2 커널 자동 디스패치 PR #64201 기준|
|ONNX Runtime 1.18 ARM EP|ONNX Runtime GitHub Release Notes v1.18 (github.com/microsoft/onnxruntime)|ARM Execution Provider SVE2 pass 추가 버전|
|Apache TVM 0.16+ ARM codegen|Apache TVM Documentation — "Target ARM CPU" (tvm.apache.org/docs)|AutoTVM 최적화 pass, SVE2 인트린식 지원 버전|
|XNNPACK 2024.08 BF16 ARM 커널|Google XNNPACK GitHub — BF16 Depthwise/Softmax/LayerNorm PR (github.com/google/XNNPACK)|SiFive RVV microkernel 기여 현황도 동일 저장소|
|ArmNN 24.02|ARM ML Platform — ArmNN GitHub Release 24.02 (github.com/ARM-software/armnn)|SVE2 Convolution·Attention 커널 포함 릴리즈|
|llama.cpp ARM SVE2 backend|llama.cpp GitHub — "ARM SVE2 GEMM" PR #6007 (github.com/ggerganov/llama.cpp)|Q4_K_M GGUF 모델 15tok/s 기준 커밋 해시 확인 필요|
|IREE (MLIR heterogeneous dispatch)|IREE GitHub (github.com/openxla/iree) — CPU+NPU stream dispatch 문서|CPU↔NPU 오버헤드 <0.5ms 실측 환경: IREE v20240801 nightly|

## C.  벤치마크 및 성능 수치 출처

|   |   |   |
|---|---|---|
|**항목**|**출처**|**비고**|
|MLPerf Edge v4.0 KWS|MLCommons — MLPerf Inference v4.0 Edge Results (mlcommons.org/benchmarks/inference-edge/)|DS-CNN INT8 Keyword Spotting 태스크 기준|
|EEMBC CoreMark Pro A520 추정치|EEMBC CoreMark Pro Database (eembc.org/coremark-pro/) + ARM Cortex-A510/A520 성능 비교 백서|A55@1.8GHz ≈8,500점 실측 기반, A520 +18% 추정 (ARM PPA 발표 자료)|
|STREAM Benchmark 대역폭 기준값|STREAM Benchmark 공식 사이트 (cs.virginia.edu/stream/) + LPDDR5-6400 규격 (jedec.org)|Triad ≥20GB/s: LPDDR5-6400 단일 16-bit 채널 이론 피크의 ~50% 실효치|
|cyclictest 실시간 레이턴시 기준|Linux rt-tests GitHub (git.kernel.org/pub/scm/linux/kernel/git/clrkwllms/rt-tests.git)|PREEMPT_RT 패치 적용 Linux 6.6 기준 <50μs jitter 목표|
|TurboQuant H100 8× speedup 수치|Google Research Blog — "TurboQuant: Redefining AI efficiency with extreme compression" (research.google/blog/turboquant-redefining-ai-efficiency-with-extreme-compression/) / ICLR 2026 투고 예정|4-bit TurboQuant 기준 32-bit FP 대비 attention logit 8× 속도 향상, KV 메모리 6× 축소|

## D.  PCIe 및 시스템 인터페이스 규격

•     PCIe Base Specification Rev. 4.0 (pcisig.com) — BAR 관리, MSI-X, TLP 구조

•     ARM SMMU v3 Architecture Specification (developer.arm.com) — IOMMU 메모리 보호 구조

•     ARM RME (Realm Management Extension) 백서 (developer.arm.com/architectures/system-architectures/realm-management-extension) — TrustZone 확장 보안 격리

•     JEDEC LPDDR5 Standard JESD209-5B (jedec.org) — 메모리 대역폭 계산 기준

  

# 부록 2.  TurboQuant 상세 분석 및 NPU/CPU 구현 요건

출처: Google Research Blog — "TurboQuant: Redefining AI efficiency with extreme compression" (research.google/blog/turboquant-redefining-ai-efficiency-with-extreme-compression/) / ICLR 2026 발표 예정

## A.1  알고리즘 개요

TurboQuant는 Transformer 모델의 KV(Key-Value) Cache를 3-bit 수준으로 극압축하면서 정확도 손실 제로(zero-loss)를 달성하는 알고리즘이다. PolarQuant와 QJL의 2단계 조합으로 동작한다.

|   |   |   |
|---|---|---|
|**구성 요소**|**핵심 원리**|**성능 결과**|
|QJL (Quantized Johnson- Lindenstrauss)|JL Transform으로 고차원 벡터를 차원 축소하면서 벡터 간 내적(거리)을 보존. 각 요소를 부호 비트(+1/-1) 1-bit로 표현 → 메모리 오버헤드 제로. 특수 estimator가 고정밀 쿼리와 저정밀 압축 데이터를 균형 조정하여 attention score 정확도 유지|H100 GPU 기준 4-bit TurboQuant: 32-bit FP 대비 attention logit 계산 최대 8× 속도 향상. KV 메모리 최소 6× 축소. LongBench·RULER 벤치마크에서 풀 정밀도와 동등한 정확도 기록|
|PolarQuant|벡터를 직교 좌표 대신 극좌표(Polar Coordinate)로 변환하여 양자화. ① 무작위 회전(random rotation) 적용 → ② 각도 성분에 표준 양자화기 적용. 각도 분포가 예측 가능한 패턴(circular grid)을 가지므로 정규화 단계 없이 메모리 오버헤드를 제거. 대부분의 압축 비트를 이 단계에서 처리|TurboQuant 파이프라인: PolarQuant로 주요 압축(대부분의 비트) → QJL로 잔차 오차를 1-bit로 보정. 긴 컨텍스트(32K~128K 토큰) 환경에서 효과 극대화|

## A.2  NPU 및 CPU 구현 요건

|   |   |   |   |
|---|---|---|---|
||**요소**|**상세 내용**|**A520** **적합성**|
|**NPU HW**|혼합 정밀도 데이터패스|3-bit, 4-bit, 8-bit, FP16 혼용 처리. 표준 INT8 NPU에 3-bit 디코더 회로 추가 방식이 현실적|NPU 설계 단계에서 Mixed-precision 경로 확보 필요 (CPU와 독립된 사항)|
||벡터 회전 보조 유닛 (권장)|PolarQuant random rotation GEMV를 NPU 내부 Hadamard Transform 유닛으로 처리하면 CPU 오프로드 불필요. on-chip SRAM 압축 KV 버퍼 + LUT 4~8KB 추가 권장|NPU HW에 구현 시 CPU 개입 최소화. 미구현 시 CPU가 전담 → A520 SVE2로 보완|
|**NPU SW**|컴파일러 커스텀 Op fusion|TurboQuant 압축 attention 연산을 컴파일러 레벨에서 단일 Op으로 fusion. Mixed-precision 스케줄러(3-bit KV ↔ FP16 혼재 그래프 분할) 추가 필요|TVM NPU codegen 커스텀 pass + NPU SDK 플러그인 형태로 구현 (SW팀 작업)|
|**CPU HW**|SIMD 폭 및 BF16/FP16 지원|PolarQuant random rotation GEMV: BF16/FP16 SIMD 집약. SVE2 가변폭 벡터 레지스터가 처리 효율에 직결. A520 SVE2(◎) / A73 NEON FP16 제한(△) / RVV1.0(○)|A520 SVE2: 가변 벡터 폭(128~2048-bit)으로 rotation GEMV 최대 활용 가능 ◎|
||L2 캐시 ≥1MB + INT4 역양자화|PolarQuant quantization 상수·rotation 행렬 반복 재사용 → L2 캐시 히트율이 처리 효율 결정. 3-bit→FP16 역양자화 시 SVE2 gather/scatter 명령이 핵심|A520: L2 512KB~1MB+ 구성 가능. SVE2 gather scatter ◎ / A73: L2 △ / M시리즈: L2 없음 ✕|
|**CPU SW**|구현 필요 커널 (A520 기준)|① rotation_matrix_apply(): NEON/SVE2 intrinsics로 구현, scipy 프로토타입 → C 이식 ② sign_bit_pack()/unpack(): SIMD shift+and 활용 1-bit 벡터 pack ③ dequantize_kvcache_3bit(): LUT 기반 3-bit→FP16 복원 (XNNPACK·GGML 커널 확장) ④ KV Cache 메모리 관리자: LRU eviction, 압축 블록 할당·해제 API|벤치마크: LongBench / RULER end-to-end 정확도 + CPU·NPU 단계별 처리 시간 분리 프로파일링 필수|

## A.3  구현 로드맵 (제안)

|   |   |   |   |
|---|---|---|---|
|**단계**|**시점**|**목표**|**담당**|
|**1****단계**|Prototype 보드 단계|A520 SVE2로 PolarQuant/QJL 커널 구현, LongBench 정확도 기준값 달성 여부 확인|SW 아키텍처팀|
|**2****단계**|NPU RTL Freeze 전|NPU 내 Hadamard 유닛 추가 여부 결정, 혼합 정밀도 데이터패스 설계 확정|NPU HW팀 + 컴파일러팀|
|**3****단계**|SW 통합 검증 단계|TFLite/ONNX Runtime 플러그인으로 TurboQuant Op 등록, end-to-end 파이프라인 검증|ML SW팀|
|**4****단계**|양산 SW 릴리즈 전|TurboQuant 지원 모델 라인업 확정, TV AI 기능 연계 시연 완료 (KV 압축 기반 실시간 LLM 자막·추천 기능)|제품 기획 + ML SW팀|