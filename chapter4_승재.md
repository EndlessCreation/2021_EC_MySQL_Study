## 4.2.8 Double Write Buffer
---
* InnoDB 스토리지 엔진의 리두로그는 로그 공간의 낭비를 막기 위해 페이지의 변경된 내용만 기록
  
* InnoDB 스토리지 엔진에서 더티 페이지를 디스크 파일로 플러시할 때 일부만 기록되는 문제가 발생하면(하드 웨어의 오작동, 시스템 비정상 종료등 으로 인해) 그 페이지의 내용은 복구가 불가능 할 수도... 
  
* 이렇게 **페이지가 일부만 기록**되는 현상을 -> **파셜페이지(Partial-page)** 또는 **톤 페이지(Torn-page)**

=> InnoDB 스토리지 엔진에서는 이러한 문제를 막기 위해 **Double-Write** 기법 이용

### Double Write
1. 변경된 데이터 페이지를 모아서 한 번의 디스크 쓰기로 시스템 테이블스페이스의 Double Write 버퍼에 기록
2. 데이터 페이지를 개별로 파일로 기록

* 페이지 기록 중 비정상적 종료가 되었을 경우, InnoDB 스토리지 엔진은 재시작될 때 항상 Double Write 버퍼의 내용과 데이터 파일의 페이지들을 모두 비교해서 다른 내용을 담고 있는 페이지가 있으면 Double Write 버퍼의 내용을 데이터 파일의 페이지로 복사.

* innodb_doublewrite 라는 시스템 변수로 사용할지 여부 제어 가능

* **안정성**을 위해 주로 사용

* 그러나 기존의 회전하는 저장 시스템 방식이 아닌 SSD에서는 부담스러움

* 데이터의 무결성이 매우 중요한 서비스에서 고려하자~

* 추가적으로 무결성에 민감한 서비스라면 해당 Double Write뿐만 아니라 리두 로그, 복제를 위한 바이너리 로그 등 트랜잭션을 COMMIT하는 시점에 동기화할 것들이 많다는 것에 주의하자!

## 4.2.9 언두 로그
---
* 백업된 데이터

* 어떻게 사용?
  1. 트랜잭션 보장
  2. 격리 수준 보장

* 중요한 역할을 담당하지만 관리 비용이 크다.

### 4.2.9.1 언두 로그 레코드 모니터링
* update 문을 이용해서 name을 'HelloMan'으로 바꾼다고 가정

* 트랜잭션을 거밋하지 않아도 실제 데이터 파일 내용은 'HelloMan'으로 변경

* 그리고 기존 값이 언두 영역에 백업

* 커밋하면 그대로 유지, 롤백하면 언두 영역의 데이터로 복구
  

* **대용량의 데이터를 처리하는 트랜잭션 OR 트랜잭션이 오랜 시간 동안 실행될 때 언두로그 양이 급격히 증가**

* 트랜잭션이 완료됐다고 해서 해당 트랜잭션이 생성한 언두 로그를 즉시 삭제할 수 있는 것은 아니다. 먼저 실행한 트랜잭션이 해당 언두로그를 참조할 수 있기에 지속적으로 유지 -> 언두 로그양 증가
  
* 이렇게 언두 로그양이 증가하면 공간적 문제 뿐 아니라 빈번하게 변경된 레코드를 조회하는 쿼리가 실행되면 InnoDB 스토리지 엔진은 언두 로그의 이력을 필요한 만큼 스캔해야 필요한 레코드를 찾을 수 있기에 쿼리 성능이 전반적으로 떨어짐

* MySQL 5.5까지는 언두 로그의 사용 공간이 한번 늘어나면 서버를 새로 구축하지 않는 한 줄일 수 없었지만 8.0인 현재 언두 로그 공간 문제 완전히 해결(언두 로그 순차적 사용, MySQL 서버가 필요한 시점에 사용공간 자동으로 줄이기 등으로)

