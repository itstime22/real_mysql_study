### (3주차) : 자동 슬로우 쿼리 감지 기능 JPA 옵션
```yml
spring:  
  jpa:
    properties:
      hibernate:        
        session.events.log.LOG_QUERIES_SLOWER_THAN_MS: 5
        
        
logging:
  level:
    org.hibernate.SQL_SLOW: info
```
- 위 옵션으로 일정 시간이 넘는 쿼리는 자동으로 어떤 쿼리가 느렸는지 알려줌

### InnoDB 스토리지 엔진은 REPEATABLE READ 격리수준이지만 넥스트 키 락을 사용하여 팬텀 리드(PHANTOM READ)를 없앰 -> 그러면 Serializable 격리수준과 같은 게 아닌가?
- 결론은 아님
- InnoDB 스토리지 엔진은 범위 검색에서만 넥스트 키 락을 걸어서 Phantom Read를 막을 뿐
- 그 이외의 동시성 제어에서는 Serializable보다 느슨
> #### Serializable
> - SELECT 연산에서도 락을 검
> #### REPEATABLE READ
> - SELECT 연산에서는 락을 걸지 않음