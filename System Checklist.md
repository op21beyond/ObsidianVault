
# prompt 
TV soc에 PCIe Gen4 1lane으로 연결되는 가속기 칩을 개발하려고해. 
이 칩은 LLM모델 추론용 NPU 2개와 4K 화질처리 엔진을 포함하고 있고 DRAM은 32bit 모바일 디램 2채널을 사용할거야 (lpddr5-6400 x32 * 2). 
NPU 자체는 CPU와 Flash가 없거나, 필요하다면(검토 대상) 최소 CPU와 그 CPU의 firmware를 저장하는 플래시 메모리를 달 수는 있어. 그렇지만 LLM 모델은 항상 soc의 eMMC5.1 에 저장할 거야.
PCIe이외에도 비디오 입출력을 위한 HDMI RX (HDCP포함. HDCP scramble/descramble엔진은 자체 포함, 키관리는 Host 또는 가속기 칩 CPU가 해야함) 인터페이스도 있어. NPU는 내부 스트리밍 프로세스가 있어서 모델을 차리하는 NPU 바이너리(NPU 프로그램)을 읽어서 해석한 후에 내부 하드웨어 모듈에서 처리되도록 제어하는 역할을 해. 
NPU IP와 화질엔진 IP는 개발자가 따로 있고 나는 시스템 관점에서 SW, HW적으로 관리해야할 부분을 하드웨어 설계부터 펌웨어개발. 그리고 부팅부터 응용운영까지 전과정을 살펴보고 리스크를 관리하려고 해. 
대략적인 프로세스를 제안하고 이어서 각 프로세스의 리스크 또는 체크포인트를 제시해줘.


# 단계별 개발 프로세스
### 아키텍처 및 사양 확정
성능 타겟 설정, PCIe 대역폭 및 DRAM 대역폭 시뮬레이션, IP 통합 설계  
- DRAM에서 하드 리얼 타임 화질IP들의 동작 보장
	- DRAM 스케쥴러에서는 QoS 메커니즘 필요 (axi id 기반으로 화질엔진 ID에 대해 우선순위 부여)
	- DRAM 스케쥴러이외에 NPU와 화질엔진의 bus manager interface에서도  QoS를 설정할 수 있는 장치가 있으면 보완이 될 듯. 어떻게 설계하면 될까?
	- 화질엔진IP의 버스쪽 버퍼를 DRAM latency / bandwidth 가 그 변동성을 충분히 두고 있는 지 체크
	- NPU나 화질IP가 AXI Burst Length를 너무 크게 내보내거나 너무 작게 쪼개서 내보낼 때에도 DRAM 스케쥴러가 QoS기반으로 스케쥴이 가능한가? 예를 들어 스케쥴러가 있지만 burst command 단위로만 스케쥴한다면 NPU가 long burst를 내보낼 경우 화질엔진은 스케쥴될 기회를 잃게된다. 그러면 DRAM 스케쥴러에서 AXI burst를 chunk로 쪼갠 후 스케쥴하도록 설계하거나, NPU에서 burst length를 Host나 가속기 CPU가 설정할 수 있는 레지스터 인터페이스를 두는 것이 좋을 수도 있다. 
- DRAM 채널 구성
	- NPU 2개, 화질 IP(여러 개)가 DRAM 32bit x 2개 랭크를 사용한다면 2 rank interleaving 방식과 독립적 2 rank 방식 중 어느 것이 적합할까?
		- 독립 rank로 가더라도 화질 엔진의 메모리 대역폭은 1개 rank를 독자적으로 사용할 만큼 크지 않으므로 NPU도 쓸 수 있어야 함.
		- NPU 2개가 각각 사용하는 메모리의 양과 메모리 트래픽 발생 타이밍 동시성 또각각의 NPU에서 발생한 트래픽이 같은 타겟을 향할지(데이터의 메모리 할당)은 시나리오 의존적인데 현재는 알 수 없음. 설계의사결정이나 운영최적화에 반드시 필요한 사항이라면 시뮬레이션을 통해 얻어야 하는 데이터로 TODO list포함 필요.
		- 2rank interleaving을 가장 die size 효율적으로 구현할 수 있는 설계 방법은? 보드에서 두 x32 dram은 멀리 떨어질 가능성이 높아서 dram controller 안에서 interleaving은 어려울 수 있음 (단, 가능성은 배제하지 않고 하나의 방안에 포함은 시킬 필요 있음)
