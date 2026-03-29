다음을 반영해야 해.
아래와 같은 모듈 단위로 설계할 것.

(A. master IP, axi4) ->
(B. rank interleaving master side adapter, input axi4 1개, output axi4 1개, input단에서 wrap burst를 incr로 쪼개는 기능 필요하고 A와 인터페이스에서는 wrap지원, 입력 burst는 rank boundary에서 한번 이상 split있을 수 있음) ->
(C. system bus, axi4, 256bit) ->
(D. rank interleaving slave side adapter, input axi4 256 1개, output axi4 1개) ->
(E. rank controller, ramulator2(https://github.com/CMU-SAFARI/ramulator2), DPI-C지원, LPDDR5-5500 x32bit 메모리, 고성능 pipelined설계, axi4 burst를 memory burst width와 length 감안하여 쪼개어야 함. write-read data cohrency를 위한 behavioral memory필요)

독립적으로 동작하는 (D+E) rank interleaving controller가 2개 (2rank).
(A+B) rank interleaving master는 32bit 1개, 64bit 1개, 256bit 2개로 구성되는 시스템.

이 사항 반영하여 prompt, verilog, document 갱신할 것.

간단하게 프로토타입 형태로 만들려고 하지 말고 고성능으로 양산가능하도록 완성도 있게 만들어야 함. 특히 B, D.