* 그래도 여전히 MySQL 서버에서 활성 상태의 트랜잭션이 장시간 유지되는 것은 성능상 안 좋다! -> 즉 **항상 언두 로그 레코드가 얼마나 되는지 항상 모니터링 해야한다! 안정적인 시점의 언두 로그 레코드 건수를 기준으로!**

### 4.2.9.2 언두 테이블스페이스 관리
* 언두 테이블스페이스 -> **언두 로그가 저장되는 공간**

* 기존 언두 로그는 시스템 테이블스페이스에 저장 -> 확장의 한계
  
* 8.0부터 기본적으로 시스템 테이블스페이스 외부의 별도 로그 파일에 기록

* 언두 테이블스페이스 구조 : **언두 테이블스페이스 > 1~128개 롤백 세그먼트 > 1개 이상의 언두 슬롯**

* 하나의 롤백 세그먼트는 InnoDB의 페이지 크기를 16바이트로 나눈 값의 개수 감큼 언두 슬롯을 가짐.

* 하나의 트랜잭션이 필요로 하는 언두 슬롯의 개수는 트랜잭션이 실행하는 INSERT, UPDATE, DELETE 문장의 특성에 따라 최대 4개 -> 일반적으로 대략 2개

* 최대 동시 트랜잭션 수 = (InnoDB페이지 크기 / 16) * (롤백 세그먼트 개수) * (언두 테이블스페이스 개수)

* 위처럼 대략 각 트랜잭션이 언두 슬롯 2개를 먹는다고 가정하면 나누기 2를 해야겠죠?

* **Undo tablespace truncate** -> 언두 테이블스페이스 공간을 필요한 만큼만 남기고 불필요하거나 과도하게 할당된 공간을 운영체제로 반납
  * **자동 모드**: InnoDB 스토리지 엔진의 **퍼지 스레드**가 주기적으로 깨어나서 언두 로그 공간에서 불필요해진 언두 로그를 삭제하고 언두 로그 파일에서 사용되지 않는 공간을 잘라내고 운영체제로 반납. -> 이 작업을 **언두 퍼지**라고 함. 언두 퍼지를 사용할지, 얼마나 자주 실행되게 할지는 시스템 변수로 설정 가능

  * **수동 모드**: 시스템 변수로 언두 퍼지의 자동 실행을 off했거나 생각보다 자동 모드로 언두 테이블스페이스의 공간 반납이 부진한 경우 -> 아예 언두 테이블스페이스를 비활성화 -> 퍼지 스레드가 비활성 상태의 언두 테이블스페이스를 찾아서 불필요한 공간을 잘라내서 운영체제로 공간 반납 -> 반납 완료후 다시 언두 테이블스페이스 활성화 (수동 모드는 언두 테이블스페이스가 최소 3개이상은 돼야 작동)

## 트랜잭션 격리 수준
---
1. **READ UNCOMMITTED** : A 트랜잭션의 변경 내용이 Commit이나 Rollback 상관없이 B 트랜잭션에서 보여짐
2. **READ COMMITTED** : A 트랜잭션의 변경 내용이 Commit되어야만 B 트랜잭션에서 보여짐
3. **REPETABLE READ** : 각 트랜잭션이 시작되기 전에 커밋된 내용에 대해서만 조회 가능
4. **SERIALIZABLE** : 가장 단순하고 엄격 -> 순수한 SELECT 작업에도 공유 잠금

* **아래로 내려갈수록 트랜잭션간 고립정도는 높아지지만, 성능은 떨어짐**

## 4.2.10 체인지 버퍼
---
* 레코드가 INSERT되거나 UPDATE될 때는 데이터 파일을 변경하는 작업 뿐 아니라 해당 테이블에 포함된 인덱스를 업데이트하는 작업도 필요 -> 랜덤하게 디스크를 읽는 작업이 필요 -> 테이블 인덱스가 많다면 소모가 큼

* 변경할 인덱스 페이지가 버퍼 풀에 있으면 바로 업데이트를 수행

* 없으면? -> 즉시 실행하지 않고 임시 공간에 저장 -> 바로 사용자에게 결과를 반환 -> 성능 향상 (원래는 디스크로부터 읽어와야했으니까!)

