## 복구
- 선행 기록 로그 (WAL 또는 커밋 로그)
: 장애, 트랜잭션 복구를 위해 disk에 저장하는 추가 전용 보조 자료 구조
캐시된 내용이 disk에 flush될 때까지 관련 작업 이력의 유일한 disk 복사본임
페이지 캐시: 페이지에 대한 변경 사항을 메모리에 버퍼링함

주요 기능)
1) disk에 저장된 페이지에 대한 변경 사항을 페이지 캐시에 버퍼링하면서 DB 시스템에서의 지속성 보장
2) 캐시된 페이지가 disk와 동기화될 때까지 작업 이력을 디스크에 저장
3) 장애 발생 시 log를 기반으로 마지막 메모리 상태 재구성
4) 로그를 재수행해서 트랜잭션 완료하거나 장애 발생 전 상태로 되돌림

- PostgreSQL
: checkpoint를 사용해 인덱스와 데이터 파일이 로그파일의 특정 레코드까지의 내용과 동기화됐는지 확인

checkpoint-process: 주기적으로 더티 페이지를 disk로 flush -> fsync() 커널 함수 호출

*fsync() 함수: 커널 페이지의 dirty-flag를 초기화, dirty-page와 disk 동기화 역할

fsync() 실패 시 OS나 디바이스에서 fsync I/O 에러가 발생하면 flush되지 못한 더티 페이지가 남아있음

*checkpoint: 모든 파일을 열어 두고 쓰기 때문에 일부 에러를 미리 감지 가능, checkpoint가 실패 없이 끝나면 데이터를 디스크에 성공적으로 flush했다고 확신

## log의 시맨틱
- WAL (Write Ahead Log)
: 추가 전용 로그 파일 구조

특징)
1) 작성된 로그 데이터는 불변하며, 쓰기 작업은 항상 순차적으로 일어난다.

2) 새로운 쓰기와 기존 데이터의 읽기 작업이 동시에 안전하게 일어날 수 있다.

- LSN (Log Sequence Number)
: WAL은 여러 로그 레코드로 이루어지며 각 레코드에는 단조롭게 증가하는 LSN이 부여된다. LSN은 내부 카운터 값이나 타임스탬프를 사용한다. 데이터 복구, 트랜잭션 순서 관리에 매우 중요하다.

특징)
1) WAL은 디스크 블록보다 작은 단위로 로그를 버퍼에 임시 저장한다.

2) force 작업으로 로그를 디스크에 flush 한다.

3) flush는 로그 버퍼가 가득 차거나, commit 시점에 수행된다.

4) 트랜잭션 완료와 로그 기록
: commit 로그 기록 -> WAL은 트랜잭션이 완료되면 commit 로그 레코드를 기록한다. 이 commit 로그가 기록되기 전까지 트랜잭션은 commit으로 간주되지 않는다. WAL 덕분에 시스템 장애에도 복구가 가능하다.

: 장애 복구를 위해 CLR (Compensation Log Record)를 사용하며 트랜잭션 undo 중에도 로그에 기록해 복구 작업을 완전하게 만든다.

5) 체크포인트: WAL을 정리하기 위해 checkpoint 수행하며 모든 데이터 페이지를 disk에 기록한 시점의 스냅샷 역할을 한다.
싱크(Sync) 체크포인트 vs 퍼지(Fuzzy) 체크포인트
: Sync 체크포인트는 모든 페이지를 한 번에 디스크에 flush하며 가장 안전하지만 비효율적이다.

: Fuzzy 체크포인트는 일부만 flush하고 나머지는 비동기적으로 flush한다. 효율적이지만 일부 페이지는 여전히 메모리에 남아 있다. 
→ PostgreSQL은 fuzzy checkpoint를 많이 사용한다.

체크포인트 정보는 로그의 last_checkpoint 포인터에 기록된다. fuzzy 체크포인트는 begin_checkpoint, end_checkpoint 레코드로 구간이 정의된다.

