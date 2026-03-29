
1. 아래 사양에 따라 상용 양산가능한 수준의 완성도로 verilog로 설계해줘
2. 아래 내용은 user_prompt.md 파일로 정리하고, 너가 반도체 설계 전문가로서 코딩AI에 prompt로 줄 구체적이고 완전한 프롬프트는 complete_prompt.md로 작성해.
3. 검증을 위해 충분한 assert를 추가해
4. 인터페이스
	1. master up, master down, up과 down은 동일한 width, 동일한 clock 사용
	2. slave interface to read and write control/status registers
	3. control register에 아래 각 기능별 enable/disable 필드가 있어야 함
	4. axi width default는 256bit
5. performance logging
	1. badwidth logging
		1. period : cycle or number of transactions (레지스터로 선택 가능), enable되어있으면 계속 반복됨.
		2. AXI size x AXI length를 곱하여 전송된 write, read byte 수를 period timeout 때마다 각각 FIFO에 저장
		3. 저장 FIFO (depth)는 parameter로 정해짐		
		4. FIFO는 even, odd가 있어서 N 번 period 때마다 저장 위치가 even->odd, odd->even으로 스위칭됨. N은 FIFO depth보다는 작은 값으로 control register에 설정되는 값임.
		5. N번 period 때마다 인터럽트(레벨 타입) 출력 할 수 있음 (interrupt enable 로 control register 설정되어있을 경우)
		6. 현재 저장 FIFO 위치(even or odd), 마지막 interrupt가 어떤 FIFO (even/odd) 저장후 발생한 것인지를 나타내는 status register있어야 함
		7. control register에 있는 주소 범위 (start, size)내의 transaction만 트래킹 하거나 전체를 트래킹함(선택하는 control register, start/size register 있어야 함)
	2. latency logging
		1. period는 bandwidth logging의 레지스터와 동일한 것을 사용
		2. write latency의 경우 AXI AW 후 B까지를 카운터
		3. read latency의 경우 AXI AR 후 1st RVALID와 last RLAST RAVLID의 latency 2개를 각각 로그
		4. 각각의 transaction을 저장하는 것이 아니고 평균값을 저장하므로 latency를 accumulation하고, transaction의 수를 각각 카운트하고 period timeout때 저장함.
		5. FIFO는 bandwidth logging처럼 even/odd로 구성되고 bandwidth logging에서의 N 에 따라 동일한 타이밍으로 동작함. 
		6. start, size 주소 범위내의 것만 트래킹할 지는 bandwidth logging과 동일
	3. bandwidth throttling
		1. period_throttle (위 logging의 period와 별개로 control register에 설정가능) 주기마다 bandwidth(위와 동일한 방식으로 카운팅됨)가 control register에 설정된 값보다 크면 aw, ar 채널의 awready, arready에 대해 control register에 설정된 delay (ar, aw 각각 다르게 설정가능)를 넣어서 강제로 bandwidth를 떨어뜨린다.
		2. throttle은 logging기능에서 base, size레지스터로 정해지는 주소 영역에 대해서만 적용한다.
	4. 타이밍은 고주파수로 동작가능하도록 설계하고 up, down 에 optional(파라메터로 선택가능) register slice를 넣는다.
	5. control register, status register는 apb 32bit (pstrob는 0xf로 고정) 인터페이스, master interface와 동일한 클럭
	6. 인터럽트는 1개