* 이때 사용하는 임시 메모리 공간 => **체인지 버퍼(Change Buffer)**

* 단 중복 여부 체크가 필요한 유니크 인덱스는 체인지 버퍼 사용 불가

* 체인지 버퍼에 임시로 저장된 인덱스 레코드 조각은 이후 백그라운드 스레드에 의해 병합 -> 이 역할을 수행하는 **버퍼 머지 스레드**

* 8.0부터 해당 버퍼링 기능이 INSERT, DELETE, UPDATE로 인해 키를 추가하거나 삭제하는 작업에 대해서도 가능하게 개선.

* 또한 시스템 변수로 체인지 버퍼의 사용이 비효율적일 때는 사용하지 않을 수 있게 개선.

* 체인지 버퍼의 공간 크기도 시스템 변수로 설정 가능.

## 4.2.11 리두 로그 및 로그 버퍼
---
* 리두 로그는 트랜잭션의 4가지 요소인 ACID 중에서 D에 해당하는 영속성과 가장 밀접하게 관련

* 리두 로그 -> 서버가 비정상적으로 종료됐을 때, 데이터 파일에 기록되지 못한 데이터를 잃지 않게 해주는 안전 장치의 역할

* 모든 DBMS의 데이터 파일은 쓰기보다 읽기 성능을 고려한 자료구조 -> 파일 쓰기는 디스크의 랜덤 엑세스 필요 -> 상대적으로 큰 비용

* 성능 저하를 막기 위해 쓰기 비용이 낮은 자료구조를 가진 리두 로그를 가짐.

* 서버의 비정상적 종료 -> 데이터 파일은 2가지의 일관되지 않은 데이터
  1. 커밋 -> 그러나 데이터 파일에 기록 X
  2. 롤백 -> 그러나 데이터 파일에 기록 X

* 1 -> 리두 로그 사용

* 2 -> 언두 로그 사용 -> 그러나 이 때도 커밋됐는지, 롤백됐는지, 아니면 트랜잭션 실행 중간 상태였는지 확인하기 위해 리두 로그 필요

* DB서버에서 리두 로그는 트랜잭션이 커밋되면 즉시 디스크로 기록되도록 시스템 변수를 설정하는 것을 권장 -> 그래야 비정상적 종료시 직전까지의 커밋 내용이라도 리두 로그에 남고 해당 시점까지 복구가 가능하니까

* 근데 커밋될 때마다 리두 로그를 디스크에 기록하고 동기화하는 작업은 많은 부하를 유발 -> 시스템 변수로 어느 주기로 디스크에 동기화할지 결정

* 리두 로그 파일들의 전체 크기는 InnoDB 스토리지 엔진이 가지고 있는 버퍼 풀의 효율성을 결정하기에 신중히 결정 -> 시스템 변수로 결정 가능

* 적절히 리두 로그 파일의 전체 크기가 설정되어 있어야 버퍼 풀에 모았다가 한 번에 모아서 디스크로 기록이 원활

* 사용량이 매우 많은 DBMS의 경우 리두 로그 기록 작업이 문제가 될 수도... -> 이러한 부분을 보완하기 위해 최대한 ACID 속성을 보장한느 수준에서 버퍼링 -> 이 버퍼링에 사용되는 공간이 **로그 버퍼**

### ACID
* 트랜잭션의 무결성을 보장하기 위해 꼭 필요한 4가지 요소
  1. A -> Atomic : 트랜잭션은 원자성 작업이어야 한다.(모든 작업이 반영되거나 롤백)
  2. C -> Consistent : 일관성
  3. I -> Isolated : 격리성
  4. D -> Durable : 한 번 저장된 데이터는 지속적으로 유지돼야한다. (지속성)

* **일관성과 격리성은 서로 다른 두 개의 트랜잭션에서 동일 데이터를 조회하고 변경하는 경우에도 상호 간섭이 없어야한다는 것을 의미!**

