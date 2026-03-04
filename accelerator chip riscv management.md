# TV SoC 연결 LLM NPU 가속기 칩 개발 프로젝트
## 시스템 리스크 관리 및 개발 방안

---

## 문서 개요

**프로젝트명**: PCIe Gen4 연결 LLM NPU 가속기 칩 개발  
**연결 방식**: PCIe Gen4 x1 Lane  
**주요 구성요소**:
- NPU x2 (LLM 추론용)
- 4K 화질처리 엔진
- LPDDR5-6400 x32 x 2채널
- HDMI RX (HDCP 포함)
- 선택사항: 최소 CPU + Flash (펌웨어용)

**문서 목적**: 시스템 관점에서 HW 설계부터 펌웨어 개발, 부팅, 응용 운영까지 전 과정의 리스크 식별 및 관리 방안 제시

---

## 1. 리스크 요약 매트릭스

| 리스크 ID     | 리스크 항목                                            | 심각도   | 긴급성   | 개발 단계         | 일차 담당        |
| ---------- | ------------------------------------------------- | ----- | ----- | ------------- | ------------ |
| ~~**R1**~~ | ~~**LPDDR5 PHY Training 실패**~~                    | ~~3~~ | ~~5~~ | ~~HW 설계~~     | ~~HW 설계팀~~   |
| **R2**     | **DRAM QoS 미흡으로 화질엔진 실시간 동작 보장 실패**               | 5     | 5     | HW 설계 / SW 개발 | 시스템 아키텍트     |
| ~~**R3**~~ | ~~**부팅 시퀀스 설계 오류 (PCIe Enumeration 전 DDR 초기화)**~~ | ~~5~~ | ~~4~~ | ~~아키텍처 설계~~   | ~~시스템 아키텍트~~ |
| **R4**     | **가속기 칩 CPU 유무 결정 지연**                            | 4     | 5     | 아키텍처 설계       | 시스템 아키텍트     |
| **R5**     | **PCIe DMA 유효 대역폭 부족**                            | 4     | 4     | HW 설계 / 성능 검증 | HW 설계팀       |
| **R6**     | **메모리 Fragmentation으로 인한 성능 저하**                  | 3     | 3     | SW 개발 / 운영    | SW 개발팀       |
| **R7**     | **2-Rank DRAM 구성 방식 결정 지연**                       | 4     | 4     | 아키텍처 설계       | 시스템 아키텍트     |
| **R8**     | **온도 변화에 따른 DDR 성능 저하**                           | 3     | 3     | HW 설계 / 펌웨어   | HW 설계팀       |
| **R9**     | **부팅 시간 스펙 초과**                                   | 4     | 3     | 시스템 통합        | 펌웨어 개발팀      |
| **R10**    | **HDCP 키 관리 및 보안 취약점**                            | 4     | 3     | SW 개발         | SW 개발팀       |
| **R11**    | **IOMMU 운영 오버헤드**                                 | 3     | 2     | SW 개발         | SW 개발팀       |
| **R12**    | **Host BAR 매핑 주소 공간 < 가속기 메모리 크기**                | 4     | 3     | 아키텍처 설계       | 시스템 아키텍트     |
**심각도/긴급성 등급**: 5 (매우 높음) / 4 (높음) / 3 (중간) / 2 (낮음

---

## 2. 단계별 개발 프로세스 및 주요 리스크

### 2.1 아키텍처 및 사양 확정 단계

#### 목표
- 성능 타겟 설정 및 검증
- PCIe/DRAM 대역폭 시뮬레이션
- IP 통합 설계 및 인터페이스 정의

#### 주요 리스크

##### **R4: 가속기 칩 CPU 유무 결정 지연** [★★★★★ 긴급성]

**리스크 이유**:
- CPU 유무에 따라 전체 아키텍처, 부팅 시퀀스, 메모리 관리 방식이 완전히 달라짐
- 조기 결정 실패 시 후속 설계 변경으로 인한 일정 지연 및 비용 증가

**개발 방안**:

| 방안             | CPU 사양                                      | 장점                                                                  | 단점                                                                              | 권장 시나리오                           |
| -------------- | ------------------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------- | --------------------------------- |
| **A. CPU 없음**  | -                                           | • Die Size 최소화<br>• 소비전력 최소화<br>• 설계 복잡도 감소                         | • Host 의존도 극대화<br>• DDR Training을 Host가 수행 필요<br>• HDCP 키 관리 어려움<br>• 실시간 제어 제약 | 단순 가속 전용<br>Host 성능 충분            |
| **B. 경량 MCU**  | ARM Cortex-M4/M7<br>32KB SRAM<br>64KB Flash | • DDR Training 자체 수행<br>• HDCP 키 관리 가능<br>• 독립적 초기화<br>• Host 부담 감소 | • 추가 Die Area<br>• 펌웨어 개발 필요<br>• 디버깅 복잡도 증가                                    | **[권장]**<br>독립 동작 필요<br>실시간 제어 필요 |
| **C. 고성능 CPU** | ARM Cortex-A53<br>512KB SRAM<br>2MB Flash   | • 완전 독립 동작<br>• 복잡한 알고리즘 실행<br>• 동적 메모리 관리                          | • 상당한 Die Area<br>• 높은 소비전력<br>• 과도한 설계                                         | Over-spec<br>불필요                  |

**권장 방안**: **방안 B (경량 MCU 탑재)**

**근거**:
1. **DDR PHY Training 자체 수행 필수**: LPDDR5-6400의 높은 복잡도로 인해 Host에 의존 시 초기화 실패 리스크 증가
2. **HDCP 키 관리**: HDMI RX의 HDCP 키를 안전하게 관리하려면 독립적인 보안 영역 필요
3. **실시간 QoS 조정**: 화질엔진 동작 중 실시간으로 DRAM 스케줄러 QoS 파라미터 조정 가능
4. **부팅 독립성**: Host SoC 부팅과 병렬로 자체 초기화 가능하여 전체 부팅 시간 단축

**담당 역할**:
- **DDR PHY Training 제어**: PHY 레지스터 설정 및 Training 시퀀스 실행
- **HDCP 키 관리**: HDMI RX HDCP descramble 엔진에 키 공급
- **QoS 동적 조정**: NPU/화질엔진 동작 모드에 따라 DRAM 스케줄러 우선순위 변경
- **온도 센서 모니터링**: Periodic DDR Re-training 트리거
- **펌웨어 업데이트**: Flash에서 펌웨어 로딩 및 업데이트 관리

**CPU 사양 상세**:
```
• Core: ARM Cortex-M7 @ 200MHz
• SRAM: 32KB (펌웨어 실행용)
• Flash: 64KB (부팅 펌웨어 + DDR Training 코드)
• Interface: APB for register access to DDR controller, HDCP engine
• Interrupt Controller: NVIC (NPU/화질엔진 완료 인터럽트 처리)
```

---

##### **R3: 부팅 시퀀스 설계 오류** [★★★★★ 심각도]

**리스크 이유**:
- 가속기 칩은 자체 저장장치가 없으므로 **"PCIe First, DDR Later"** 전략 필수
- 잘못된 순서로 초기화 시 칩 먹통 또는 Host 통신 불가

**부팅 시퀀스 설계안**:

```
[Boot ROM 단계] (가속기 내장 ROM, ~100ms)
  ├─ Power-On-Reset
  ├─ PCIe PHY 초기화
  ├─ PCIe Link Training (Gen4 x1)
  └─ Host에 Endpoint로 노출
         ↓
[PCIe Enumeration] (Host 주도, ~200ms)
  ├─ Host SoC가 가속기 인식
  ├─ BAR0 매핑 (가속기 내부 SRAM)
  ├─ BAR1 매핑 (예약: 향후 DDR 일부 영역)
  └─ Host가 펌웨어를 BAR0를 통해 SRAM에 전송
         ↓
[펌웨어 부팅] (가속기 CPU 실행, ~300ms)
  ├─ SRAM에서 펌웨어 실행 시작
  ├─ **DDR PHY Training** ← **가장 큰 리스크 지점**
  │   ├─ Write Leveling
  │   ├─ Read Leveling
  │   ├─ Gate Training
  │   └─ VREF Calibration
  ├─ DDR 메모리 맵 설정
  └─ Host에 "Ready" 인터럽트 전송 (MSI)
         ↓
[Peripheral 초기화] (~100ms)
  ├─ HDMI RX 초기화
  ├─ HDCP Descramble 엔진 키 로딩
  ├─ 화질엔진 초기화
  └─ NPU 초기화 (모델은 아직 로딩 안 됨)
         ↓
[응용 준비] (시나리오별 상이)
  ├─ Host가 LLM 모델을 PCIe DMA로 전송
  └─ NPU 실행 대기
```

**총 부팅 시간 예상**: ~700ms (최적화 시 ~500ms 가능)

**Critical Path**:
1. **DDR PHY Training** (~200-300ms): LPDDR5-6400의 높은 속도로 인해 Training 실패율 높음
2. **PCIe Link Training**: Gen4는 Gen3 대비 Training 시간 증가 (~50ms 추가)

**리스크 완화 방안**:
- **DDR Training 실패 시 Fallback**: LPDDR5-6400 → LPDDR5-4800로 다운그레이드 후 재시도
- **Training 파라미터 저장**: 성공한 Training 값을 Flash에 저장하여 다음 부팅 시 빠른 초기화
- **온도 센서 기반 조정**: 칩 온도가 Training 시점과 10°C 이상 차이 나면 Re-training 트리거

---

##### **R7: 2-Rank DRAM 구성 방식 결정** [★★★★ 심각도/긴급성]

**리스크 이유**:
- Rank Interleaving vs. 독립 Rank 방식에 따라 메모리 컨트롤러 설계 복잡도 및 성능 차이 큼
- NPU 2개 + 화질엔진의 트래픽 패턴이 불확실하여 최적 구성 판단 어려움

**방안 비교**:

| 구성 방식 | 설계 복잡도 | 대역폭 효율 | 레이턴시 | QoS 구현 난이도 | Die Size |
|-----------|-------------|-------------|----------|-----------------|----------|
| **2-Rank Interleaving** | 높음 | 높음 (90%) | 낮음 | 중간 | 작음 |
| **독립 2-Rank** | 낮음 | 중간 (75%) | 중간 | 쉬움 | 큼 |
| **Hybrid (동적 전환)** | 매우 높음 | 최고 (95%) | 최저 | 어려움 | 중간 |

**분석**:

**2-Rank Interleaving 방식**:
- **장점**:
  - 연속된 주소가 자동으로 2개 Rank에 분산되어 Bank Conflict 감소
  - 이론적 최대 대역폭 활용 가능
  - 하나의 통합 컨트롤러로 구현 가능
- **단점**:
  - 보드 레이아웃 상 2개 x32 DRAM이 멀리 떨어져 있을 경우 Skew 관리 어려움
  - Interleaving 로직으로 인한 약간의 레이턴시 증가
  - QoS 구현 시 Rank별 우선순위 제어 복잡

**독립 2-Rank 방식**:
- **장점**:
  - NPU0 → Rank0, NPU1 → Rank1로 명확히 분리 가능
  - 화질엔진은 두 Rank 모두 접근 가능하게 설정하여 유연성 확보
  - QoS 구현 간단 (Rank별 독립 스케줄러)
  - 보드 레이아웃 제약 없음
- **단점**:
  - NPU 트래픽이 한쪽 Rank에 집중될 경우 대역폭 낭비
  - 2개의 독립 컨트롤러 필요 시 Die Size 증가

**권장 방안**: **독립 2-Rank 방식 + Address Mapping 최적화**

**구현 전략**:
```
Rank0 (LPDDR5-6400 x32):
  • NPU0 주 사용 영역: 0x0000_0000 - 0x7FFF_FFFF (2GB)
  • 화질엔진 버퍼: 0x8000_0000 - 0x9FFF_FFFF (512MB)
  • 공용 영역: 0xA000_0000 - 0xBFFF_FFFF (512MB)

Rank1 (LPDDR5-6400 x32):
  • NPU1 주 사용 영역: 0x0000_0000 - 0x7FFF_FFFF (2GB)
  • 펌웨어/제어: 0x8000_0000 - 0x8FFF_FFFF (256MB)
  • 공용 영역: 0x9000_0000 - 0xBFFF_FFFF (768MB)
```

**SW 레벨 최적화**:
- LLM 모델 파라미터를 2개 Rank에 균등 분배 로딩
- KV Cache는 현재 활성 NPU의 Rank에 할당
- 화질엔진 버퍼는 고정 영역 사용 (Fragmentation 방지)

**시뮬레이션 필요 데이터** (TODO List):
1. NPU0과 NPU1의 동시 동작 시나리오별 메모리 트래픽 패턴
2. LLM Inference 단계별 메모리 Access 패턴 (Prefill vs. Decode)
3. 화질엔진의 프레임별 버스트 크기 및 주기

---

### 2.2 HW 설계 및 검증 단계

#### **R1: LPDDR5 PHY Training 실패** [★★★★★ 심각도/긴급성]

**리스크 이유**:
- LPDDR5-6400은 데이터율 6400 MT/s로 매우 고속이며, 신호 무결성 확보 극도로 어려움
- Training 실패 시 칩 전체가 사용 불가능 (메모리 없이는 아무것도 못 함)
- 온도, 전압, 공정 편차에 민감

**기술적 난제**:

1. **Write Leveling**: 6400 MT/s에서 DQS와 CLK의 스큐를 ±25ps 이내로 맞춰야 함
2. **Read Leveling**: Read Data Eye 폭이 좁아 Margin 확보 어려움
3. **VREF Calibration**: 송수신단 전압 기준점을 동적으로 조정해야 함
4. **온도 드리프트**: 부팅 시 25°C → 동작 시 70°C로 변화 시 타이밍 파라미터 변동

**개발 방안**:

| 항목 | 방안 | 구현 상세 |
|------|------|-----------|
| **Periodic Re-training** | 온도 센서 기반 | • 온도 10°C 변화 시 Re-training 트리거<br>• Re-training 시간: ~50ms<br>• NPU/화질엔진 동작 중단 필요 → **성능 영향** |
| **Training 파라미터 저장** | Flash에 Golden 값 저장 | • 공장 테스트에서 최적값 Flash에 저장<br>• 부팅 시 Golden 값으로 시작 후 Fine-tuning만 수행<br>• 부팅 시간 ~100ms 단축 |
| **Adaptive Training** | 런타임 Error Rate 모니터링 | • DDR 컨트롤러에서 ECC Error Rate 모니터링<br>• Error Rate > 임계값 시 Background Re-training<br>• 복잡도 높음, 검증 어려움 |
| **Fallback 속도** | Training 실패 시 속도 다운그레이드 | • LPDDR5-6400 실패 → 5600 → 4800 순차 시도<br>• 대역폭 감소하지만 동작 보장 |

**권장 전략**: **Training 파라미터 저장 + Periodic Re-training + Fallback**

**PHY IP 선정 기준**:
- LPDDR5-6400 검증 실적 있는 IP (Samsung, Cadence, Synopsys)
- Auto-Training 기능 지원
- Temperature Sensor Interface 지원
- 상세한 Margin 리포트 기능 (Si 검증용)

**검증 계획**:
- **FPGA Prototype**: 먼저 LPDDR5-4800로 기본 동작 검증
- **Si Emulation**: 실제 칩 제작 전 Process Corner (SS, TT, FF) 별 Training Margin 시뮬레이션
- **Temperature Cycling Test**: -20°C ~ 85°C 범위에서 Training 성공률 확보 목표 99.9%

---

#### **R2: DRAM QoS 미흡으로 화질엔진 실시간 보장 실패** [★★★★★ 심각도]

**리스크 이유**:
- 화질엔진은 60fps 4K (3840x2160) 처리 시 **프레임당 16.67ms 데드라인** 존재
- NPU의 LLM Inference는 burst가 크고 불규칙하여 DRAM 독점 가능성
- QoS 미흡 시 화면 끊김, 티어링 등 치명적 화질 저하 발생

**정량적 요구사항 분석**:

**화질엔진 대역폭 요구**:
```
4K 60fps YUV420 처리 기준
• 입력: 3840 x 2160 x 1.5 (YUV420) x 60fps = 746 MB/s
• 출력: 동일 = 746 MB/s
• 총: ~1.5 GB/s (버퍼링 고려 시 2 GB/s 필요)

LPDDR5-6400 x32 x 2채널 이론 대역폭:
• 6400 MT/s x 32bit x 2 / 8 = 51.2 GB/s
• 실효 대역폭 (Efficiency 70%): 35.8 GB/s

→ 화질엔진은 전체 대역폭의 5.6% 사용
```

**NPU 대역폭 요구** (추정):
```
LLM Inference (7B 모델 기준, INT8)
• Prefill 단계: 높은 대역폭 (20-30 GB/s burst)
• Decode 단계: 낮은 대역폭 (2-5 GB/s)
```

**문제**: NPU Prefill 단계의 Long Burst가 화질엔진 요청을 블록킹

**QoS 메커니즘 설계**:

**레벨 1: DRAM 스케줄러 QoS**
```verilog
// DRAM Controller 내부 Arbiter
priority_queue[0] = 화질엔진 (AXI ID 0x00-0x0F)   // 최고 우선순위
priority_queue[1] = NPU1 (AXI ID 0x10-0x1F)       // 중간 우선순위
priority_queue[2] = NPU0 (AXI ID 0x20-0x2F)       // 중간 우선순위
priority_queue[3] = 펌웨어/기타 (AXI ID 0x30-0x3F) // 최저 우선순위

// Weighted Round-Robin with Burst Chunking
if (화질엔진_request_pending && elapsed_time > latency_budget) {
    preempt_current_transaction();
    schedule(화질엔진);
}
```

**레벨 2: Burst Chunking**

문제: NPU가 AXI Burst Length = 256으로 요청 시, 하나의 Burst 완료까지 화질엔진 대기
→ 해결: DRAM 스케줄러에서 Long Burst를 Chunk로 분할

```
NPU의 ARLEN=256 요청을
→ 32 x (ARLEN=8) Chunk로 분할
→ 각 Chunk 사이에 화질엔진 요청 삽입 가능
```

**구현**:
```
DRAM Scheduler Config Register:
• MAX_BURST_CHUNK_SIZE = 8  // 최대 8 beats까지만 연속 수행
• PQ_ENGINE_LATENCY_BUDGET = 100 cycles  // 화질엔진 최대 대기 시간
```

**레벨 3: NPU/화질엔진 Bus Manager QoS**

NPU IP 내부의 Bus Interface에 QoS 설정 레지스터 추가:
```
NPU_QoS_Config:
  • max_outstanding_requests: 4 (동시 요청 제한)
  • max_burst_length: 16 (큰 Burst 금지)
  • bandwidth_limiter: 80% (전체 대역폭의 80%까지만 사용)
```

**레벨 4: 화질엔진 버퍼 사이즈 검증**

화질엔진 IP 내부 Line Buffer 크기 확인:
```
필요 버퍼 크기 = (DRAM Latency + Jitter) x Pixel Rate
• Worst-case DRAM Latency: 500ns
• Jitter (NPU 간섭): 1000ns (1us)
• Pixel Rate (4K 60fps): 3840 x 2160 x 60 = 497 Mpixels/s

필요 버퍼 = 1.5us x 497M x 1.5 bytes = ~1.1 KB (라인당)
→ 화질 IP가 최소 2KB Line Buffer 보유 확인 필요
```

**권장 구성**:
1. **DRAM 스케줄러**: Weighted Round-Robin + Burst Chunking + Latency Budget
2. **NPU**: Burst Length 제한 (max 16) + Outstanding 제한 (4)
3. **화질엔진**: 충분한 Line Buffer (4KB 이상)
4. **런타임 조정**: 가속기 CPU가 동작 모드별 QoS 파라미터 동적 변경

**검증 방법**:
- **시뮬레이션**: NPU + 화질엔진 동시 동작 시나리오에서 화질엔진 Latency 측정
- **Worst-case 분석**: NPU 2개 Full Load + 화질엔진 동시 동작 시 데드라인 준수 확인
- **스트레스 테스트**: FPGA Prototype에서 장시간 동작 시 티어링/끊김 발생 여부 확인

---

#### **R5: PCIe DMA 유효 대역폭 부족** [★★★★ 심각도]

**리스크 이유**:
- PCIe Gen4 x1 이론 대역폭: ~2 GB/s (양방향 합산 ~4 GB/s)
- 실제 DMA Efficiency: 70-85% (TLP 오버헤드, Flow Control 등)
- LLM 모델 로딩 시간에 직접적 영향

**대역폭 요구 분석**:

**LLM 모델 로딩** (7B INT8 모델):
```
• 모델 크기: 7GB
• 목표 로딩 시간: 5초 이내
• 필요 대역폭: 7GB / 5s = 1.4 GB/s

PCIe Gen4 x1 실효 대역폭: 2 GB/s x 0.8 = 1.6 GB/s
→ 이론적으로 가능하나 Margin 부족
```

**KV Cache 교환** (Multi-user 시나리오):
```
• 사용자별 Context Length: 2048 tokens
• KV Cache 크기 (7B 모델): ~500 MB per user
• 사용자 전환 시간 목표: 1초
• 필요 대역폭: 500 MB/s

→ PCIe 대역폭으로 충분
```

**개발 방안**:

| 방안                           | DMA 방향           | 장점                                     | 단점                                                     | 권장 시나리오      |
| ---------------------------- | ---------------- | -------------------------------------- | ------------------------------------------------------ | ------------ |
| **A. Host-initiated Write**  | Host → 가속기 (MWr) | • Host에서 속도 제어 용이<br>• 표준적 방식          | • Posted Write로 인한 Completion 지연<br>• Out-of-Order 가능성 | 대부분의 경우      |
| **B. Device-initiated Read** | Host ← 가속기 (MRd) | • 가속기가 필요한 만큼만 요청<br>• Flow Control 정확 | • Non-Posted로 인한 Latency 증가<br>• Host 메모리 Lock 필요      | Pull 방식 선호 시 |
| **C. Hybrid**                | 상황별 전환           | • 최적 성능                                | • 복잡한 프로토콜                                             | 성능 극한 추구 시   |

**권장**: **방안 A (Host-initiated Write)** + DMA Descriptor Chaining

**최적화 기법**:

1. **Large TLP Size**: Max Payload Size = 256 bytes (PCIe Gen4 최대)
2. **Descriptor Chaining**: 여러 DMA 전송을 하나의 Descriptor Chain으로 묶어 오버헤드 감소
3. **Prefetch**: Host가 다음에 필요한 모델 레이어를 미리 전송
4. **압축**: LLM 모델을 압축하여 전송 후 가속기에서 압축 해제 (시간 vs. 복잡도 Trade-off)

**DMA 엔진 설계 요구사항**:
```
• Scatter-Gather DMA 지원
• Descriptor Chaining (최대 256 descriptors)
• Completion 인터럽트 (MSI/MSI-X)
• Error Handling (CRC Error, Timeout)
```

**측정 필요 항목** (TODO):
- 실제 FPGA Prototype에서 PCIe Gen4 x1 DMA 성능 측정
- Host SoC의 Root Complex 성능 확인 (일부 저가형 SoC는 Gen4를 지원해도 Gen3 수준 성능)

---

#### **R8: 온도 변화에 따른 DDR 성능 저하** [★★★ 심각도]

**리스크 이유**:
- 부팅 시 칩 온도: ~25°C
- Full Load 동작 시: ~70-80°C
- DDR Timing 파라미터는 온도에 민감 (특히 LPDDR5 고속 동작)

**개발 방안**:

| 방안 | 구현 | 장점 | 단점 |
|------|------|------|------|
| **Periodic Re-training** | 온도 10°C 변화 시 자동 Re-training | • 성능 유지<br>• 안정성 높음 | • Re-training 중 시스템 중단 (~50ms) |
| **Temperature Sensor + Margin 확보** | 넓은 Margin으로 Training | • Re-training 불필요<br>• 연속 동작 | • 최고 속도 미달 가능<br>• 전력 증가 |
| **Dynamic Derating** | 온도 상승 시 속도 자동 감소 | • 안정성 보장 | • 성능 저하 |

**권장**: **Periodic Re-training** (Background 방식)

**구현**:
```c
// 가속기 CPU 펌웨어
void temperature_monitor_task() {
    static int last_temp = 25;
    int current_temp = read_temperature_sensor();
    
    if (abs(current_temp - last_temp) >= 10) {
        // NPU/화질엔진 Idle 구간 대기
        wait_for_idle_window();
        
        // DDR Re-training 수행
        ddr_phy_retrain();
        
        last_temp = current_temp;
        log_info("DDR Re-trained at %d°C", current_temp);
    }
}
```

**Temperature Sensor 배치**:
- DDR PHY 근처에 센서 배치 (Die 온도 직접 측정)
- 정확도 ±2°C 이내

---

### 2.3 SW 스택 및 펌웨어 개발 단계

#### **R6: 메모리 Fragmentation** [★★★ 심각도]

**리스크 이유**:
- 여러 LLM 모델 또는 사용자별 KV Cache를 동적으로 할당/해제 시 Fragmentation 발생 가능
- Fragmentation 심화 시 큰 메모리 블록 할당 실패 → 모델 로딩 불가

**시나리오별 분석**:

**시나리오 1: 고정 메모리 할당**
```
메모리 맵:
[0x0000_0000 - 0x001F_FFFF]  2MB   펌웨어 + 제어 구조체
[0x0020_0000 - 0x0FFF_FFFF]  254MB 화질엔진 버퍼 (고정)
[0x1000_0000 - 0xFFFF_FFFF]  3.75GB NPU용 (LLM 모델 + KV Cache)

→ Fragmentation 리스크 없음 (모든 영역 고정)
```

**시나리오 2: 사용자별 Context 유지**
```
• LLM 모델: 7GB (고정 영역에 항상 로딩)
• KV Cache per user: 500MB
• 동시 사용자: 최대 4명

메모리 사용:
[고정] 7GB 모델
[동적] 500MB x 4 = 2GB KV Cache

→ 사용자 전환 시 KV Cache 해제/재할당
→ 경미한 Fragmentation 가능 (하지만 크기가 균일하므로 관리 용이)
```

**시나리오 3: 멀티 모델 동적 로딩**
```
• 응용 A: 7B 번역 모델 (7GB)
• 응용 B: 3B 요약 모델 (3GB)
• 응용 C: 1B 감성분석 모델 (1GB)

→ 모델 교체 빈번 시 Fragmentation 심각
→ 최악의 경우: 10GB 여유 공간 있으나 연속된 7GB 블록 없어 로딩 실패
```

**개발 방안**:

| 방안                     | 구현 주체   | 알고리즘                     | 장점                                | 단점                     |
| ---------------------- | ------- | ------------------------ | --------------------------------- | ---------------------- |
| **고정 할당**              | Host    | Static Partition         | • Fragmentation 없음<br>• 간단        | • 유연성 없음<br>• 메모리 낭비   |
| **Best-fit Allocator** | 가속기 CPU | Best-fit with Coalescing | • 범용적<br>• Fragmentation 감소       | • 할당 시간 증가<br>• 복잡도 증가 |
| **Slab Allocator**     | 가속기 CPU | 크기별 Pool                 | • 빠른 할당<br>• Fragmentation 최소     | • 메모리 낭비<br>• 크기 제한    |
| **IOMMU 활용**           | Host    | Paging                   | • 가상 연속 메모리<br>• Fragmentation 해결 | • 성능 오버헤드<br>• 복잡도 높음  |

**권장**: **시나리오별 차별화 전략**

**시나리오 1, 2**: **고정 할당** (가장 단순, 충분)
**시나리오 3**: **Slab Allocator** + **Compaction**

**Slab Allocator 설계**:
```c
// 가속기 메모리 풀
memory_pool_t pools[] = {
    {.block_size = 512*MB,  .count = 8},  // 소형 모델용
    {.block_size = 1*GB,    .count = 4},  // 중형 모델용
    {.block_size = 7*GB,    .count = 1},  // 대형 모델용
};

void* allocate_model_memory(size_t size) {
    for (int i = 0; i < NUM_POOLS; i++) {
        if (size <= pools[i].block_size) {
            return pool_alloc(&pools[i]);
        }
    }
    return NULL;  // Out of memory
}
```

**Compaction** (선택적):
- 메모리 할당 실패 시, 사용 중인 블록들을 앞으로 이동시켜 연속 공간 확보
- 비용: 수 GB 메모리 복사 시간 (~1초)
- 트리거: 할당 실패 시에만 수행

**IOMMU 불필요 판단**:
- 가속기 메모리는 Host가 직접 관리할 필요 없음 (가속기 CPU가 독립 관리)
- 페이징 오버헤드 (TLB Miss 시 수백 사이클)가 성능에 악영향
- Protection은 PCIe BAR Access Control로 충분

→ **IOMMU 미사용 권장**

---

#### **R10: HDCP 키 관리 및 보안** [★★★★ 심각도]

**리스크 이유**:
- HDMI RX에서 수신한 콘텐츠는 HDCP로 암호화되어 있음
- HDCP 키가 유출되면 콘텐츠 보호 무력화 → 법적 문제
- 키는 반드시 안전한 저장소에 보관 필요

**HDCP 동작 흐름**:
```
[HDMI Source] --암호화--> [HDMI RX (가속기)] --복호화--> [화질엔진] --> [출력]
                              ↑
                         HDCP 키 공급 필요
```

**개발 방안**:

| 방안                                 | 키 저장 위치             | 키 관리 주체 | 보안 수준 | 구현 복잡도 |
| ---------------------------------- | ------------------- | ------- | ----- | ------ |
| **A. Host SoC 관리**                 | Host Secure Storage | Host    | 중간    | 낮음     |
| **B. 가속기 CPU + Flash**             | 가속기 Flash (암호화)     | 가속기 CPU | 높음    | 중간     |
| **C. OTP (One-Time Programmable)** | 가속기 Die 내부 OTP      | 가속기 CPU | 매우 높음 | 높음     |

**권장**: **방안 B (가속기 CPU + Flash 암호화 저장)**

**근거**:
1. **독립성**: Host SoC 해킹 시에도 HDCP 키 안전
2. **유연성**: 펌웨어 업데이트로 키 교체 가능
3. **비용**: OTP 대비 저렴하면서도 충분한 보안

**구현 상세**:

**Flash 메모리 구조**:
```
[0x0000 - 0x7FFF]  32KB  부팅 펌웨어
[0x8000 - 0x9FFF]   8KB  HDCP 키 (AES-256 암호화)
[0xA000 - 0xFFFF]  24KB  예약
```

**키 로딩 시퀀스**:
```c
// 가속기 CPU 펌웨어
void hdcp_key_load() {
    // 1. Flash에서 암호화된 키 읽기
    uint8_t encrypted_key[8*KB];
    flash_read(0x8000, encrypted_key, sizeof(encrypted_key));
    
    // 2. Device-unique Key로 복호화 (HW 기반)
    uint8_t device_key[32];
    hw_get_device_key(device_key);  // Die마다 고유한 키 (PUF or Fuse)
    
    uint8_t hdcp_key[8*KB];
    aes256_decrypt(encrypted_key, device_key, hdcp_key);
    
    // 3. HDCP Descramble 엔진에 키 로딩 (HW 레지스터)
    hdcp_engine_load_key(hdcp_key);
    
    // 4. 메모리에서 키 삭제 (보안)
    memset(hdcp_key, 0, sizeof(hdcp_key));
}
```

**Device-unique Key 생성 방법**:
- **PUF (Physical Unclonable Function)**: 공정 편차를 이용해 칩마다 고유한 키 생성
- **eFuse**: 제조 시점에 Random Key를 Fuse에 기록
- 권장: **eFuse** (검증된 기술, 낮은 리스크)

**보안 요구사항**:
- HDCP 키는 절대 Plain Text로 외부 노출 금지
- 가속기 CPU의 SRAM에 키를 로드한 후 즉시 HDCP 엔진 레지스터로 전송, SRAM에서 삭제
- Debug Interface (JTAG) 통해 키 읽기 불가능하도록 Access Control

---

#### **R12: Host BAR 매핑 주소 공간 < 가속기 메모리 크기** [★★★★ 심각도]

**리스크 이유**:
- TV SoC의 PCIe Root Complex가 지원하는 BAR 주소 공간이 제한적일 수 있음
- 예: SoC는 32-bit 주소만 지원 → 최대 4GB BAR 매핑 가능
- 가속기 메모리: LPDDR5 x32 x 2 = 8GB 총 용량
- 4GB < 8GB → Host가 가속기 메모리 전체를 직접 접근 불가

**문제 시나리오**:
```
가속기 메모리: 0x0_0000_0000 - 0x1_FFFF_FFFF (8GB)
Host BAR0 매핑: 0x0_0000_0000 - 0x0_FFFF_FFFF (4GB만 가능)

→ 4GB 이상 영역은 Host가 직접 접근 불가
→ LLM 모델 로딩 시 문제 발생 가능
```

**개발 방안**:

| 방안                    | 구현                     | 장점                        | 단점                                      | 권장도   |
| --------------------- | ---------------------- | ------------------------- | --------------------------------------- | ----- |
| **A. Sliding Window** | BAR 레지스터로 매핑 영역 변경     | • 전체 메모리 접근 가능<br>• HW 간단 | • 빈번한 Window 이동 시 성능 저하<br>• SW 복잡도 증가  | ★★★★  |
| **B. DMA 전용**         | Host는 BAR 미사용, DMA만 사용 | • BAR 크기 무관<br>• 표준적      | • Host가 메모리 직접 읽기 불가<br>• Debugging 어려움 | ★★★★★ |
| **C. 메모리 축소**         | LPDDR5 x32 x 1채널만 사용   | • BAR에 모두 매핑              | • 성능 저하<br>• HW 낭비                      | ★     |
| **D. IOMMU**          | 가상 주소 매핑               | • 유연성 최고                  | • 성능 오버헤드<br>• Host IOMMU 필요            | ★★    |

**권장**: **방안 B (DMA 전용) + 방안 A (Sliding Window for Debug)**

**정상 동작 모드**: DMA만 사용
```c
// Host Driver
// LLM 모델을 Host DDR에서 가속기 DDR로 DMA 전송
pcie_dma_transfer(host_addr, accel_addr=0x10000000, size=7*GB);
```

**디버깅 모드**: Sliding Window로 메모리 덤프
```c
// Host Debug Tool
void dump_accel_memory(uint64_t addr, size_t size) {
    while (size > 0) {
        // BAR Window를 원하는 주소로 이동
        write_reg(BAR_WINDOW_BASE, addr & ~0xFFFFFFFF);
        
        // BAR를 통해 읽기
        uint32_t offset = addr & 0xFFFFFFFF;
        memcpy(buffer, bar_ptr + offset, MIN(size, 4*GB));
        
        addr += 4*GB;
        size -= MIN(size, 4*GB);
    }
}
```

**Sliding Window HW 구현**:
```
BAR0: 4GB 고정 매핑 (0x0_0000_0000 - 0x0_FFFF_FFFF)
BAR1: 4GB Sliding Window

BAR1 제어 레지스터:
• BAR1_WINDOW_BASE (RW): Window의 시작 주소 (4GB 단위)
  - 0x0: Window -> 0x0_0000_0000 - 0x0_FFFF_FFFF
  - 0x1: Window -> 0x1_0000_0000 - 0x1_FFFF_FFFF

Host에서 BAR1 + offset 접근 시:
  physical_addr = (BAR1_WINDOW_BASE << 32) | offset
```

**성능 영향**:
- DMA 사용 시: 영향 없음 (Window 사용 안 함)
- Window 이동 시: Register Write 1회 (~100ns) → 무시 가능

---

### 2.4 시스템 통합 및 부팅 (Bring-up) 단계

#### **R9: 부팅 시간 스펙 초과** [★★★★ 심각도]

**리스크 이유**:
- TV 제품의 부팅 시간은 사용자 경험에 직결 (목표: 3초 이내)
- 가속기 칩 초기화가 병목이 될 수 있음

**부팅 타임 분해 분석**:

| 단계                 | 시간 (ms)  | 병렬화 가능?    | 최적화 방안                       |
| ------------------ | -------- | ---------- | ---------------------------- |
| Host SoC 부팅        | 1500     | -          | (제어 불가)                      |
| PCIe Link Training | 50       | Host와 병렬   | Gen4 → Gen3 Fallback 시 시간 단축 |
| PCIe Enumeration   | 200      | -          | Host 최적화 필요                  |
| 펌웨어 로딩 (Host→가속기)  | 100      | -          | SRAM 크기 최소화                  |
| **DDR Training**   | **300**  | **병목**     | Golden 값 사용 시 100ms로 단축      |
| HDMI/화질엔진 초기화      | 100      | DDR과 병렬 가능 | -                            |
| **LLM 모델 로딩**      | **4000** | **병목**     | 압축 전송, 백그라운드 로딩              |
| **합계**             | **6250** | -          | **목표: 3000**                 |

**Critical Path**: DDR Training + LLM 모델 로딩 = 4.3초

**최적화 전략**:

**1. DDR Training 단축** (300ms → 100ms)
- Golden Training 값 Flash 저장 활용
- Fine-tuning만 수행

**2. LLM 모델 백그라운드 로딩**
```
부팅 시퀀스 변경:
[기존]
  Host 부팅 → 가속기 초기화 → 모델 로딩 → 응용 시작
  
[최적화]
  Host 부팅 → 가속기 초기화 (200ms)
    ↓
  응용 시작 (화면 ON, 모델 없이 기본 기능 동작)
    ↓ (백그라운드)
  모델 로딩 (4초, 사용자는 화면 보면서 대기)
    ↓
  AI 기능 활성화
```

**체감 부팅 시간**: ~1.7초 (화면 ON까지)
**실제 부팅 시간**: ~5.7초 (AI 기능 준비까지)

**3. 모델 압축 전송**
```
7B INT8 모델: 7GB
→ LZ4 압축: ~4GB (압축률 57%)
→ PCIe 전송 시간: 4GB / 1.6GB/s = 2.5초
→ 가속기에서 압축 해제: ~0.5초
→ 총: 3초 (기존 4초 대비 1초 단축)
```

**4. Suspend-to-RAM (STR) 활용**
```
STR 진입 시:
• 가속기 메모리를 Self-Refresh 모드로 유지
• LLM 모델 + KV Cache 보존

STR 복귀 시:
• DDR Training 생략 (Self-Refresh에서 복귀)
• 모델 재로딩 생략
• 복귀 시간: ~500ms
```

**STR 차별성 검토**:
- **장점**: 즉시 AI 기능 사용 가능 (사용자 경험 향상)
- **단점**: Standby 전력 증가 (~100mW, LPDDR5 Self-Refresh 전력)
- **결론**: **차별성 있음 - 적용 권장**

---

### 2.5 최적화 및 응용 운영 단계

#### 성능 최적화

**NPU-화질엔진 병행 실행 최적화**:
```
시나리오: 사용자가 4K 영상 시청 중 AI 자막 생성 요청

[병행 동작]
• 화질엔진: 60fps 4K 업스케일링 (연속)
• NPU: LLM 자막 생성 (Burst)

메모리 대역폭 할당:
• 화질엔진: 2 GB/s (고정, 최우선)
• NPU: 나머지 대역폭 (최대 33 GB/s) 사용

QoS 설정:
priority_queue[0] = 화질엔진 (절대 우선)
priority_queue[1] = NPU (Opportunistic)
```

**전력 관리 (DVFS)**:
```
동작 모드별 전력:
• Idle: NPU OFF, 화질엔진 OFF, DDR Self-Refresh
  → 전력: ~50mW
  
• 화질 단독: NPU OFF, 화질엔진 ON, DDR Active
  → 전력: ~2W
  
• NPU 단독: NPU ON, 화질엔진 OFF, DDR Full Speed
  → 전력: ~8W
  
• 병행 동작: NPU ON, 화질엔진 ON, DDR Full Speed
  → 전력: ~10W
```

**온도 기반 Throttling**:
```c
if (temperature > 85°C) {
    // NPU 클럭 감소
    set_npu_clock(800MHz → 600MHz);
    // 또는 화질엔진 FPS 감소
    set_pq_engine_fps(60fps → 30fps);
}
```

---

## 3. 개발 방안 상세 설명

### 3.1 가속기 칩 CPU 선정 및 역할

**최종 권장**: ARM Cortex-M7 @ 200MHz

**담당 역할**:

1. **부팅 및 초기화**
   - Boot ROM에서 PCIe PHY 초기화
   - Host로부터 펌웨어 수신 (PCIe)
   - DDR PHY Training 수행
   - HDMI/화질엔진/NPU 초기화

2. **실시간 제어**
   - DRAM QoS 파라미터 동적 조정
   - HDCP 키 관리 및 HDMI RX 제어
   - 온도 센서 모니터링 및 Periodic DDR Re-training
   - 인터럽트 1차 처리 (NPU 완료, 화질엔진 에러 등)

3. **전력 관리**
   - 동작 모드별 클럭/전압 조정 (DVFS)
   - Idle 시 NPU/화질엔진 전원 차단
   - STR 진입/복귀 제어

4. **디버깅 및 모니터링**
   - 성능 카운터 수집
   - Error 로깅
   - Host와 제어 메시지 통신 (Mailbox)

**펌웨어 구조**:
```
[Boot ROM] (8KB, Mask ROM)
  • PCIe PHY 초기화
  • Host로부터 펌웨어 수신 대기
  
[Main Firmware] (64KB, Flash)
  • RTOS (FreeRTOS 또는 Bare-metal)
  • DDR Training 코드
  • HDCP 키 관리
  • QoS 제어 알고리즘
  • 전력 관리
  
[Runtime] (SRAM)
  • Task Scheduler
  • Interrupt Handlers
```

---

### 3.2 메모리 아키텍처 최종 설계

**DRAM 구성**: 독립 2-Rank

**물리적 구성**:
```
Rank0: LPDDR5-6400 x32, 4GB
  • NPU0 주 사용
  • 화질엔진 공유 접근

Rank1: LPDDR5-6400 x32, 4GB
  • NPU1 주 사용
  • 펌웨어 및 제어 구조체
  • 화질엔진 공유 접근
```

**메모리 맵**:
```
Rank0 (4GB):
0x0_0000_0000 - 0x0_DFFF_FFFF (3.5GB)  NPU0 LLM 모델 + KV Cache
0x0_E000_0000 - 0x0_FFFF_FFFF (512MB)  화질엔진 버퍼 (Line Buffer, Frame Buffer)

Rank1 (4GB):
0x1_0000_0000 - 0x1_001F_FFFF (2MB)    펌웨어 + 제어 구조체
0x1_0020_0000 - 0x1_DFFF_FFFF (3.5GB)  NPU1 LLM 모델 + KV Cache
0x1_E000_0000 - 0x1_FFFF_FFFF (512MB)  예약 (화질엔진 확장 또는 공유)
```

**Access Control**:
- 각 마스터(NPU0, NPU1, 화질엔진, CPU)는 모든 Rank 접근 가능
- DRAM 스케줄러가 QoS 기반으로 중재
- SW(Host 또는 가속기 CPU)가 각 NPU에 사용 영역 할당

---

### 3.3 PCIe 인터페이스 설계

**BAR 구성**:
```
BAR0 (64-bit, 4GB):
  • 가속기 메모리 0x0_0000_0000 - 0x0_FFFF_FFFF 직접 매핑
  • 용도: 디버깅 시 메모리 덤프

BAR1 (64-bit, 4GB, Sliding Window):
  • 가속기 메모리 상위 4GB 매핑
  • 제어 레지스터로 Window 위치 조정

BAR2 (32-bit, 64KB):
  • 제어 레지스터
    - DDR 컨트롤러 설정
    - NPU 제어 (시작, 중지, 상태)
    - 화질엔진 설정
    - Mailbox (Host ↔ 가속기 CPU 통신)
    - 인터럽트 제어
    - BAR1 Window 제어
```

**DMA 엔진 사양**:
```
• Channel: 4개 (병렬 전송 가능)
• Descriptor: Scatter-Gather, Chaining 지원
• Max Payload Size: 256 bytes
• Max Read Request Size: 512 bytes
• 인터럽트: MSI-X (32 vectors)
```

**통신 프로토콜**:
```
[Host → 가속기] (명령 전송)
1. Host가 BAR2 Mailbox에 명령 작성
   • 명령 타입 (모델 로딩, Inference 시작 등)
   • 파라미터 (주소, 크기 등)
2. Host가 Doorbell 레지스터 Write
3. 가속기 CPU가 인터럽트 받고 Mailbox 읽기
4. 가속기 CPU가 명령 수행
5. 완료 시 Host에 MSI 인터럽트 전송

[Host ← 가속기] (데이터 요청, 선택적)
1. 가속기 CPU가 필요한 데이터 주소를 Mailbox에 작성
2. Host에 MSI 인터럽트
3. Host가 DMA로 데이터 전송
```

**권장**: Host-initiated DMA 방식 (가속기는 수동적)

---

### 3.4 부팅 시퀀스 상세

**전체 타임라인** (최적화 버전):

```
T=0ms      Power-On
T=0-100ms  [Boot ROM] PCIe PHY Init + Link Training
T=100ms    Host가 가속기 인식
T=100-300ms [Host] PCIe Enumeration, BAR 설정
T=300-400ms [Host] 펌웨어 DMA 전송 (64KB → SRAM)
T=400ms    [가속기 CPU] 펌웨어 실행 시작
T=400-500ms [가속기 CPU] DDR Training (Golden 값 사용)
T=500ms    DDR 사용 가능
T=500-600ms [가속기 CPU] HDMI/화질엔진/NPU 초기화
T=600ms    가속기 준비 완료, Host에 "Ready" 인터럽트
T=600ms-   [백그라운드] Host가 LLM 모델 DMA 전송 (7GB)
           (사용자는 이미 화면 사용 중)
T=3600ms   LLM 모델 로딩 완료, AI 기능 활성화
```

**체감 부팅 시간**: 600ms (화면 ON)
**AI 기능 준비**: 3600ms

---

### 3.5 보안 설계

**HDCP 키 보호**:
```
[저장]
Flash (암호화): AES-256(HDCP_KEY, Device_Unique_Key)
Device_Unique_Key: eFuse에 저장 (256-bit Random)

[로딩]
1. CPU가 Flash에서 암호화된 키 읽기
2. HW AES 엔진으로 복호화
3. HDCP Descramble 엔진 레지스터에 직접 전송
4. CPU SRAM에서 즉시 삭제

[Access Control]
• HDCP 엔진 레지스터는 CPU만 접근 가능 (PCIe 불가)
• Debug Interface (JTAG)로 eFuse 읽기 금지
```

**PCIe Access Control**:
```
BAR2 레지스터 중 일부는 Read-Only (Host가 쓰기 불가)
• DDR Training 파라미터 (보안상 중요)
• HDCP 관련 레지스터
```

---

## 4. 리스크 완화 체크리스트

### 4.1 설계 단계

- [ ] **R4**: 가속기 CPU 사양 확정 (ARM Cortex-M7 권장)
- [ ] **R7**: DRAM 구성 방식 결정 (독립 2-Rank 권장)
- [ ] **R3**: 부팅 시퀀스 설계 검토 (PCIe First, DDR Later)
- [ ] **R12**: Host SoC BAR 매핑 크기 확인 (Sliding Window 준비)

### 4.2 HW 설계 단계

- [ ] **R1**: DDR PHY IP 선정 (LPDDR5-6400 검증 실적 확인)
- [ ] **R1**: Training 파라미터 저장용 Flash 설계 (64KB)
- [ ] **R1**: 온도 센서 배치 (DDR PHY 근처)
- [ ] **R2**: DRAM 스케줄러 QoS 메커니즘 설계 (Burst Chunking 포함)
- [ ] **R2**: NPU Burst Length 제한 레지스터 추가
- [ ] **R2**: 화질엔진 Line Buffer 크기 확인 (4KB 이상)
- [ ] **R5**: PCIe DMA 엔진 설계 (Scatter-Gather, Chaining)
- [ ] **R10**: HDCP 보안 설계 (eFuse + AES)

### 4.3 SW 개발 단계

- [ ] **R6**: 메모리 할당 전략 확정 (시나리오별)
- [ ] **R9**: 부팅 시간 최적화 (백그라운드 모델 로딩)
- [ ] **R10**: HDCP 키 관리 펌웨어 구현
- [ ] **R11**: IOMMU 불필요 확정 (DMA 전용 방식)

### 4.4 검증 단계

- [ ] **R1**: DDR Training 성공률 테스트 (-20°C ~ 85°C)
- [ ] **R2**: NPU + 화질엔진 동시 동작 시 Latency 측정
- [ ] **R5**: PCIe DMA 실효 대역폭 측정 (FPGA)
- [ ] **R8**: Temperature Cycling Test
- [ ] **R9**: 부팅 시간 측정 (목표: 3초 이내 체감)

### 4.5 양산 단계

- [ ] **R1**: DDR Training Golden 값 Flash 저장 (공장 테스트)
- [ ] **R10**: HDCP 키 암호화 및 eFuse 프로그래밍

---

## 5. 개발 일정 및 마일스톤

| 단계 | 기간 | 주요 산출물 | Critical Risk |
|------|------|-------------|---------------|
| **아키텍처 설계** | 4주 | • CPU 사양 확정<br>• DRAM 구성 결정<br>• 부팅 시퀀스 설계 | R3, R4, R7 |
| **HW 설계** | 12주 | • RTL 설계<br>• DDR PHY 통합<br>• PCIe IP 통합 | R1, R2, R5 |
| **FPGA Prototype** | 8주 | • 기본 동작 검증<br>• DDR Training 검증<br>• PCIe 통신 검증 | R1, R5 |
| **펌웨어 개발** | 10주 | • Boot ROM<br>• DDR Training 코드<br>• HDCP 키 관리 | R1, R10 |
| **Si Tape-out** | - | • GDSII | - |
| **Si Bring-up** | 6주 | • 실물 칩 동작 확인<br>• 성능 측정<br>• 버그 수정 | R9 |
| **양산 준비** | 4주 | • Golden Training 값 생성<br>• 제조 테스트 프로그램 | R1 |

**총 개발 기간**: ~44주 (약 11개월, Si 제조 기간 제외)

---

## 6. 결론 및 권고사항

### 핵심 의사결정 사항

1. **가속기 CPU 탑재**: ARM Cortex-M7 (필수)
   - DDR Training, HDCP 키 관리, 실시간 QoS 제어를 위해 필수
   
2. **DRAM 구성**: 독립 2-Rank
   - 구현 단순성, QoS 구현 용이성, 보드 레이아웃 유연성
   
3. **부팅 전략**: PCIe First, DDR Later
   - 자체 저장장치 없으므로 Host 의존적 부팅 필수
   
4. **메모리 관리**: DMA 전용, IOMMU 미사용
   - 성능 오버헤드 없이 충분한 기능 제공
   
5. **보안**: eFuse + AES 암호화
   - HDCP 키 보호에 충분한 보안 수준

### 최우선 리스크 완화 활동

1. **DDR PHY 검증** (R1): 즉시 IP 선정 및 시뮬레이션 시작
2. **QoS 메커니즘 설계** (R2): HW 팀과 NPU/화질엔진 IP 팀 협업 필수
3. **부팅 시퀀스 설계** (R3): 펌웨어 팀과 HW 팀 조기 협의
4. **CPU 사양 확정** (R4): 모든 후속 설계의 전제조건

### 추가 검토 필요 항목 (TODO)

1. **NPU 트래픽 시뮬레이션**: LLM Inference 시 메모리 Access 패턴 분석
2. **Host SoC BAR 크기 확인**: TV SoC 사양 확인 필요
3. **화질엔진 IP 상세 스펙**: Line Buffer 크기, Burst 패턴 확인
4. **전력/발열 시뮬레이션**: Worst-case 동작 시 칩 온도 예측

---

**문서 버전**: 1.0  
**작성일**: 2026-01-31  
**작성자**: 시스템 아키텍트  
**검토 필요**: HW 설계팀, NPU IP팀, 화질엔진 IP팀, 펌웨어팀, 보안팀
