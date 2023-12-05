# 아키텍처

## MySQL 엔진과 스토리지 엔진

책을 읽으면 MySQL 엔진과 스토리지 엔진이 나누어져 있는 것을 볼 수 있습니다. 왜 MySQL은 MySQL 엔진고 스토리지 엔진을 구분해서 사용하는지 의문이 들었습니다.

MySQL엔진은 클라이언트의 접속 및 쿼리 요청을 처리하는 커넥션 핸들러와 SQL 파서, 옵티마이저가 중심을 이룹니다.

반면 스토리지 엔진은 실제로 디스크에 있는 데이터를 쓰거나 읽어오는 부분을 담당합니다. 또한 MySQL엔진은 하나지만 스토리지 엔진은 여러 개를 동시에 사용할 수 있게 됩니다.

제가 생각했을 때 MySQL 엔진과 스토리지 엔진을 구분한 이유는 다음과 같습니다.

만약 스토리지 엔진을 구분하지 않고 통합된 데이터베이스 엔진만을 사용하는 경우, 특정 요구 사항을 처리하는 데 제한 사항이 발생할 수 있습니다. 트랜잭션 지원 혹은 다양한 성능 최적화 요구를 충족시키기 위해서는 데이터베이스 엔진을 변경하거나 조정해야 할 것입니다. 그러나 통합된 데이터베이스 엔진의 경우 이러한 변경은 유지 보수의 어려움과 시스템 복잡도 증가와 같은 문제를 초래할 수 있다고 생각합니다.

[MySQL의 다양한 스토리지 엔진들](https://dev.mysql.com/doc/refman/8.0/en/storage-engines.html)

### 플러그인 스토리지 엔진 모델

MySQL의 독특한 구조 중 대표적인 것이 플러그인 모델입니다. 스토리지 엔진뿐만 아니라 다양한 기능들(전문 검색, 인증..)을 위한 플러그인들이 구현되어 제공됩니다. 왜 다양한 플러그인 스토리지 엔진 모델을 지원할까요? 그전에 먼저 쿼리가 실행되는 내용을 살펴보겠습니다.

MySQL에서 쿼리가 실행되는 과정은 크게 아래와 같습니다.

> { SQL파서 -> SQL옵티마이저 -> SQL 실행기 } -> { 데이터 읽기/쓰기 } -> Disk

위에서 데이터 읽기/쓰기 작업이 스토리지 엔진을 통해 수행이 됩니다. 즉 다른 스토리지 엔진을 사용하는 테이블에 대해 쿼리를 실행하더라도 MySQL 엔진에서 처리하는 내용은 대부분 동일하고 데이터를 읽기/쓰기 하는 작업이 스토리지 엔진에 따라 결정된다는 것입니다.

즉 데이터 읽기/쓰기 작업이 얼마나 달라지는지에 따라 프로그램의 요구사항에 맞추어 적절한 스토리지 엔진을 선택할 수 있습니다.

### 컴포넌트 아키텍처

MySQL 8.0 기존 플러그인 아키텍처를 대체하기 위해 컴포넌트 아키텍처가 제공됩니다. 어떤 단점이 있었길래 컴포넌트 아키텍처가 등장했을까요?

1. 플러그인은 오직 MySQL 서버와 인터페이스할 수 있고 플러그인끼리는 통신 불가
2. 플러그인은 MySQL 서버의 변수나 함수를 직접 호출하기 때문에 안전하지 않음
3. 플러그인은 상호 의존관계를 설정할 수 없어 초기화가 어려움

이에 반해 컴포넌트는 서버와 인터페이스할 수 있을 뿐만 아니라 다른 컴포넌트와도 통신을 할 수 있습니다. <br>
또한 컴포넌트가 설치되면 해당 컴포넌트에 의해 구현된 상태 변수가 노출되며, 이러한 변수들은 컴포넌트의 특정한 접두어(prefix)로 시작하게 됩니다. 예를 들어 `log_filter_dragnent` 오류 로그 필터 컴포넌트는 `log_error_filter_rules`라는 시스템 변수를 구현하며 이 변수의 전체 이름(fullname)은 `dragnet.log_error_filter_rules`입니다. 이렇게 되면 변수 이름 충돌을 방지하고 데이터베이스 시스템에서 변수가 어떤 컴포넌트에 속해 있는지 알기 명확해집니다.

## MySQL 스레딩 구조

MySQL은 프로세스 기반이 아닌 스레드 기반으로 동작합니다.

이유는 무엇일까요? 저는 다음과 같이 생각했습니다.

1. 요청마다 프로세스를 생성한다고 가정하면 프로세스를 생성하는 것은 메모리 자원을 사용하는 것이기 때문에 비싸며 스레드보다 컨텍스트 스위칭의 비용이 높습니다. 스레드는 프로세스에 비해 가벼우며, 컨텍스트 스위칭 비용도 낮습니다. 따라서 MySQL은 스레드를 사용해 더 많은 동시 요청을 처리할 수 있다고 생각합니다.
2. 스레드는 부모 프로세스의 자원을 공유하기 때문에 자원의 효율성과 통신의 단순화를 도와준다고 생각합니다.
3. 풀링을 통한 재사용을 수행해 생성 및 제거의 오버헤드를 줄일 수도 있습니다.

끝!