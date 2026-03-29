---
tags:
  - CPU선정
---

# **[보고서] 플래그십 TV AI 가속기 칩 CPU 선정 최종안**

---

# 1. 배경 (간략)

본 과제는 메인 TV SoC와 PCIe로 연결되는 **32 TOPS NPU 기반 AI 가속기 칩**의 CPU Subsystem을 선정하는 것으로, 단순 제어용 CPU가 아닌 **NPU 중심 이종 컴퓨팅 시스템의 성능을 결정하는 핵심 요소**이다.

기존 구조 대비 차별점은 다음과 같다.

- NPU 내부 scheduler가 대부분의 연산을 수행 → CPU 개입 최소
- 그러나 최근 Transformer 기반 AI에서는
    - dynamic execution
    - KV cache 처리
    - quantization/압축 처리  
        등 **CPU 개입 비중 증가**

따라서 CPU는 단순 제어기가 아닌  
👉 **“AI Data Orchestrator + Hybrid Compute Engine”**로 정의된다

---

# 2. 목적 (간략)

- 플래그십 TV AI 차별화를 위한 **최적 CPU 아키텍처 선정**
- 성능 / 전력 / 비용 / SW 리스크를 모두 반영한 **정량적 의사결정 체계 수립**
- 향후 Transformer 및 극한 압축 기술(TurboQuant 등)에 대한 **미래 대응성 확보**

---

# 3. CPU 선정 프로세스 (통합 모델)

본 보고서는 4개 조사 내용을 통합하여 아래 **3단계 의사결정 구조**로 정리하였다.

---

## 3.1 Step 1 — 역할 기반 요구 정의

CPU는 다음 4가지 시나리오를 모두 만족해야 한다.

### (1) NPU Control Plane

- enqueue/dequeue latency: **< 5 μs**
- interrupt handling: **< 1 μs**
- multi-queue scheduling

### (2) Heterogeneous AI (핵심 증가 영역)

- Transformer 중간 연산:
    - Softmax
    - LayerNorm
    - KV cache 관리
- Dynamic routing / MoE
- Token sampling

👉 CPU 성능 요구 급증 영역

---

### (3) CPU 직접 추론 (경량 모델)

- MobileNet-V2 INT8: **< 5ms**
- Wake-word / VAD always-on

---

### (4) Host SoC 인터페이스

- PCIe Gen4 기반 DMA
- MSI/MSI-X interrupt
- shared memory coherence

---

## 3.2 Step 2 — 평가 체크리스트 (핵심 항목)

### HW 관점

- SIMD 성능 (NEON / SVE2 / RVV)
- BF16 / INT8 dot product 지원
- L2 cache ≥ 1MB
- cache coherency (ACE/CHI)
- interrupt latency
- power efficiency (DMIPS/mW)

---

### SW 관점 (매우 중요)

#### 프레임워크

- TensorFlow Lite
- ONNX Runtime
- Apache TVM
- ExecuTorch

#### 라이브러리

- Arm Compute Library (ACL)
- CMSIS-NN
- XNNPACK
- oneDNN

#### 필수 기능

- heterogeneous execution
- graph partitioning
- custom op 지원
- quantization kernel

---

## 3.3 Step 3 — 정량 평가 방법

### 벤치마크 구성

|항목|벤치마크|목표|
|---|---|---|
|CPU 기본 성능|CoreMark|>5 CoreMark/MHz|
|ML 성능|MLPerf Tiny|KWS < 2ms|
|Transformer|llama.cpp|≥15 token/s|
|실시간성|cyclictest|<50μs jitter|
|메모리|STREAM|>20 GB/s|

👉 CPU 단독이 아닌 **CPU+NPU 혼합 성능 측정 필수**

---

# 4. 후보 CPU 종합 평가

후보:

- Cortex-A73
- Cortex-A520
- Cortex-M55
- SiFive P550 후속

---

## 4.1 핵심 비교 요약

### Cortex-A73

- 장점: 높은 single-thread 성능, 검증된 SW
- 단점:
    - BF16 미지원
    - 전력 비효율
    - 구세대 구조

👉 플래그십 AI 기준 **탈락 수준**

---

### Cortex-A520 (핵심 후보)

- ARMv9 기반 효율 코어
- SVE2 / BF16 / INT8 dot product 지원
- 높은 perf/W

👉 특징:

- Transformer hybrid 처리 대응 가능
- SW ecosystem 최강

👉 종합:

- **성능 / 전력 / SW 균형 최적**

---

### Cortex-M55/M85

- 장점:
    - ultra-low power
    - real-time 최고
- 단점:
    - Linux 불가
    - 대형 AI 처리 불가

👉 단독 사용 불가 (보조 코어로는 우수)

---

### SiFive P550 후속

- 장점:
    - RVV vector → Transformer 최적
    - custom ISA 가능
    - 비용 경쟁력
- 단점:
    - SW ecosystem 미성숙
    - 개발 리스크 높음

👉 중장기 전략용

---

## 4.2 정량 평가 결과 (통합)

|항목|A73|A520|M55|P550|
|---|---|---|---|---|
|NPU 제어|3.5|4.5|4.5|3.5|
|Heterogeneous AI|2.5|4.5|2.8|3.5|
|CPU 추론|2.8|4.2|4.0|3.0|
|시스템|4.0|4.5|2.5|3.2|
|PPA|1.5|4.5|5.0|3.8|
|SW|3.8|5.0|3.5|2.5|

👉 총점:

- **A520: 4.4 / 5 (최고)**
- M55: 3.3
- P550: 3.2
- A73: 2.8

---

# 5. 최종 결과 및 권고

## 5.1 최종 선택

👉 **Cortex-A520 기반 CPU 채택**

---

## 5.2 권장 구성

### 권장 아키텍처

- A520 Dual 또는 Quad
- - optional M55 (always-on / RT)

---

## 5.3 선정 근거 (핵심 4가지)

### ① AI 구조 변화 대응

- Transformer hybrid execution 대응 가능
- BF16 / SVE2 지원 필수 조건 충족

---

### ② 전력/면적 최적

- NPU 중심 구조에서 CPU는 **효율이 핵심**
- perf/W 최고 수준

---

### ③ SW 리스크 최소

- TensorFlow Lite / ONNX / TVM 완전 지원
- Arm ecosystem 안정성 확보

---

### ④ TurboQuant 등 미래 기술 대응

- FP16/BF16 SIMD
- INT4/INT8 처리 가능
- cache 구조 적합

---

## 5.4 비권고 및 조건부 전략

- A73 → **즉시 제외**
- M-series → 보조 코어로만 사용
- RISC-V → 차세대 제품군에서 검토

---

# 6. 실행 계획

1. A520 기반 prototype
2. MLPerf + transformer benchmark 수행
3. NPU driver + TVM integration
4. 양산 설계 확정

---

# [부록 A] TurboQuant 상세 (요약)

TurboQuant 는

- 3~4bit extreme quantization
- KV cache 6배 이상 감소
- attention 성능 유지

---

## CPU 요구사항

### HW

- SIMD (BF16/FP16)
- bit manipulation
- large L2 cache

---

### SW

- dequant kernel
- rotation kernel
- KV cache manager

---

## 핵심 영향

👉 CPU 개입 증가 → **heterogeneous AI 중요도 상승**

---

# [부록 B] 근거 및 출처

- 내부 조사 문서 4종 통합
    
- MLPerf Tiny v1.1 benchmark 정의
- TensorFlow Lite / ONNX Runtime / TVM 공식 문서
- Google Research TurboQuant 기술 블로그 및 ICLR 2026 발표 내용