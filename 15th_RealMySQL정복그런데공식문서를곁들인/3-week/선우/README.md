## 페이지 압축 vs 테이블 압축

| 항목 | 페이지 압축 (Page Compression) | 테이블 압축 (Table Compression)    |
|:---|:---|:------------------------------|
| 저장 시점 | 디스크에 저장할 때 "페이지 단위"로 압축 | 디스크에 저장할 때 "블럭 단위"로 압축        |
| 메모리 적재 시 | 압축 해제하고 버퍼 풀에 적재 | 압축 해제하고 버퍼 풀에 적재              |
| 적용 방법 | `COMPRESSION="zlib"` 옵션 사용 | `ROW_FORMAT=COMPRESSED` 옵션 사용 |
| 운영체제 의존성 | 있음 (펀치 홀 필요) | 없음 (운영체제 상관없이 사용 가능)          |
| 사용 추천 상황 | 특수한 상황, 실무에서는 거의 안 씀 | 읽기 위주 테이블에서 유용                |
| 주의사항 | 파일 시스템, 하드웨어 제약 많음 | 데이터 변경이 많으면 비추                |

# 6.1 페이지 압축
- 16KB 페이지를 디스크에 쓰기 전에 압축함 (예: 7KB로)
- 디스크에는 압축된 상태로 저장
- 읽어올 때 압축 해제
- 버퍼 풀에서는 항상 압축 해제된 원본 페이지만 다룸 (16KB)

✅ 압축 전후에도 "페이지 단위(16KB)" 체계를 반드시 지킨다.

# 6.2 테이블 압축 (Table Compression)
- 블럭를 디스크에 저장할 때 압축해서 쌓음
- 테이블 포맷(`ROW_FORMAT=COMPRESSED`) 자체가 압축용 포맷
- 페이지도 압축된 블럭들로 채워짐
- 읽어오면 압축 해제해서 16KB 페이지에 풀림

✅ 압축된 블럭들로 페이지를 채우되, 메모리에선 풀어야 한다.

## 6.2.2 KEY_BLOCK_SIZE 결정
- InnoDB가 압축할 때 사용할 블록 크기를 지정하는 옵션이다.
- 압축 단위가 "페이지(16KB 전체)"가 아니라, 4KB 또는 8KB "블록" 단위로 압축을 수행한다.

### KEY_BLOCK_SIZE 설정의 특징
- 4KB 블록을 선택하면 압축률은 높지만, 페이지당 저장할 수 있는 레코드 수가 줄어 성능에 영향을 줄 수 있다.
- 8KB 블록을 선택하면 압축률은 낮지만, 레코드 밀도가 높아져 성능은 상대적으로 낫다.

### 4KB 블록과 8KB 블록 압축률 차이가 거의 없는 경우
- 압축이라는 건 기본적으로 데이터 안에 “중복”이나 “패턴”이 많을수록 효과가 크다.
- 이미 짧고 고르게 분산되어 있거나,
- 중복이 별로 없거나,
- 난수처럼 복잡한 데이터라면

## 6.2.3 압축된 페이지의 버퍼 풀 적재 및 사용

### unzip_LRU란?
- 테이블 압축을 쓰면 디스크에는 압축된 데이터만 저장된다.
- 그런데 버퍼 풀에 올릴 때는 압축을 풀어야 한다.
- 디스크에는 압축본이 필요하고, 메모리에는 풀린 데이터가 필요하다. 이걸 효율적으로 관리하려고 만든 게 unzip_LRU 이다.

### 왜 unzip_LRU가 필요할까?
- 압축 해제본을 메모리에 잠깐 들고 있으면서 디스크 I/O 비용을 줄이는 목적이다.

### 올라가 있는 데이터 한눈에 보기

```json
디스크
 └─ 압축된 데이터(4~8KB)
     ↓
버퍼 풀
 ├─ LRU 리스트
 │    ├─ 일반 테이블 데이터 페이지 (16KB)
 │    ├─ 압축 테이블의 compressed page (4~8KB)
 └─ Unzip_LRU 리스트
      └─ 압축 테이블의 압축 해제본 (16KB)
```
## 단순 궁금증 해결해보자

### 1. 왜 굳이 두 가지나 있을까?

**답변:** 둘이 "목표"가 약간 다르기 때문.

- 페이지 압축 → 디스크 I/O 최적화 (파일 시스템 레벨에서 공간 아끼기)
- 테이블 압축 → 디스크 저장 최적화 (포맷 자체를 바꿔서 더 많은 데이터 저장)

추가로, 페이지 압축은 펀치 홀 기능 의존성 때문에 제약이 많아서,  
**테이블 압축이 실무에서는 훨씬 많이 사용된다.**

### 2. 압축했으면 메모리에서도 적게 쓰는 거 아닌가?

**답변:** 아니다. 둘 다 메모리에 올릴 때는 **압축을 해제**해서 올린다.

- 버퍼 풀(메모리)은 압축된 페이지를 저장하지 않는다.
- (일부 압축 캐시를 저장할 수 있지만 부가적)
- 결국 버퍼 풀 공간은 줄어들지 않고, 오히려 더 많이 차지할 수도 있다.

✅ 디스크 공간은 줄어들지만, 메모리 공간은 줄지 않는다.

### 3. 그러면 압축을 쓰면 무조건 좋은 것인가?

**답변:** 아니다. 트레이드오프(trade-off)가 있다.

| 압축 사용 | 결과 |
|:---|:---|
| 디스크 공간 절약 | 좋다 |
| 디스크 I/O 감소 | 좋을 수 있다 |
| CPU 오버헤드 발생 (압축/해제 작업) | 나쁘다 |
| 버퍼 풀 공간 활용률 하락 | 나쁘다 |
| 데이터 변경이 잦으면 압축률 떨어짐 | 나쁘다 |

특히,
- UPDATE, DELETE 같은 변경이 많은 테이블에서는 오히려 성능이 나빠질 수도 있다.
- 압축 해제 작업 때문에 쿼리 지연 시간(latency)이 늘어날 수 있다.

✅ 테이블 압축은 읽기 위주(read-heavy) 테이블에 적합하다.

### 4. 압축 테이블(ROW_FORMAT=COMPRESSED)에서 INSERT할 때, AUTO_INCREMENT가 아니라 UUID 같은 랜덤 키를 쓰면 비효율적일까?

**답변:** 비효율적이다.

- AUTO_INCREMENT는 항상 오름차순으로 값이 증가하니까, 항상 B+트리 가장 오른쪽 리프 노드에만 INSERT 하면 된다.  
  -> 디스크 페이지를 잘 건드리지 않는다. (localized writes)

- 반면, UUID는 값이 완전 랜덤이라 매번 새로운, 다양한 페이지를 건드려야 한다.  
  -> 압축 해제와 재압축이 엄청 자주 발생한다.
