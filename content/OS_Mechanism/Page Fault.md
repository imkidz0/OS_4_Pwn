
Page Fault는 프로세스가 접근하려는 가상 메모리 페이지가
현재 물리 메모리(RAM)에 없거나, 접근 권한을 위반했을 때 발생하는 CPU 예외이다.
이는 메모리 접근 도중 발생하는 동기적 예외 상황이다.

Page Fault는 지연 로딩([[Demand Paging]]), [[Copy-on-Write (CoW)]], [[Memory-Mapped File]]과 같은 가상 메모리 메커니즘을 구현하기 위해 필요하다.

Page Fault가 일어나는 과정은 아래와 같다:

먼저 프로세스가 RAM에 없거나 접근 권한을 위반하는 가상 주소에 접근하면, CPU는 page fault를 발생시키고 커널의 page fault handler로 제어를 넘긴다. ([[Interrupt]])
커널은 해당 주소가 swap/file-backed 상태인지 확인한 뒤, 필요한 페이지를 디스크에서 읽어와 물리 메모리에 적재하고 [[Page Table (1-Level Page Table)]]을 갱신한다.
그 후 faulting instruction을 다시 실행하여 프로세스가 중단된 지점부터 계속 실행되도록 한다.

Page Fault는 SMEP/SMAP가 적용된 현대 커널에서 사용자 코드가 커널의 제어 흐름으로 진입할 수 있는 몇 안 되는 정상적인 통로 중 하나이다.
(SMEP와 SMAP에 대한 내용은 [[Kernel Mitigations]]문서에 자세히 기술되어 있다.)

또한 Page Fault를 통해 운영 체제는 물리적 메모리의 한정된 크기에도 불구하고, 프로세스가 더 큰 가상 메모리 공간을 사용할 수 있게 한다.