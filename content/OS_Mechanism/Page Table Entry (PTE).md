
Page Table Entry (PTE)는 가상 페이지 번호(VPN)에 대응되는 물리 페이지 프레임 번호(PFN)와 접근 권한을 기록한 Page Table의 기본 단위이다.

우선 x86-64 기준에서의 PTE 구조에 대해 살펴보자.

기본적으로 PTE는
`PTE = [ PFN | Permission & Status Flags ]`
으로 이루어져 있다.

이때 Permission & Status 즉, 권한과 상태를 나타내는 flag bits의 종류가 다양한데,
x86_64 기준 Linux 커널 v6.18 코드에서는 아래와 같이 정의되어 있다:

```
// torvalds/linux/blob/v6.18/arch/x86/include/asm/pgtable_types.h

#define _PAGE_BIT_PRESENT	0	/* is present */
#define _PAGE_BIT_RW		1	/* writeable */
#define _PAGE_BIT_USER		2	/* userspace addressable */
#define _PAGE_BIT_PWT		3	/* page write through */
#define _PAGE_BIT_PCD		4	/* page cache disabled */
#define _PAGE_BIT_ACCESSED	5	/* was accessed (raised by CPU) */
#define _PAGE_BIT_DIRTY		6	/* was written to (raised by CPU) */
#define _PAGE_BIT_PSE		7	/* 4 MB (or 2MB) page */
#define _PAGE_BIT_PAT		7	/* on 4KB pages */
#define _PAGE_BIT_GLOBAL	8	/* Global TLB entry PPro+ */
#define _PAGE_BIT_SOFTW1	9	/* available for programmer */
#define _PAGE_BIT_SOFTW2	10	/* " */
#define _PAGE_BIT_SOFTW3	11	/* " */
#define _PAGE_BIT_PAT_LARGE	12	/* On 2MB or 1GB pages */
#define _PAGE_BIT_SOFTW4	57	/* available for programmer */
#define _PAGE_BIT_SOFTW5	58	/* available for programmer */
#define _PAGE_BIT_PKEY_BIT0	59	/* Protection Keys, bit 1/4 */
#define _PAGE_BIT_PKEY_BIT1	60	/* Protection Keys, bit 2/4 */
#define _PAGE_BIT_PKEY_BIT2	61	/* Protection Keys, bit 3/4 */
#define _PAGE_BIT_PKEY_BIT3	62	/* Protection Keys, bit 4/4 */
#define _PAGE_BIT_NX		63	/* No execute: only valid after cpuid check */
```

PTE는 하위 12비트(0~11)에 각종 flag bits가 위치하고, 그 위쪽 비트들 중`PHYSICAL_PAGE_MASK`에 해당하는 구간이 PFN으로 사용된다. 리눅스 커널은 이를 `PTE_PFN_MASK`로 정의한다.

보통 x86-64 (4KB 페이지 기준)에서는 PFN이 bit 12 이상에 위치하며, 실제 범위는 CPU가 지원하는 물리 주소 폭(`PHYSICAL_PAGE_MASK`)에 의해 결정된다. 상위 비트들은 NX, Protection Key, 소프트웨어 비트로 사용된다.

여기서 주의깊게 봐야할 것은, flag bits들이다.
PTE의 flags는 단순한 메타데이터가 아니라, CPU가 메모리 접근 권한을 관리하는 규칙이기 때문이다.

Pwn 해커의 관점에서 중요한 비트 7개만 알아보자.

##### 1. `_PAGE_BIT_PRESENT`

해당 비트는 해당 가상 페이지가 현재 물리 메모리(RAM)에 매핑되어 있는지를 나타낸다.
이 비트가 0이면 해당 페이지는 **아직 로드되지 않았거나 swap/file-backed 상태**임을 의미하고, 1이면 페이지의 데이터가 **RAM에 존재하는 상태**임을 의미한다.

