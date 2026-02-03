
이 문서에서는 현대의 커널에서 사용되는 대표적인 보안 완화 기법(Mitigations)을 다룬다.
이 기법들은 모두 유저 공간과 커널 공간 사이의 경계를 강제하거나, 커널 메모리의 무결성을 보호하기 위해 도입되었다.

본 문서를 이해하기 위해 [[User & Kernel Space]] 문서를 먼저 읽고 오는 것을 권장한다.

---
#### 1. KASLR (Kernel Address Space Layout Randomization)
KASLR은 커널의 가상 주소 배치를 부팅 시 무작위화하여, 공격자가 커널 코드나 데이터의 정확한 주소를 알기 어렵게 만드는 보호 기법이다.
유저랜드에서의 ASLR(Address Space Layout Randomization)을 커널 주소에도 적용시킨 것이라고 생각하면 이해가 수월할 것이다.
#### 2. Canary (Kernel Stack Canary)
Kernel Canary는 커널 스택에 삽입되는 무결성 검증 값이다.
커널 스택 버퍼 오버플로우가 발생해 이 값이 변조될 경우, 커널은 이를 감지하고 즉시 시스템을 중단하거나 Crash를 발생시킨다.
마찬가지로, 유저랜드에서의 Canary를 커널 스택에도 적용시킨 것이라고 생각하면 이해가 수월할 것이다.
#### 3. SMEP (Supervisor Mode Execution Prevention)
SMEP는 커널 모드(Ring 0)에서 유저 공간에 위치한 코드 페이지를 실행하는 것을 금지하는 CPU 하드웨어 보호 기법이다.
#### 4. SMAP (Supervisor Mode Access Prevention)
SMAP는 커널이 기본적으로 유저 공간 메모리에 접근하지 못하도록 차단하는 보호 기법이다.  
`copy_to_user()`와 같은 **커널이 인정한 유저 공간 메모리 접근**이 필요한 경우에는 일시적으로 SMAP를 해제한다.
#### 5. KPTI (KAISER)
KPTI는 유저 모드와 커널 모드에서 사용하는 [[Page Table (1-Level Page Table)]]을 분리하여, 유저 모드(Ring 3)에서는 커널 공간의 메모리 매핑 자체가 존재하지 않도록 하는 보호 기법이다.