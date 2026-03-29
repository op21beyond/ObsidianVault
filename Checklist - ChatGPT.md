---
tags:
  - CPU선정
---

# 📋 **CPU 선정용 Vendor 요구 체크리스트 (최종)**

## 1. CPU 기본 성능 & 마이크로아키텍처 데이터

| 카테고리    | 요구 항목                        | 요구 데이터 (정량)                     | 측정 방법 / 툴          | 제출 형식    |
| ------- | ---------------------------- | ------------------------------- | ------------------ | -------- |
| Core 성능 | IPC (Instructions per Cycle) | SPECint2017 IPC, SPECfp2017 IPC | SPEC CPU2017       | 리포트 + 로그 |
| 기본 성능   | CoreMark/MHz                 | ≥ 5 CoreMark/MHz                | CoreMark           | 로그       |
| 메모리 영향  | Memory stall %               | Load/store stall 비율 (%)         | perf stat          | 프로파일     |
| 분기 성능   | Branch misprediction rate    | < 5%                            | perf, branch trace | 로그       |
| OoO 성능  | ROB size / issue width       | 구조 설명 + 수치                      | HW spec            | 문서       |

---

## 2. NPU Control Plane 성능 (핵심 ⭐)

|카테고리|요구 항목|요구 데이터|측정 방법|제출 형식|
|---|---|---|---|---|
|인터럽트|IRQ latency|avg / max (μs) < 1μs|cyclictest|로그|
|NPU 제어|enqueue latency|< 5 μs|vendor microbench|코드 + 로그|
|DMA|descriptor 처리 시간|ns 단위|DMA profiling|로그|
|PCIe|round-trip latency|μs|PCIe loopback test|로그|
|스케줄링|multi-queue latency|queue depth별 latency|custom test|리포트|

---

## 3. Heterogeneous AI 성능 (가장 중요 ⭐⭐⭐⭐⭐)

|카테고리|요구 항목|요구 데이터|벤치마크|제출 형식|
|---|---|---|---|---|
|Transformer|attention latency|per-layer latency (ms)|llama.cpp|로그|
|KV cache|KV cache 처리 시간|read/write latency|custom + llama.cpp|로그|
|Softmax|softmax throughput|GFLOPS|oneDNN|로그|
|LayerNorm|latency|μs|oneDNN / XNNPACK|로그|
|SIMD 활용|vector utilization|%|perf / VTune|리포트|
|hybrid 실행|CPU 개입 비율|% (전체 inference 대비)|TVM profiling|리포트|

---

## 4. CPU 직접 추론 성능 (Small Model)

|카테고리|요구 항목|요구 데이터|벤치마크|제출 형식|
|---|---|---|---|---|
|CNN|MobileNet-V2 INT8|latency (ms) < 5ms|MLPerf Tiny v1.1|로그|
|Vision|ResNet-50|images/sec|ONNX Runtime|로그|
|NLP|BERT-tiny|latency (ms)|ONNX Runtime|로그|
|음성|KWS|latency < 2ms|MLPerf Tiny|로그|
|경량 LLM|token/sec|≥ 15 token/s|llama.cpp|로그|

---

## 5. 메모리 & 데이터 이동

|카테고리|요구 항목|요구 데이터|벤치마크|제출 형식|
|---|---|---|---|---|
|대역폭|DRAM bandwidth|GB/s (>20GB/s)|STREAM|로그|
|캐시|L1/L2 hit rate|%|perf|로그|
|coherency|CPU-NPU latency|ns|custom|리포트|
|DMA|zero-copy 성능|throughput|DMA-buf test|로그|

---

## 6. SW 에코시스템 지원 (정성 + 정량)

|카테고리|항목|요구 내용|검증 방법|제출|
|---|---|---|---|---|
|OS|Linux support|mainline kernel 버전|boot log|로그|
|RTOS|Zephyr / FreeRTOS|IRQ latency|cyclictest|로그|
|컴파일러|GCC / LLVM|version + optimization flag|build test|리포트|
|프레임워크|TensorFlow Lite|delegate 지원 여부|모델 실행|로그|
||ONNX Runtime|EP 지원|실행|로그|
||Apache TVM|codegen 가능 여부|build + run|로그|
|라이브러리|Arm Compute Library|지원 여부|test|리포트|
||XNNPACK|vector kernel 성능|benchmark|로그|

---

## 7. 전력 / 열 / 면적 (PPA)

|카테고리|요구 항목|요구 데이터|측정 방법|제출|
|---|---|---|---|---|
|전력|CPU power (active)|W|RTL power model|리포트|
|전력|idle power|mW|silicon / sim|로그|
|효율|perf/W|score/W|benchmark 기반|리포트|
|열|junction temp|°C|thermal sim|리포트|
|면적|die area|mm²|layout|문서|

---

## 8. TurboQuant 대응 능력 (미래성 ⭐)

|카테고리|요구 항목|요구 데이터|측정 방법|제출|
|---|---|---|---|---|
|압축|INT4/INT3 처리|latency|custom kernel|로그|
|SIMD|BF16/FP16 성능|GFLOPS|oneDNN|로그|
|bit ops|bit manipulation 성능|GB/s|custom|로그|
|KV cache|압축 cache 처리|latency|llama.cpp modified|로그|

---

## 9. 개발 리스크 / 생산성 (정성)

|카테고리|항목|요구 내용|제출|
|---|---|---|---|
|개발 기간|SW bring-up|주 단위 일정|계획서|
|드라이버|NPU driver|개발 난이도|리포트|
|레퍼런스|기존 사례|양산 사례 수|리스트|
|지원|vendor 지원|SLA / 엔지니어 수|계약서|
|커뮤니티|ecosystem|GitHub activity|리포트|

---

# 📊 최종 의사결정용 핵심 KPI (딱 5개만 보면 됨)

벤더에게 반드시 요구해야 하는 핵심 지표:

1. **NPU enqueue latency (μs)**
2. **CPU 개입 비율 (% of inference time)**
3. **Transformer token/sec (llama.cpp 기준)**
4. **Perf/W (MLPerf Tiny 기준)**
5. **Porting 기간 (TVM + ONNX)**