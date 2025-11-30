## 커버링 인덱스가 무엇인가? (커버링 인덱스로 처리되는 쿼리는 디스크의 레코드를 읽지 않아도 됨)
- 커버링 인덱스(Covering Index)는 쿼리가 필요한 모든 컬럼이 인덱스 안에 이미 들어 있어서, 실제 테이블(PK 인덱스의 row)을 읽을 필요가 없는 상황

> ### 예)
> - 인덱스를 다음과 같이 생성했을 때, (멀티 컬럼 인덱스)
> ```sql
> CREATE INDEX idx_email_name_age ON users(email, name, age);
> ```
> 
> ```sql
> SELECT name, age
> FROM users
> WHERE email = 'aa@bb.com';
> ```
> - 위와 같은 쿼리를 실행시킨다면, 세컨더리 인덱스에서 원하는 컬럼을 다 찾을 수 있음 -> 데이터파일에서 값을 꺼내올 필요가 없음 (이미 데이터가 인덱스에 모두 존재)