## 작업 로그 vs 데이터 로그
Shadow Paging
: 새 데이터는 버퍼에 쓰고 commit 시에 page table pointer를 변경해 새로운 데이터로 스위칭한다.
System R 같은 DBMS는 Shadow Paging을 사용해 무결성을 보장한다.
→ 쓰기 작업 시 페이지 복사를 기반으로 한다.

1) 물리적 로그 vs 논리적 로그
물리적 로그: 바이트 단위로 변화를 기록한다. redo, undo 모두 물리적인 페이지 상태를 기반으로 한다. redo 시 모든 변경 페이지를 참조해야 하므로 비용이 크다.
논리적 로그: SQL 문장이나 “키 X 값을 Y로 변경” 같은 의미적 기록이다. 페이지가 아니라 작업 단위로 기록하며 redo 시 실제 작업을 재수행한다.

→ 대부분 DB는 물리적 로그와 논리적 로그 둘 다 사용한다.

## 스틸과 포스 정책 (Steal vs No-Steal, Force vs No-Force)
1) Steal / No-Steal
Steal 정책: 트랜잭션 수행 중 dirty page를 디스크에 flush 허용한다. 장애 시 undo가 필요하다.

No-Steal 정책: 트랜잭션 commit 전에 dirty page를 디스크에 flush하는 것을 금지한다. 장애 시 undo 불필요하지만 메모리 부담이 크다.

2) Force / No-Force
Force 정책: commit 시 모든 dirty page를 디스크에 flush한다. redo 불필요하지만 성능이 저하된다.

No-Force 정책: commit 시 디스크 flush를 강제하지 않는다. redo가 필요하지만 성능이 우수하다.

→ 대부분 DBMS는 성능을 위해 Steal + No-Force 조합을 사용한다.

## 용어
Redo: 장애 복구 시 다시 수행해야 하는 작업.

Undo: 장애 시 완료되지 않은 작업을 취소하는 작업.

LSN: 로그의 순서를 식별하기 위한 일련번호.

CLR: undo 중에 발생한 작업을 보정하기 위한 로그.

## ARIES 복구 알고리즘
ARIES: Steal/No-Force 정책 기반 복구 알고리즘

빠른 복구를 위해: 물리적 로그 → redo, 논리적 로그 → undo

* ARIES 복구 과정

1) 분석 단계 (Analysis)
장애 직전 LSN과 Dirty Page Table 분석, 미완료 트랜잭션 파악, 트랜잭션 복구 준비

2) 재실행 단계 (Redo)
장애 직전까지의 작업을 모두 재수행, LSN 기준으로 redo를 적용, 커밋 여부 관계 없이 redo 수행

3) 언두 단계 (Undo)
미완료 트랜잭션의 작업을 되돌림, Compensation Log Record(CLR)를 사용해 undo 진행, 장애 시 undo 작업도 로그에 기록



## 동시성 제어 (Concurrency Control)
: 여러 트랜잭션이 동시에 수행될 때 상호 간섭 방지하여 데이터 일관성과 무결성을 지킴

동시성 제어 방식
1) 낙관적 동시성 제어 (OCC): 트랜잭션이 충돌 없이 동시에 실행, 끝날 때 검증해서 충돌 있으면 Rollback, 충돌 가능성 낮은 환경에 유리

2) 다중 버전 동시성 제어 (MVCC): 각 트랜잭션마다 타임스탬프로 버전 구분, 과거 시점 데이터 참조 가능, 무잠금 방식으로 구현 가능, 대표적인 방식: PostgreSQL의 MVCC

3) 비관적 동시성 제어 (PCC): 잠금 기반 방식, 공유 자원에 잠금 설정, 충돌 가능성이 높은 환경에서 사용, 단점: 교착 상태 발생 가능

## MVCC (Multi Version Concurrency Control)
: 과거 데이터를 버전별로 관리, 트랜잭션 시작 시점 기준의 데이터 스냅샷을 사용함, 무잠금 방식 또는 2단계 잠금으로 구현 가능

* 교착상태: 여러 트랜잭션이 서로 잠금을 해제하기를 기다리는 상황, 비관적 잠금 환경에서 자주 발생

해결 방법: Lock Timeout 설정, Deadlock Detection 알고리즘 사용

