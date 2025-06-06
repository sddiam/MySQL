# 데이터 압축
> MySQL 서버에서 디스크에 저장된 데이터 파일의 크기는 일반적으로 쿼리의 처리 성능과도 직결되지만 백업 및 복구 시간과도 밀접하게 연결된다.
> 디스크의 데이터 파일이 크면 클수록 쿼리를 처리하기 위해서 더 많은 데이터 페이지를 InnoDB 버퍼 풀로 읽어야 할 수도 있고, 새로운 페이지가 버퍼 풀로 적재되기 때문에 그만큼 더티 페이지가 더 자주 디스크로 기록돼야 한다.
> 그리고 데이터 파일이 크면 클수록 백업 시간이 오래 걸리며, 복구하는데도 그만큼의 시간이 걸린다. 물론 그만큼의 저장공간이 필요하기 때문에 비용 문제고 있을 수 있다.
> 많은 DBMS가 이런 문제점을 해결하기 위해 데이터 압축 기능을 제공한다.

## 1. 페이지 압축
MySQL 서버가 디스크에 저장하는 시점에 데이터 페이지가 압축되어 저장되고, 반대로 MySQL 서버가 디스크에서 데이터 페이지를 읽어올 때 압축이 해제된다.(Transparent Page Compression)
즉, 버퍼 풀에 데이터 페이지가 한 번 적재되면 InnoDB 스토리지 엔진은 압축이 해제된 상태로만 데이터 페이지를 관리한다. 그래서 MySQL 서버의 내부 코드에서는 압축 여부와 관계없이 '투명(Tranparent)'하게 작동한다.

여기서 문제점이 존재하는데, 16KB 데이터 페이지를 압축한 결과가 용량이 예측 불가한데 적어도 하나의 테이블은 동일한 크기의 페이지(블록)로 통일돼야 한다는 것이다.
그래서 페이지 압축 기능은 운영체제 별로 특정 버전의 파일 시스템에서만 지원되는 펀치 홀(Punch hole)이라는 기능을 사용한다.

___________________________________________________________________________________________________
1) 16KB 페이지를 압축(압축 결과를 7KB로 가정)
2) MySQL 서버는 디스크에 압축된 결과 7KB를 기록 (이때 MySQL 서버는 압축 데이터 7KB에 9KB의 빈 데이터를 기록)
3) 디스크에 데이터를 기록한 후, 7KB 이후의 공간 9KB에 대해 펀치 홀을 생성
4) 파일 시스템은 7KB만 남기고 나머지 디스크의 9KB 공간은 다시 운영체제로 반납
_____________________________________________________________________________________________________
*^ 운영체제의 블록 사이즈 512바이트, 16KB 크기의 페이지를 압축할 경우의 작동 방식 ^*
9KB의 펀치 홀이 생성된 상태에서는 실제 디스크의 공간은 7KB만 차지한다. 하지만 운영체제에서 16KB를 읽으면 압축된 데이터 7KB와 펀치 홀 공간인 9KB를 합쳐서 16KB를 읽는다.

하지만 펀치 홀 기능은 운영체제뿐만 아니라 하드웨어 자체에서도 해당 기능을 지원해야 사용 가능하다. 그리고 아직 파일 시스템 관련 명령어가 펀치 홀을 지원하지 못하다는 것이다.
## 2. 테이블 압축