### 4.2.11.1 리두 로그 아카이빙
* 8.0부터 리두 로그를 아카이빙할 수 있는 기능 추가

* 리두 로그 아카이빙 기능은 데이터 변경이 많아서 리두 로그가 덮어쓰여도 백업이 실패 X

* 백업 툴이 리두 로그 아카이빙을 사용하기 위해서 먼저 아키이빙된 리두 로그가 저장될 디렉터리를 시스템 변수에 설정 -> 해당 디렉터리는 MySQL 서버를 실행하는 유저만 접근이 가능해야함!

* 리두 로그 아카이빙이 비정상적으로 끊어지면 해당 아카이빙 파일도 지워지게 된다!


### 4.2.11.2 리두 로그 활성화 및 비활성화
* MySQL 서버에서 트랜잭션이 커밋돼도 데이터 파일은 즉시 디스크로 동기화되지 않는 반면, 리두 로그(트랜잭션 로그)는 항상 디스크로 기록

* 8.0부터는 수동으로 리두 로그를 활성화, 비활성화 가능

* 데이터를 복구하거나 대용량 데이터를 한번에 적재하는 경우, 리두 로그를 비활성화해서 데이터의 적재 시간을 단축시킬 수 있음.

* 위처럼 비활성화 후 다시 로그 활성화 필수! (데이터 복구의 문제, 체크 포인트 시점의 문제등 발생 가능)

## 4.2.12 어댑티브 해시 인덱스
---
* **어댑티브 해시 인덱스** -> 사용자가 수동으로 생성하는 인덱스가 아니라 **InnoDB 스토리지 엔진에서 사용자가 자주 요청하는 데이터에 대해 자동으로 생성하는 인덱스** -> 시스템 변수를 통해 활성화, 비활성화 가능

* B-Tree 인덱스에서 특정 값을 찾기 위해 -> Root노드 -> 브랜치 노드 -> 리프 노드 순으로 탐색

* 적당한 사양의 컴퓨터에서 해당 작업을 동시에 몇 개 실행하는 것은 상관없으나 동시에 몇천 개의 스레드로 실행하면 컴퓨터의 CPU는 엄청난 프로세스 스케줄링을 하게 되고 자연히 쿼리의 성능이 떨어짐.

* 어댑티브 해시 인덱스는 이러한 B-Tree 검색 시간을 줄여주기 위한 기능

* InnoDB 스토리지 엔진은 **자주 읽히는** 데이터 페이지의 키 값을 이용해 해시 인덱스를 만들고, 필요할 때마다 어댑티브 해시 인덱스를 검색해서 레코드가 저장된 **데이터 페이지를 즉시 찾아**갈 수 있음 -> B-Tree를 루트 노드부터 리프 노드까지 찾아가는 **비용이 사라짐** -> **CPU가 더 적은 일**을 하게되고 **쿼리의 성능이 빨라**짐. -> **더 많은 쿼리를 동시 처리** 가능

* 해시 인덱스는 "**인덱스 키 값, 데이터 페이지의 주소**"쌍으로 관리 

* 인덱스 키 값 : B-Tree 인덱스의 고유번호(Id)와 B-Tree 인덱스의 실제 키 값의 조합으로 생성

* B-Tree 인덱스의 고유번호가 들어가는 이유는 InnoDB 스토리지 엔진에서 어댑티브 해시 인덱스는 하나만 존재하기 때문

* 즉 모든 B-Tree 인덱스에 대한 어댑티브 해시 인덱스가 하나의 해시 인덱스에 저장되기 때문에 -> 특정 키 값이 어느 인덱스에 속한 것인지 구분 필요함

* **데이터 페이지의 주소**는 실제 키 값이 저장된 데이터 페이지의 메모리 주소를 가짐 -> 버퍼 풀에 로딩된 페이지의 주소를 의미 -> 그러므로 어댑티브 해시 인덱스는 버퍼 풀에 올려진 데이터 페이지에 대해서만 관리 -> 버퍼 풀에서 해당 데이터 페이지가 없어지면 어댑티브 해시 인덱스에서도 해당 페이지 정보가 사라짐.