- swap-backed: 원래 RAM에 존재했으나 메모리 부족으로 인해 디스크의 swap 영역으로 밀려난 상태를 뜻함.
- file-backed: 처음부터 RAM에 없었으며 mmap() syscall을 사용해 파일을 매핑한 페이지를 뜻함.

따라서 이 비트가 0인 PTE에 접근할 경우 [[Page Fault]]가 일어나게 되며, 이때 CPU의 제어는 커널의 page fault handler로 이동한다.

##### 2. `_PAGE_BIT_RW`

x86에서 읽기 권한은 항상 허용되기 때문에, 해당 비트는 쓰기 가능 여부만을 나타낸다.
이 비트가 0일 경우에는 쓰기 권한이 없다는 것을 의미하고, 해당 비트가 1일 경우에는 **쓰기 권한이 있다**는 것을 의미한다.

##### 3. `_PAGE_BIT_USER`

해당 비트는 유저의 접근 가능 여부를 나타낸다.
이 비트가 0일 경우에는 커널 공간에서의 접근만 허용함을 의미하고, 이 비트가 1일 경우에는 **유저 공간에서의 접근도 허용함**을 의미한다.
([[User & Kernel Space]])

##### 4. `_PAGE_BIT_ACCESSED`

해당 비트는 CPU가 해당 페이지에 접근한 적이 있는지의 여부를 나타낸다.
이 접근은 읽기, 실행도 포함한다.

이 비트가 0일 경우에는 CPU가 해당 페이지에 **접근한 적이 없음**을 의미하고, 이 비트가 1일 경우에는 CPU가 해당 페이지에 **접근한 적이 있음**을 의미한다.

##### 5. `_PAGE_BIT_DIRTY`

해당 비트는 CPU가 해당 페이지에 쓰기를 한 적이 있는지의 여부를 나타낸다.

이 비트가 0일 경우에는 CPU가 해당 페이지에 **쓰기를 한 적이 없음**을 의미하고, 이 비트가 1일 경우에는 CPU가 해당 페이지에 **쓰기를 한 적이 있음**을 의미한다.

##### 6. `_PAGE_BIT_GLOBAL`

TLB(Translation Lookaside Buffer)란 가상 메모리 주소를 물리 주소로 변환하는 속도를 높이기 위해 사용하는 CPU 안의 캐시다.
TLB에는 최근에 일어난 주소 변환 결과와 권한 비트(PTE flags)를 함께 캐시한다.
즉, TLB는 PTE에 대한 캐시라고 할 수 있다.

해당 비트는 CR3(Page Table 기준 주소)가 바뀔 때, 이 PTE에 해당하는 TLB 엔트리를 유지할지 버릴지의 여부를 나타낸다.

운영체제의 이론에서 커널의 주소 공간은 모든 프로세스에서 동일하기 때문에 [[Context Switch]]가 일어날 때마다 유저의 페이지만 바뀌고, 커널의 페이지는 항상 동일하다.

따라서 커널 주소의 경우 `_PAGE_BIT_GLOBAL`이 1로 설정되어 있으면 커널 페이지 주소 변환이 TLB에 잔류하여 성능 향상의 이점이 있다.

따라서 이 비트가 0일 경우에는 Context Switch가 일어날 때 TLB에서 해당 PTE가 제거됨을 의미하며, 이 비트가 1일 경우에는 Context Switch가 일어나도 TLB에선 해당 PTE를 유지할 것임을 의미한다.

##### 7. `_PAGE_BIT_NX`

해당 비트는 해당 가상 페이지의 실행 가능 여부를 나타낸다.
유저랜드 바이너리에서 사용되는 NX(Non-eXecutable)과 똑같은 용도의 비트이다.

이 비트가 0일 경우에는 해당 페이지가 실행 가능한 영역임을 의미하고, 이 비트가 1일 경우에는 실행 불가능한 영역임을 의미한다.