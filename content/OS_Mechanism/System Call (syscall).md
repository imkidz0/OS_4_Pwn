
System Call (시스템 콜)은 유저 공간에서 파일 작성이나 메모리 할당 등의 유저 권한(Ring 3)에서 할 수 없는 작업을 수행하기 위해 커널에 작업을 요청하는 것을 의미한다.
([[User & Kernel Space]])


유저 권한에서 "할 수 없는 작업들"을 커널에 요청하기 때문에,
Linux 커널에선 작업에 종류에 따라 번호(syscall number)를 매기고, 유저는 해당 번호를 인자로 하여 시스템 콜을 호출하게 된다.

커널이 시스템 콜을 받는다는 것은, 유저가 커널에 작업을 요청한 것이기 때문에,
커널의 시스템 콜 처리 과정에서 커널에서 사용되는 핵심 자료구조를 건드리게 된다.

이해를 돕기 위해 몇 가지 시스템 콜의 예시를 들어보자면 다음과 같다:

- `open` - file struct, fd table
- `mmap` - vm_area_struct, [[Page Table (1-Level Page Table)]]
- `fork` - task_struct, mm_struct
- `read` - page cache, file buffer
- `write` - page cache, file buffer
- `ioctl` - 드라이버 내부 상태