* 어댑티브 해시 인덱스는 하나의 메모리 객체인 이유로 경합이 상당히 심함 -> 8.0부터 **파티션** 기능을 제공 -> 시스템 변수로 파티션 개수 변경 가능

* 어댑티브 해시 인덱스를 의도적으로 비활성하는 경우도 많음
  
* 어댑티브 해시 인덱스가 성능 향상에 큰 도움이 안되는 경우
  1. 디스크 읽기가 많은 경우
  2. 특정 패턴의 쿼리가 많은 경우 (조인 or Like패턴 검색)
  3. 매우 큰 데이터를 가진 테이블의 레코드를 폭넓게 읽는 경우

* 성능향상에 도움이 되는 경우
  1. 디스크의 데이터가 InnoDB 버퍼 풀 크기와 비슷한 경우(디스크 읽기가 많지 않은 경우)
  2. 동등 조건 검색(동등 비교와 IN 연산자)이 많은 경우
  3. 쿼리가 데이터 중에서 일부 데이터에만 집중되는 경우
  4. 데이터 페이지를 디스크에서 읽어오는 경우가 빈번한 경우(어댑티브 해시 인덱스는 데이터 페이지를 메모리(버퍼 풀)내에서 접근하는 것을 더 빠르게 만드는 기능이기 때문)

* 해당 기능도 결국 메모리를 사용하는 기능임. 그것도 때로는 상당히 큰 메모리를 차지할 수 있음 -> 그렇기에 성능향상에 도움이 안될 경우 비활성화하는 것이 좋음

* 어댑티브 해시 인덱스 활성화시: 데이터 페이지의 인덱스 키가 해시 인덱스로 만들어져야하고 불필요한 경우 제거도 필요 -> InnoDB 스토리지엔진은 해당 기능이 활성화 중에는 그 키 값이 해시 인덱스에 있든 없든 검색을 해봐야 알 수 있음. 즉 해시 인덱스의 효율이 없는 경우에도 InnoDB는 계속 해시 인덱스를 사용

* 삭제 작업 시에도 해당 데이터를 삭제시 어댑티브 해시 인덱스에서도 삭제해야하기에 CPU자원을 많이 사용해야함

* 즉 어댑티브 해시 인덱스의 도움을 많이 받을수록 테이블 삭제 또는 변경 작업이 더욱 치명적인 작업이 됨

* 즉 우리 서비스 패턴에 맞게 도움이 되는지 아니면 불필요한 오버헤드만 만들고 있는지 판단 필요

## 4.2.13 InnoDB와 MyISAM, MEMORY 스토리지 엔진 비교
---
* 8.0버전이 되면서 사실상 MyISAM, MEMORY 스토리지 엔진은 InnoDB에 비해 가지는 장점이 사라짐

# 4.3 MyISAM 스토리지 엔진 아키텍쳐
* MyISAM 스토리지 엔진의 성능에 영향을 미치는 요소
  1. 키 캐시
  2. 운영체제의 캐시/버퍼

## 4.3.1키 캐시
---
* InnoDB의 버퍼 풀과 비슷한 역할

* 인덱스만을 대상으로 작동(버퍼 풀은 인덱스 뿐만 아니라 데이터 파일도..)

* 인덱스의 디스크 쓰기 작업에 대해서만 부분적으로 버퍼링 역할

* 키 캐시 히트율 = 100 - (Key_reads / Key_read_requests * 100)

* Key_reads : 인덱스를 디스크에서 읽어 들인 횟수를 저장하는 상태 변수

* Key_read_requests : 키 캐시로부터 인덱스를 읽은 횟수를 저장하는 상태 변수다.

* 설정을 통해 키 캐시 크기를 변경 가능하고 제한된 값 이상의 크기를 쓰고 싶으면 별도로 키 캐시 공간을 설정하여 부여할 수도 있다.