- PCIe 이론적 대역폭은 문제 없음. 실제 PCIe DMA통해 얻을 수 있는 유효 대역폭은 얼마인지 확인해야 할 것임. 그리고 각 데이터의 방향(Host<>가속기)에 대해서도 Host side에서 명령(read/write)을 시작할 수도 있고, 가속기 side에서 명령(write/read)을 시작할 수도 있을 것임. 어느 방식이 더 바람직할까?
## HW 설계 및 검증
RTL 설계, 전력/발열 분석, PCIe/DRAM PHY 최적화, FPGA Prototyping
- NPU와 화질은 항상 ON된다면 소비전력은 얼마인가?
- DDR은 PHY training 하는 부팅 시점과 사용 시점의 칩 온도가 다를텐데 다이나믹하게 periodic re-training을 하는 것이 필요한가? 
- 발열 관리를 위해 temperature sensor가 필요한가?
- 이 페이지의 내용을 모두 감안할 때 가속기 칩의 CPU가 필요하다면 적절한 사양과 담당하게 될 역할 상세 내용은?
## SW 스택 및 펌웨어 개발
Bootloader, PCIe Driver, Memory Management, NPU/PQ SDQ 개발
- 가속기 메모리가 Host에게 보여야 하고, Host가 관리해야 할 범위는 무엇인가?
- IOMMU 대응 - IOMMU는 Host가 가속기 메모리를 Host 메모리의 일부로 보면서 관리할 때 도구인가? 아니면 다른 필요성이 있을까? Page단위로 IOMMU 운영하려면 테이블 생성 교체에도 Host 또는 가속기 칩의 CPU 의 부담이 있을텐데 얼마정도 일까? (table 생성 갱신 시간 빈도, Host가 담당할 경우 table 복사 시간등). IOMMU는 protection이나 security용도로도 사용하지만 이 부분울 강조할 필요는 없으므로 메모리 용량과 대역폭측면에서 필요성만 검토하면 됨. 
- 가속기칩의 Memory fragmentation리스크는 실재하는가? 어떤 시나리오에서 가능한지 시나리오 별로 검토해 볼필요있음.
	- 시나리오1: 화질엔진과 펌웨어등은 고정된 주소에 일정크기의 메모리만 필요. 나머지 모들 메모리 영역을 NPU가 사용가능.
	- 시나리오2: 위와 동일. NPU LLM의 컨텍스트를 사용자별로, 또는 현재 foreground응용별로 유지하고자 함. 단 모든 모델 파라메터는 가속기 칩메모리에 올려놓을 수 있음.
	- 시나리오3: 위와 동일. 단 응용에 따라 사용되는 모델이 바뀌면 Host가 가속기 칩으로 갖다 줘야 함.	
	- 또 다른 시나리오를 상상해 볼 것
- soc에서 PCIe통해 볼 수 있는 BAR mapping가능한 메모리 주소 공간(soc bus address mapping)이 가속기칩의 메모리크기보다 작으면 가속기칩의 IOMMU활용이나 가속기칩의 CPU역할 분담을 정하는데 영향을 미치는가? 
- 가속기 칩의 인터럽트는 가속기 안에서 일차로 처리해야 할까? Host가 대응해도 될까? (NPU동작시 큰 덩치의 동작은 NPU내부 스트리밍 프로세스가 담당함)
- 가속기 칩과 Host간 통신은 PCI MSI가 좋을까? legacy GPIO 방식으로도 충분할까?
## 시스템 통합 및 부팅(Bring up)
실물 칩 테스트, OS포팅, PCIe Enumeration 확인, 기본 기능 검증
- 제품의 부팅 타임 스펙에 가속기 칩의 동작이나 제어가 병목이 될 부분은 없는가? 
- PCIe / HDMI(입력) / V by One (패널 출력) 통신 가능 시점
- NPU 초기화 및 모델 로딩 타임	
	- 모델 로딩 시간은 부팅, 앱별 다른 모델 사용등 시나리오 별로 따져볼 것	
* Suspend-to-RAM에 가속기칩의 메모리도 Suspend (self-refresh)를 유지해야 할 까? LLM 컨텍스트 (KV Cache)를 유지하는 게 제품 차별성에 기여할까? 일회성 동작 시간에만 영향을 미치므로 STR을 불필요한 것일까?
* 예상 하드웨어 초기화 순서
	* 가속기 칩에는 저장장치가 없으므로 **"PCIe First, DDR Later"** 전략이 핵심입니다.
		1. **Boot ROM 단계:** 칩 전원 투입 후 내부 Boot ROM이 동작하여 PCIe PHY를 초기 하고 Host(SoC)에 장치(Endpoint)로 노출됩니다.
		2. **PCIe Enumeration:** Host SoC가 가속기 칩을 인식하고, PCIe를 통해 가속기용 펌웨어(OS 포함)를 가속기 내부 SRAM으로 전송합니다.
		3. **DDR Training (가장 큰 리스크):** * LPDDR5-6400은 속도가 매우 빨라 **PHY 트레이닝(Write/Read Leveling 등)**이 매우 까다롭습니다.
		4. 펌웨어가 실행된 후 DDR 컨트롤러를 초기화해야 하는데, 이때 트레이닝이 실패하면 칩이 먹통이 됩니다. 온도 변화에 따른 **Periodic Training** 설계도 고려해야 합니다.
		5. **Peripheral Init:** DDR이 붙은 후, HDMI(HDCP), V-by-One, USB 등을 초기화합니다.  
## 최적화 및 응용 운영
LLM 모델 양자화/포팅, 화질-NPU 병행 실행 최적화, 전력관리(DVFS)
- 가속기 메모리 fragmentation을 신경써야 하는 시나리오가 있을까?
- 누가(Host? 가속기 CPU?) 어떻게 (IOMMU? 간단한 segmetation mapper?) 대응해야 할까?


