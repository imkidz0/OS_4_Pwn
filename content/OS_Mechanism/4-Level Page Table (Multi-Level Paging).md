
본 문서는 이미 Page Table의 개념에 대해 알고 있거나, [[Page Table (1-Level Page Table)]] 문서를 읽고 이해하였음을 전제로 하여 설명한다.

먼저 4-Level Page Table이 등장한 배경에 대해 이해할 필요가 있다.
잠시 과거의 32-bit 운영체제를 살펴보자.

32-bit 운영체제에서는 Page Table이 2단계로 구성되면 가상 메모리([[Virtual Memory]]) 시스템을 구현할 수 있었다.

왜 2단계이면 충분할까?

각 Page Table의 크기 자체도 Page 크기이다.
PTE 1개의 크기가 4 바이트인 32-bit 운영체제에서,
하나의 Page Table의 Page에 들어가는 PTE의 개수는 `0x1000 / 0x4 = 0x400` 즉, 1024개이다.

2단계 Page Table의 구조는 다음과 같다.

- PD (Page Directory): 상위 Page Table로, 총 1024개의 Page Table의 엔트리를 갖는다.
- PT (Page Table): Page Directory 배열이 갖는 하나의 Page Table 주소로, 총 1024개의 PTE를 갖는다.

수학적으로 계산해보면 쉽다.
Page Directory는 1024개의 Page Table을 갖고, Page Table은 다시 1024개의 PTE를 갖는다.
이때 하나의 PTE는 하나의 물리 페이지로 매핑되므로, 하나의 PTE는 0x1000만큼의 메모리를 관리한다.
따라서 `2^10 × 2^10 × 2^12 = 2^32` 이므로,
2-Level Page Table만으로 32-bit 가상 주소 공간 전체를 정확히 커버할 수 있다.

그러나 64-bit 운영체제에서는 전체 표현 가능한 메모리의 개수가 2^64로 늘어나게 된다.
또한 PTE 1개의 크기가 4 바이트가 아닌 8 바이트가 되기 때문에
하나의 Page Table의 Page에 들어가는 PTE의 개수는 `0x1000 / 0x8 = 0x200` 즉, 512개이다.

이 문제를 해결하기 위해 4-Level Page Table이 등장하였다.
우선 4-Level Page Table을 사용할 경우,
32-bit 운영체제에서 사용한 수학적 계산법을 따라

`2^9 × 2^9 × 2^9 × 2^9 × 2^12 = 2^48`로, x86-64에서 현재 사용되는 가상 주소 공간은 64비트가 아닌 "48비트"이다.

64비트를 전부 사용해야 하는 게 아닌가? 싶지만, 2^48은 256 TB이며, 이 정도의 가상 주소 공간은 현재의 시스템에서 충분히 크기 때문에, x86-64는 전체 64비트를 전부 사용하는 대신,
Page Table 크기와 TLB 효율을 고려해 48비트만 사용하도록 설계되었다.
(TLB에 대한 자세한 설명은 [[Page Table Entry (PTE)]] 문서 참고.)

그리고 미래에 2^48 만큼의 가상 메모리로는 사용이 어려워질 때, Page Table을 한 단계 더 만들어서 `2^48 × 2^9 = 2^57`로 늘리는 설계를 채택하였다.

그렇다면 남은 16비트는 어떻게 사용할까?
남은 상위 16비트는 sign-extension (canonical form)으로 사용한다.
48비트 정수를 64비트 signed integer로 확장한 것임을 의미한다.

이때, 상위 16비트는 47번째 비트의 복사본이어야 한다.
즉 47번째 비트가 0이면, 상위 16비트는 모두 0이어야 하며, 47번째 비트가 1이면, 상위 16비트는 모두 1이어야 한다는 것이다.

이해를 돕기 위해 커널 공간 주소와 유저 공간 주소([[User & Kernel Space]])를 예시로 들어보겠다.

```
유저 공간 주소
0x 0000 7fff abcd e012
```
이 경우, 47번째 비트는 0이기 때문에 `(0x7 = 0111)`, 상위 16비트도 전부 0이다. `(0x0 = 0000)`

```
커널 공간 주소
0x ffff 8000 0000 0000
```
이 경우 47번째 비트가 1이기 때문에 `(0x8 = 1000)`, 상위 16비트도 전부 1이다. `(0xf = 1111)`

그렇다면 47번째 비트가 1인데, 상위 16비트가 전부 1이 아니라면 어떻게 될까?
```
0x 0001 8000 0000 0000
```
이 경우 47번째 비트는 1인데`(0x8 = 1000)`, 상위 16비트가 전부 1이 아니다. (0x1 = 0001)
이 주소에 접근하면 CPU는 이를 `non-canonical address`로 판단하고 `#GP fault`를 발생시킨다.

이제 4-Level로 Page Table을 관리하게 된 이유를 알았으니, 각 단계가 어떻게 구성되었는지 알아보도록 하자.

Linux 커널에서 사용하는 4단계 Page Table의 구조는 다음과 같다.

- PGD (Page Global Directory): 최상위 페이지 테이블
- PUD (Page Upper Directory): 상위 디렉토리
- PMD (Page Middle Directory): 중간 디렉토리
- PTE (Page Table): 페이지 테이블 (페이지 테이블 엔트리 아님!)

이때 CPU는
	`PGD == PML4 (page-map level 4)`
	`PUD == PDP (page-directory pointer)`
	`PMD == PD (page-directory)`
	`PTE == PT (page-table)`
라고 지칭한다. 리눅스에서의 이름인 Page Table로서의 PTE와 Page Table Entry인 PTE가 혼동의 여지가 있어 설명은 CPU의 용어로 할 것이다.

한 가지 주의할 점은, Linux의 하위 페이지 테이블인 PTE는 [[Page Table Entry (PTE)]]와 다르다는 것을 알아야 한다.
표현만 PTE고, 실제로는 Page Table을 의미한다.

이제 이 4단계 Page Table이 실제 가상 주소에서 어떻게 표현되는지 살펴보자.
설명을 위해 위에서 썼던 주소의 예시를 다시 가져와서 설명하겠다.

```
유저 공간 주소
0x00007fffabcde012
```

해당 주소는 4단계 Page Table에서 아래와 같이 표현할 수 있다.

예시를 사용하여 설명하면,
```
Virtual Memory = 0x00007fffabcde012

Offset = 0x012
VPN = (0x00007fffabcde012 >> 12) = 0x00007fffabcde
or
VPN = (0x00007fffabcde012 / 0x1000) = 0x00007fffabcde

PT   = (0x00007fffabcde012 >> 12) & 0x1FF = 0x0de
PD   = (0x00007fffabcde012 >> 21) & 0x1FF = 0x157
PDP  = (0x00007fffabcde012 >> 30) & 0x1FF = 0x0fe
PML4 = (0x00007fffabcde012 >> 39) & 0x1FF = 0x0ff

PT[0x0de] -> (PTE converts to PFN) -> Physical Address!
```
가 되는 것이다.