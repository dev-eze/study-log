
# ⭐️ 정규화  
> 데이터베이스 설계 시 **데이터의 중복을 줄이고**,  
> **종속 관계를 명확히 하여 무결성을 유지**하기 위해 수행하는 과정입니다.
> 과도한 조인과 중복 데이터 최소화 사이에서 적정 수준을 선택해서 진행해야 한다.

---

## ⭐️ 정규화를 하는 이유?
1. **중복 데이터 제거**  
   → 저장공간 낭비 방지, 데이터 관리 용이

2. **데이터 무결성 유지**  
   → 하나의 데이터만 수정해도 일관성이 유지됨

3. **갱신·삭제·삽입 이상 방지 (Anomaly 방지)**  
   → 예외 상황에서의 데이터 오류를 줄임 (데이터 갱신/삭제/삽입 이상 방지 )

---

## ⭐️ 정규화 단점

1. 많은 join 으로 인한 조회 성능 저하  
   - 일부 중복을 허용하는 반정규화
   - 조회 전용 DB구성
     -- 조회용 database를 두는 database per service + CQRS 패턴 적용으로 개선 가능  

---

## ⭐️ 정규화 단계

- 1NF → 2NF → 3NF → BCNF 순으로 진행
- 2NF를 만족한다는것은 1NF를 만족함을 의미
- 이 외 다른 단계들이 존재하지만 실무에서는 보통 3NF, BCNF까지 진행한다. 


### ✅  제 1정규형  
- 모든 속성이 원자값을 갖도록 설계.  
- 각 컬럼에 복수값 금지하여 하나의 값만 가지도록 설계.  
📌 예) 상품 테이블에 상품명 셀에 여러 상품명.

### ✅  제 2정규형  
- 부분 종속 제거 (복합 기본키가 있을 경우, 그 일부에만 종속된 속성 제거.)  
- 모든 속성이 전체 기본키에 완전 종속되도록 설계.
- non-prime 속성이 전체 기본키에 종속되도록 설계 해야 한다. 
📌 예) 중복, 반복되는 값이 없도록 설계

### ✅  제 3정규형  
- 이행적 종속 제거.  
-  속성 간 종속 제거. (기본키가 아닌 컬럼에 다른 컬럼이 종속되는 경우 제거)
-  non-prime 속성과 non-prime 속성 사이에는 FD(Functional dependency), 연관짓는 디펜던시가 있으면 안된다. 
📌 예) 주문 테이블에 고객명, 고객주소 컬럼이 설계된 경우 고객명에 고객주소가 종속됨. 고객 정보를 별도 테이블에 분리해야 함.

### ✅  BCNF  
- 3NF를 만족하면서도, 후보키 간 결정자 이상 제거
- 한 테이블에 여러 후보키가 있는 경우, 후보키도 모든 결정자가 되어야함.


## ✍️ 개인 회고

> 정규화에 대해 설명할 일이 있었는데,  
> 개념은 러프하게 알고 있었지만 명확하고 간결하게 전달하지 못해 아쉬웠다.  
> 이번 기회에 다시 정리하면서 개념을 더 명확히 잡을 수 있었다.
