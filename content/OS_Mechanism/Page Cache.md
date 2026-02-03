
Page Cache는 디스크에 존재하는 파일의 내용을  
file-backed physical pages([[Demand Paging]]) 형태로 표현해 두는 커널에서의 저장소이다.

리눅스에서 파일은 원래 디스크에 존재하지만,  
다음과 같은 상황에서 커널은 파일의 내용을 Page Cache에 페이지 단위로 적재한다:

- 파일을 읽을 때
- `mmap`([[System Call (syscall)]])으로 매핑할 때
- 실행 파일을 로드할 때

이 과정에서 커널은 파일의 오프셋(inode, offset)을 RAM의 물리 페이지와 연결한다.  
Page Cache는 바로 이 "(inode, offset) → physical page" 매핑들을 담는 커널의 테이블이다.

중요한 점은 Page Cache가 프로세스 ([[Process]])별이 아니라 시스템 전체에서 공유된다는 것이다.  
즉, 한 파일의 한 오프셋은 시스템 전체에서 **단 하나의 Page Cache 페이지만 가진다.**

예를 들어:

프로세스 A에서 `/tmp/note.txt`를 읽고 해당 파일에 `"Hello"`를 썼다면,  
이 내용은 `/tmp/note.txt`의 Page Cache 페이지에 반영된다.

이때 프로세스 B와 C가 같은 파일을 읽으면,  
모두 동일한 Page Cache 페이지를 참조하기 때문에  
똑같이 `"Hello"`를 보게 된다.

이는 모든 프로세스가 `/tmp/note.txt`의 동일한 Page Cache 페이지를 공유하고 있기 때문이다.

[[Memory-Mapped File]]이 생성되면,  
그 가상 주소 영역의 PTE ([[Page Table Entry (PTE)]])는 익명 메모리를 가리키는 것이 아니라  
Page Cache의 physical page를 가리키게 된다.


따라서 Memory-Mapped File은 다음과 같은 구조를 가진다:
`Virtual Address -> Physical Page (Page Cache Page) -> File (inode + offset)`