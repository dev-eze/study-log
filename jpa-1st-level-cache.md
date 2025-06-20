# jpa 1차 캐시
- 1차 캐시란 JPA에서 EntityManager 단위로 관리되는 캐시  
- 트랜잭션 범위 내에서 동일한 EntityManager를 통해 조회된 엔티티는 메모리에 보관된다.  
즉 같은 EntityManager에서 동일한 엔티티를 조회하면 DB에 접근하지 않고 캐시된 객체를 반환한다.
