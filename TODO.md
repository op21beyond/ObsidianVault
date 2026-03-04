# 부팅 시퀀스
### Cold Boot
1. Host: Host CPU, CPCPU: Companion Chip CPU, CPIC: Companion Chip
2. CPIC : 자체 ROM으로부터 CPCPU 1st 부팅 (Host 부팅과 병행)
   -> PLL 초기화
   -> PCIe PHY 초기화
   -> PCIe configuration
3. Host : PCIe Enumeration & BAR configuration
   -> PCIe통해 CPIC DDR초기화 코드를 CPIC SRAM으로 복사하고 CPCPU에 Ready 알림 
     (또는 Host가 PCIe통해 직접 CPIC DDR초기화)  
4. CPIC : CPCPU가 SRAM의 코드 실행해서 DDR PHY training & DDR 초기화
   -> Host에 DDR Ready 알림 (Host는 특정 SRAM주소를 polling하면서 상태 확인)
5. Host : CPIC의 CPU/DSP/NPU펌웨어, HDCP Key을 CPIC DDR에 저장(AI모델은 제외)
6. CPIC : CPCPU가 DDR로부터 코드 실행하여 부팅 완료
7. Host : AI 모델은 앱이 로딩되면서 PCIe통해 전달 (느린 eMMC로 읽는 시간감안하여 실제 앱 실행되기 전에 미리 background로 CPID DDR에 write할 필요 있음)  
### Suspend-to-RAM (AI모델 로딩 시간 제거 가능)
1. 위 1~3 동일. 3에서 BootMode 정보도 SRAM에 써 놓으면 CPCPU가 그에 맞게 부팅 진행
# CPCPU
* ARM Cortex-M* 계열 또는 RISC-V, 수십KB SPI-Flash, 간단한 OS는 필요 (여러가지 task 관리)
* Cortex-M7은 SRP 사이즈와 유사 (TV SoC에서 SRP대체 검토된 적 있음)
### 필요성 大
1. PCIe 미사용시, USB Device 펌웨어 실행 포함 모든 하드웨어 제어
2. DDR초기화, PCIe PHY초기화, HDCP Key 관리, WoC제어등 하드웨어 관리 오프로딩(Host부담 완화) - PCIe 메모리는 Host에게는 NonCacheable이며 latency가 크므로 latency가 중요한 하드웨어 관리는 CPIC 로컬 CPU이 더 유리
3. 부팅 시간 증가 리스크 완화
### 필요성 中 - Host가 할 수도 있음
1. PCIe 버퍼 매니저 - Host address map에서 직접 CPIC의 메모리 전체 영역이 보이지 않는다면 NPU모델 또는 CPU메모리로 스왑된 KV-Cache 전달은 Host가 CPIC DDR 버퍼에 써 놓으면 CPIC가 임의 주소로 이동 또는 주소 변환 테이블 업데이트
	- Host가 항상 접근가능한 주소범위에 있는 PCIe BAR, CPID PCIe DMA descriptor, NPU IOMMU table등을 재설정 하여 NPU CPIC DDR 타겟 주소 변환이 가능
2. LLM 메모리 매니저 - Paged KV Cache, 멀티 모델 컨텍스트 스위칭등 상황에서 DDR Rank간 대역폭, 용량 밸런스 감안하면서 CPIC DDR 메모리를 할당/해제. NPU IOMMU Table관리
	* CPIC NPU 런타임에서 처리하는 작업이고 CPIC DDR의 실제 NPU 데이터를 접근할 필요없으므로 Host가 할 수 있음
### CPCPU없다면
1. CPIC의 모든 레지스터들은 PCIe통해 보이도록 CPIC 설계 필수
2. PCIe 초기화도 Host CPU가 해야 한다면 I2C 또는 SPI subordinate interface to CPIC internal Bus 인터페이스 필요함
# DDR
1. 부팅 시 PHY Training 조건과 고온 동작 상태의 조건 변화하면(io skew) 런타임 튜닝 필요한 지(retraining), 방법은?
2. LPDDR4/5, 또는 용량 혼합하여 사용 가능 필요? 
# DRAM 성능 및 QoS 관리
- 화질IP는 Hard-realtime, 고정적인 작은 용량, LLM대비 작은 BW
- NPU IP는 Non-realtime, 동적 가변 사용량 (모델 변경, KV Caching), Memory-bound. 256bit 700MHz bus interface. 버스 및 메모리 시스템에서 병목발생 하면 안됨.
- Bus manager 개수와 동작 시나리오가 많지 않으므로 동적 QoS 관리는 불필요하고 정적 QoS 세팅으로 충분할 것으로 예상되나, NPU0, NPU1이 DDR rank에서 충돌할 경우 각 NPU가 실행하는 모델 또는 응용에 따라 우선순위 설정필요할 수도 있음.
### 대응 방안
#### DRAM Controller
- AXI ID 기반 Scheduling (programmable해야 함)
- Burst Chunking(Splitting) 기능 - AXI Burst 단위로만 스케쥴링하면 Bus manager가 long AXI burst를 보낼 경우 스케쥴링 기회가 없어짐. DRAM Burst 단위 (예, BL=16)로 Chunking후 스케쥴링 필요 
- NPU는 256bit-700MHz AXI 설계로 LP5-6400과 스피드 매칭 설계되어있으므로 Clock Domain Crossing 이나 DDR Controller에서 Narrower Bus 부분에 의한 병목 발생 설계 피해야 함
#### 시스템 버스
- NPU는 256bit-700MHz AXI 설계로 LP5-6400과 스피드 매칭 설계되어있으므로 Clock Domain Crossing 이나 System Bus에서 Narrower Bus 부분에 의한 병목 발생 설계 피해야 함 
- Bandwidth throttling (transaction rate limiter) per high-thruput manager IP such as NPU, PQ
- Configurable number of maximum outstanding request for read and write
* NPU IP-side 또는 Bus-side 중에서 구현하면 됨. 화질IP는 항상 highest-priority
- IOMMU - Can be used for balancing bandwidth by address remapping
- Early Write Response 지원 (QoS관리 방안보다는 write latency 줄이는 방법임)
#### IP
- Configurable burst length
- Configurable number of maximum outstanding request for read and write  
#### 2-Rank DRAM의 구성
1. 독립적 2 Rank 
2. 인터리빙 2 Rank @DDR controller
3. 인터리빙 2 Rank @System bus
4. NPU 데이터가 DDR Rank 2개에 수동으로 용량과 대역폭 측면 모두 균형있게 할당가능하다면 1번 방식으로 충분. NPU 데이터가 불균형하게 할당될 수 밖에 없다면 3번 방식의 대역폭 및 용량의 자동 분할 효과로 대역폭 낭비 또는 트래픽 충돌로 인한 LLM 성능 저하 발생 리스크 낮출 수 있음
5. 설계 단계에서 선택, 또는 1, 3 방식 구현하고 응용 평가 후 선택. 업체가 구현할 수 있는 지 확인 필요.

