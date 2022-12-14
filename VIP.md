# Virtual IP

## 요약
- Oracle RAC 환경에서 사용한다.
- 클라이언트의 TCP 타임아웃 관리용이다.
- 여러개의 Oracle 노드 운영중에 한 노드가 다운되면 VIP로 연결을 시도한 클라이언트는 노드가 다운되었다는 것을 즉시 알 수 있다.(ORA-12541) 하지만, public IP에 직접 연결한 클라이언트는 로드밸런싱으로 연결해도 다운된 노드에 연결을 시도하고 긴 시간동은 대기하는 상황에 빠질 수 있다. 클라이언트가 첫번째로 연결을 시도한 노드가 다운된 상태이면 다음 노드로 시도할 때까지 기다리는 시간은 클라이언트나 클라이언트의 OS의 TCP 관련 파라미터에 의해 결정된다. VIP는 모든 클라이언트에게 즉시 응답할 수 있고 다운된 노드로 접속하는 클라이언트는 다음 노드로 연결을 시도할 수 있게 한다. 
- ~~같은 VIP에 1개 이상의 노드를 설정할 수 있는 것 같다. 만약 2개 노드가 1개 VIP를 가지고 있다면 1번 노드가 VIP를 사용하고 2번 노드는 stand-by 상태인 것으로 보인다. 그리고, 사용중인 1번 노드가 다운되면 2번노드가 이 VIP를 차지하게 되고 전환하는 시간동안에는 ORA-12541 상태가 되어 클라이언트는 다른 VIP로 연결을 시도할 수가 있다.~~

## 관련 설정
https://docs.oracle.com/cd/E11882_01/network.112/e10835/sqlnet.htm#NETRF210
```
SQLNET.EXPIRE_TIME=10
SQLNET.INBOUND_CONNECT_TIMEOUT=3
```

## 참고
https://docs.oracle.com/database/121/RACAD/GUID-6C72F02D-BB43-4C56-9F46-244C8D6BB670.htm#RACAD8950
https://logicalread.com/virtual-ip-with-oracle-rac-mc04/#.Yz5BSHZBxD8
https://docs.oracle.com/cd/E11882_01/network.112/e10835/sqlnet.htm#NETRF210

## Failover TNS 작성
FAILOVER_MODE
- TYPE: 
    - SESSION
        - 재접속이 필요없음
        - select 구문으로 fetch 도중 장애 발생시 실패
        - 세션이 자동으로 다른 인스턴스에 접속
    - SELECT
        - 재접속 불필요
        - Select 구문으로 fetch하던 레코드를 복구함
        - 세션이 자동으로 다른 인스턴스에 접속
- METHOD: 
    - BASIC
        - On-demand 방식
        - Failover 발생시 정상 인스턴스로 oracle server process를 가동
        - 2개 노드 운영시 한개 노드로 세션이 쏠리는 현상(과부가 발생 가능)
    - PRECONNECT
        - 다른 노드에 미리 oracle server process를 가동해 두고 장애시 전환하는 오버헤드를 줄일 수 있음
        - 불필요한 프로세스를 미리 생성해 두는 것이기 때문에 자원 낭비

```
TNSNAME =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = vip1)(PORT = 1521))
    (ADDRESS = (PROTOCOL = TCP)(HOST = vip2)(PORT = 1521))
    (LOAD_BALANCE = no)
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = TESTDB)
      (FAILOVER_MODE =
        (TYPE = SELECT)
        (METHOD = BASIC)
      )
    )
  )
```