## 4.3.2 운영체제의 캐시 및 버퍼
---
* MyISAM 테이블의 데이터에 대해서는 디스크로부터의 I/O를 해결해 줄 만한 어떠한 캐시나 버퍼링 기능도 X

* 그냥 운영체제에 의존

* 운영 체제의 캐시 기능은 남는 메모리를 사용하는 것이 기본 원칙
  
## 4.3.3 데이터 파일과 프라이머리 키(인덱스) 구조
---
* MyISAM 테이블은 데이터 파일이 Heap 공간(순서X)처럼 활용

* MyISAM 테이블에 레코드는 프라이머리 키 값과 무관하게 INSERT되는 순서대로 데이터 파일에 저장

* MyISAM 테이블에 저장되는 레코드는 모두 ROWID라는 물리적인 주솟값을 가짐

* PK와 세컨더리 인덱스는 모두 데이터 파일에 저장된 레코드의 ROWID값을 포인터로 가짐

* MyISAM에서 ROWID는 가변 길이와 고정 길이 두 가지 방법으로 저장 가능

# 4.4 MySQL 로그 파일
* **로그 파일**을 이용하면 MySQL 서버의 깊은 내부 지식이 없어도 MySQL의 상태나 부하를 일으키는 원인을 쉽게 찾아서 해결 가능

## 4.4.1 에러 로그 파일
---
* **MySQL이 실행되는 도중에 발생하는 에러나 경고 메시지가 출력되는 로그 파일**

* MySQL 설정 파일(my.cnf)에서 log_error라는 이름의 파라미터로 정의된 경로에 생성

* 별도 정의 X -> 데이터 디렉터리(datadir 파라미터에 설정된 디렉터리)에 .err라는 확장자가 붙은 파일로 생성

### 4.4.1.1 MySQL이 시작하는 과정과 관련된 정보성 및 에러 메시지
* MySQL의 설정 파일을 변경하거나 데이터베이스가 비정상적으로 종료된 이후 다시 시작하는 경우에는 반드시 MySQL 에러 로그 파일을 통해 설정된 변수의 이름이나 값이 명확하게 설정되고 의도한 대로 적용됐는지 확인 필요

### 4.4.1.2 마지막으로 종료할 때 비정상적으로 종료된 경우 나타나는 InnoDB의 트랜잭션 복구 메시지
* InnoDB의 경우 MySQL 서버가 비정상적 또는 강제적으로 종료시 다시 시작되면서 완료되지 못한 트랜잭션을 정리하고 디스크에 기록되지 못한 데이터가 있다면 다시 기록하는 재처리 작업을 진행

* 이 과정에 대한 간단한 메시지가 출력되는데, 간혹 문제가 있어서 복구되지 못할 때는 해당 에러 메시지를 출력하고 MySQL 다시 종료

* 이 단계에서 발생한 문제는 상대적으로 어려운 문제인 경우가 많음

* 앞에서 살펴본 `innodb_force_recovery` 파라미터를 0보다 큰 값으로 설정하고 재시작해야함


### 4.4.1.3 쿼리 처리 도중에 발생하는 문제에 대한 에러 미시지
* 쿼리의 실행 도중 발생한 에러나 복제에서 문제가 될 만한 쿼리에 대한 경고 메시지가 해당 에러 로그에 기록

### 4.4.1.4 비정상적으로 종료된 커넥션 메시지(Aborted Connection)
* 클라이언트 애플리케이션에서 정상적으로 접속 종료를 하지 못하고 프로그램이 종료된 경우

* 중간에 네트워크 문제가 있어서 의도하지 않게 접속이 끊어지는 경우

* 해당 메시지가 너무 많이 쌓일 경우 애플리케이션의 종료 로직을 검토해볼 필요가 있음

* `max_connect_errors` 시스템 변수 값이 너무 낮게 설정된 경우 "Host 'host_name' is blocked"라는 에러가 발생할 수도 있음

* 클라이언트 호스트에서 발생한 에러(커넥션 실패, 강제 연결 종료등)의 횟수가 `max_connect_errors` 변수 값을 넘게 되면 발생 -> 해당 값을 좀 증가시켜주면 됨 -> 물론 왜 이 에러가 발생했는지 원인을 알아보는게 좋음