| 항목                    | 방식1: 독립 2-Rank        | 방식2: Interleaving @ DDR Controller | 방식3: Interleaving @ System Bus                                    |
| --------------------- | --------------------- | ---------------------------------- | ----------------------------------------------------------------- |
| **HW 구조**             | 2개 독립 DDR Controller  | 1개 통합 DDR Controller               | 1개 DDR Controller + Bus Interleaving Logic                        |
| **Address Mapping 예** | Rank 선택 = Address[33] | Rank 선택 = Address[6] (64B 단위 교번)   | Rank 선택 = Bus Address Decoder                                     |
| **Die Size**          | 큼 (컨트롤러 2개)           | 작음 (컨트롤러 1개)                       | 방식1 + Bus Logic (모든 bus master에 request splitter/response merger) |
| **Board Layout 제약**   | 낮음 (Rank 물리적 분리)      | 높음 (Skew 관리 필수)                    | 낮음 (Bus 레벨 분리)                                                    |
| **이론 최대 대역폭**         | 51.2 GB/s             | 51.2 GB/s                          | 51.2 GB/s                                                         |
| **실효 대역폭**            | 38-40 GB/s (75-78%)   | 46-48 GB/s (90-94%)                | 43-45 GB/s (84-88%)                                               |
| **Bank Conflict 회피**  | 수동 (SW 메모리 할당)        | 자동 (HW Interleaving)               | 자동 (Bus 레벨)                                                       |
# PCIe 
* 대용량 전송 시나리오 1. AI 모델 로딩 (부팅 시 또는 런타임 모델 체인지). 수 GByte - PCIe 보다는 SoC eMMC read 시간이 병목 (예, eMMC5.1 sequential read 평균 속도 200MB/s이면, 1GB 모델 읽는데 5초) 
* 대용량 전송 시나리오 2. KV Cache swap to Host DDR (필요?) 
* SoC와 CPIC의  PCIe의 실성능 확인 필요 (무늬만 Gen4 또는 Gen5일 가능성)
* SoC PCIe DMA는 scatter-gather, descriptor chaining, MSI/MSI-X interrupt지원 필요
* Host PCIe : BAR 메모리 매핑 (Host CPU가 볼 수있는 주소 영역), Device-initiatied Read/Write to Host DDR 지원여부 확인 필요
* CPIC PCIe : PCIe to CPIC  메모리 맵 및 Bus Connection (누가 어디를 볼 수 있는 지), IOMMU여부, DMA사양 확인필요
# Interrupt Routing / MailBox
* 각 NPU 내부에는 Host/CP/DSP 사이의 메시지 통신 및 Interrupt (software interrupt, timer interrupt) 전달을 MailBox통하지만, 시스템 수준에서 CPCPU, Host (SoC CPU)간에도 
* CPIC Interrupt Controller : 각 NPU, 화질IP, PCIe/PCIe DMA 그밖의 인터럽트 소스들을 세팅에 따라 CPIC CPU와 Host CPU로 라우팅. Host CPU로 전달할 때는 GPIO Interrupt출력 또는 PCIe Message Signaled Interrupt (TV SoC에서 지원여부 확인 필요)
* CPIC MailBox : SoC Host CPU와 CPCPU간 메시지 통신 경로. SRAM + DoorBell Registers
# Debuging Support
#### GPIO Flag
- PCIe, DDR 부팅 상태 플래그 표시 : GPIO 출력 또는 I2C/SPI로 CPIC 내부 상태 레지스터 읽을 수 있도록 설계 필요.
#### Watdog timer
- 동작 중 멈춤 감지 : Watchdog timer 
#### System Bus
* 버스 에러 로그 : Error AXI ID or Bus manager ID, Address
* 버스 Protection Logic : Turstzone-like AXI ID and Address Range based protection, AXI Error logging, Bandwidth/latency monitor counter, Badwidth throttle
#### 온도 센서 
- 최소 NPU1, NPU2, DDR 온도 감지
#### TRACE32 지원
*  NPU내부 CP Control Processor(RISC-V) : RISC-V trace to CoreSight (NPU CP가 Trace지원 여부 확인 필요)
* CPIC CPU ARM일 경우 CoreSight, RISC-V 일 경우 RISC-V Trace to CoreSight 변환