### 4.4.1.5 InnoDB의 모니터링 또는 상태 조회 명령(SHOW ENGINE INNODB STATUS같은)의 결과 메시지
* InnoDB의 테이블 모니터링이나 락 모니터링, 또는 InnoDB의 엔진 상태를 조회하는 명령은 상대적으로 큰 메시지를 에러 로그 파일에 기록

* InnoDB의 모니터링을 활성화해놓고 그대로 유지해놓으면 에러 로그 파일이 너무 커져서 파일 시스템 공간을 다 사용해 버릴지도 모르기에 모니터링 사용 후 다시 비활성화 해야함

### 4.4.1.6 MySQL의 종료 메시지
* MySQL이 이유도 모르게 종료되거나 재시작될 경우 에러 로그 파일에서 MySQL이 마지막으로 종료되면서 출력한 메시지를 화긴하는 것이 왜 MySQL 서버가 종료됐는지 확인하는 유일한 방법

## 4.4.2 제너럴 쿼리 로그 파일(제너럴 로그 파일, General log)
---
* 쿼리 로그 파일은 시간 단위로 실행됐던 쿼리의 내용이 모두 기록

* 제너럴 쿼리 로그는 실행되기 전에 MySQL이 쿼리 요청을 받으면 바로 기록하기 때문에 쿼리 실행 중에 에러가 발생해도 일단 로그 파일에 기록

* 파일로 기록할 수도, 테이블로 기록할 수도 있음

## 4.4.3 슬로우 쿼리 로그
---
* 서버의 쿼리 튜닝 시
  1. 서비스 적용되기 전에 전체적으로 튜닝하는 경우
  2. 서비스 운영 중에 MySQL 서버의 전체적인 성능 저하를 검사하거나 정기적인 점검을 위한 튜닝

* 1번은 검토해야할 대상 쿼리가 전부라 모두 튜닝

* 2번은 어떤 쿼리가 문제 쿼리인지 알기 힘듬

* 이때 어떤 쿼리가 문제인지 판별하는데 슬로우 쿼리 로그가 큰 도움

* 정상적으로 실행이 완료됐고 실행하는데 걸린 시간이 `long_query_time` 시스템 변수에 정의된 시간보다 많이 걸린 쿼리가 기록됨

* 파일로 기록할 수도, 테이블로 기록할 수도 있음

* 슬로우 쿼리 내용 -> Time, Query_time 등 책 참조

* 슬로우 쿼리 또는 제너럴 로그 파일의 내용이 상당히 많을 경우 쿼리를 하나씩 검토하기 매우 빡세다 -> Percona에서 개발한 Percona Toolkit의 pt-query-digest 스크립트를 이용해서 로그 파일 분석

* 로그 파일의 분석이 완료되면 아래의 3개의 그룹으로 나뉘어 저장

### 4.4.3.1 슬로우 쿼리 통계
* 분석 결과의 최상단에 표시

* 모든 쿼리를 대상으로 슬로우 쿼리 로그의 실행 시간(Exec time), 그리고 잠금 대기 시간(Lock time)등에 대해 평균 및 최소/최대 값을 표시

### 4.4.3.2 실행 빈도 및 누적 실행 시간순 랭킹
* 각 쿼리별로 응답 시간과 실행 횟수를 보여줌

* `--order-by` 옵션으로 정렬 순서를 변경 가능 (기본값: 단일 Query_time 이 가장 큰 쿼리가 먼저 나열)

* QueryID는 실행된 쿼리 문장을 정규화(쿼리에 사용된 리터럴을 제거)해서 만들어진 해시 값 -> 일반적으로 같은 모양의 쿼리라면 동일한 QueryID

### 4.4.3.3 쿼리별 실행 횟수 및 누적 실행 시간 상세 정보
* QueryID 별 쿼리를 쿼리 랭킹에 표시된 순서대로 자세한 내용을 보여줌