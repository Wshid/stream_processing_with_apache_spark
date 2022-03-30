# [CHAP.13] 고급 상태 기반 작업
- CHAP.08 - 구조적 스트리밍에서의 집계
- CHAP.12 - 이벤트 시간 처리
- 단, **기본 제공 모델**에서 직접 지원하지 않는 **사용자 지정 집계 기준**을 충족해야하는 경우 존재
  - 이 상황을 해결하기 위해 **고급 상태 기반 작업**을 수행하는 방법
- **구조적 스트리밍**은 **임의의 상태 기반 처리**를 구현하는 API 제공
  - 이 `API`는 `mapGroupsWithState` 및 `flatMapGroupsWithState`의 두가지 작업으로 표현
  - 두 작업 모두 **상태**의 **사용자 정의**를 만들고
  - 시간이 지남에 따라 새로운 데이터가 들어올 때,
    - 이 상태가 **어떻게 발전하는지**에 대한 규칙을 설정
    - **만료시기** 결정
  - **상태 정의**를 `들어오는 데이터`와 결합하여 **결과를 산출하는 방법** 제공
- `mapGroupsWithState` 및 `flaptMapGroupsWithState`의 차이점
  - `mapGroupsWithState`: 각 처리된 그룹에 대해 **단일 결과** 생성
  - `flaptMapGroupsWithState`: `0개 이상`의 결과 생성
  - 의미론적으로
    - `새로운 데이터`가 항상 `새로운 상태`로 귀결 => `mapGroupsWithState` 사용
    - `다른 모든 경우` => `flatMapGroupsWithState` 사용
- 내부적으로 **구조적 스트리밍**은
  - **운영 간의 상태 관리**를 담당하며
  - 시간이 지남에 따라 **스트리밍 프로세스**동안 **가용성**과 **내결함성 보존**을 보장

## 13.1. 예제: 차량 유지 보수 관리
- 각 차량은 지리적 `위치`, `연료 레벨`, `가속도`, `베어링`, `엔진 온도` 등과 같은 많은 **작동 파라미터**를
  - 정기적으로 보고
- 이 원격 측정 데이터를 활용하여
  - 비즈니스 운영 및 재무 측면 관리에 도움이 되는 App 개발
- 여행의 개념을 `출발부터 정지`까지의 **주행 도로 구간**으로 가정
  - 개별적으로 **여행**의 개념 => **연료 효율**을 계산하거나 **가상 경계 협정 준수**를 감시
  - 그룹으로 분석하면
    - `교통 패턴`, `교통 핫스팟`을 드러낼 수 있음
  - 다른 센서 정보와 결합시, `도로 상태` 보고 가능
- 스트림 처리 관점에서는
  - 여행을 `차량이 움직이기 시작할 때 열리고, 정지할 때 닫히는 임의의 윈도우`로 볼 수 있음
- 순전히 **시간**에 근거하는 것이 아닌
  - 임의의 조건에 근거한 **강력한 상태 정의**가 필요
- 예제에서 이 조건은 **차량이 주행중**임을 의미

## 13.2. 상태 작동을 통한 그룹의 이해
- 임의 작동 상태인 `mapGroupsWithState` 및 `flatMapGroupWithState`는
  - `scala` 또는 `java` binding을 사용하여 **입력한 데이터셋 API**에서만 동작
- 처리하고 있는 **데이터**와, **상태 기반 변환**을 요건을 바탕으로
  - 세 가지 유형 정의를 제공해야 함
  - 전형적으로 `case class`또는 `java Bean`으로 인코딩
- **세 가지 유형**
  - 입력 이벤트(I)
  - 유지할 임의의 상태(S)
  - 출력(O) - 이 타입은 적절한 경우 **상태 표현**과 동일할 수 있음
- 이러한 모든 타입은 `Spark SQL`로 **인코딩**할 수 있어야 함
  ```scala
  import spark.implicits._
  ```
  - 모든 **기본 타입**, **튜블** 그리고 `case class`에 충분
- 이러한 타입을 사용할 경우, **사용자 지정 상태 처리 논리**를 구현하는 **상태 변환 함수**를 공식화 가능
  - `mapGroupsWithState`에서는 이 함수가 **단일 필수 값**을 리턴할 것을 요청
    ```scala
    def mappingFunction(key: K, values: Iterator[I], state: GroupState[S]): O
    ```
  - `flatMapGroupsWithState`에서는 이 함수가 `0개 이상의 요소를 포함`할 수 있는 `Iterator`를 리턴
    ```scala
    def flatMappingFunction(
        key: K, values: Iterator[I], state: GroupState[S]): Iterator[O]
    ```
    - `GroupState[S]`는 **구조적 스트리밍**에서 제공하는 **래퍼**이며
      - 내부적으로 **실행 전반**에 걸쳐 상태 `S`를 관리하는데 사용
    - 이 함수내에서, `GroupState`는
      - 상태에 대한 `mutation` 액세스와 **시간 초과**를 확인 및 설정하는 기능 제공

#### Caution: mappingFunction/FlatMappingFunction의 구현은 직렬화 가능해야 함
- Runime에 이 함수는 **자바 직렬화**를 사용하여 클러스터의 **Executor**에 배포
- 이 요구사항은 **카운터**나 기타 **뮤테이블 변수**(mutable variable)와 같은 **국소 상태**를
  - **함수에 본문에 포함시키지 않아야 한다**는 결과를 가지고 있음
- 모든 **관리 상태**를 **상태 표시 클래스**에 **캡슐화**해야 함


## 13.3. MapGroupsWithState의 사용
- 슬라이딩 윈도우
  - **시간 윈도우**를 기준으로 **이동 평균** 계산
  - 윈도우에서 `찾은 요소 수와 상관 없이` 결과 생성
- `마지막 10개 요소`의 **이동 평균**을 계산하기
  - `필요한 요소 수`를 가져오는데 시간이 얼마나 걸릴지 모름
  - 시간 윈도우를 사용할 수 없음
  - 대신 `MapGroupsWithState`와 함께 **사용자 지정 상태**를 사용하여
    - **자체 카운트 기반 윈도우** 정의 가능
- 온라인 리소스 참조
  - `map_groups_with_state` 노트북
  - https://github.com/stream-processing-with-spark

#### 스트리밍 데이터셋 초기화
```scala
// 기상 관측소 이벤트의 표현
case class WeatherEvent(stationId: String,
  timestamp: Timstamp,
  location: (Double, Double),
  pressure: Double,
  temp: Double)

val weatherEvents: Dataset[WeatherEvents] = ...
```

#### 상태 정의
- 상태(`S`)를 정의
- `최신 n개 요소를 유지`하고, 더 오래된 것을 삭제
- Queue와 같은 `FIFO` 컬렉션 적용
- 최신 요소가 **큐 앞**에 추가되고, 최신 `n`을 유지하고, 이전 요소를 삭제
- 상태 정의는
  - 사용을 용이하게 하는 몇 가지 **헬퍼 메서드**가 있는 **큐**가 지원하는 `FIFOBuffer`가 됨
- 코드
  ```scala
  case class FIFOBuffer[T](
    capacity: Int, data: Queue[T] = Queue.empty
  ) extends Serializable {
    def add(element: T): FIFOBuffer[T] = this.copy(data = data.enqueue(element).take(capacity))
    def get: List[T] = data.toList
    def size: Int = data.size
  }
  ```

### 출력 유형 정의
- 출력 유형(`O`) 정의
- **상태 기반 연산**의 바람직한 결과는
  - 입력 `WeatherEvent`에 있는 **센서값**의 **이동 평균**
- 또한 연산에 사용된 값의 **시간 범위**를 알고 싶기 때문에
  - 이를 기반으로 출력 유형 `WeatherEventAverage` 설계
- 코드
  ```scala
  import java.sql.Timestamp
  case class WeatherEventAverage(stationId: String,
                                 startTime: Timestamp,
                                 endTime: Timestamp,
                                 pressureAvg: Double,
                                 tempAvg: Double)
  ```
- 위와 같은 타입을 정의하면,
  - **기존 상태**와 **새 요소**를 **결과**로 결합하는 `mappingFunction`을 계속 만들 수 있음

#### CODE.13.1. 카운트 기반 이동 평균에 mapGroupsWithState 사용
- `Groups` 랩퍼가 제공하는 함수를 통해 **내부 상태**를 업데이트 하는 역할도 수행
- 상태를 `null`로 업데이트할 수 없음 => `IllegalArgumentException`이 발생
- 상태를 **제거**하려면 `state.remove()` 메서드 활용
- 코드
  ```scala
  def mappingFunction(
    key: String,
    values: Iterator[WeatherEvent],
    state: GroupState[FIFOBuffer[WeatherEvent]]
  ): WeatherEventAverage = {
    // 윈도우의 크기
    val ElementCountWindowSize = 10

    // 현재 상태값을 받거나, 상태값이 존재하지 않는다면 새롭게 생성
    val currentState = state.getOption
      .getOrElse(
        new FIFOBuffer[WeatherEvent](ElementCountWindowSize)
    )
    
    // 새로운 이벤트가 발생하면, 상태값 변화
    val updatedState = values.foldLeft(currentState) {
      case (st, ev) => st.add(ev)
    }

    // 최신화된 상태로 상태값 갱신
    state.update(updatedState)

    // 데이터가 충분하면 상태에서 `WeatherEventAverage`를 생성하고
    // 그렇지 않다면 `0`으로 기록
    val data = updatedState.get
    if (data.size > 2) {
      val start = data.head
      val end = data.last
      val pressureAvg = data
          .map(event => event.pressure)
          .sum / data.size
      val tempAvg = data
          .map(event => event.temp)
          .sum / data.size
        WeatherEventAverage(
          key,
          start.timestamp,
          end.timestamp,
          pressureAvg,
          tempAvg
        )
    } else {
      WeatherEventAverage(
        key,
        new Timestamp(0),
        new Timestamp(0),
        0.0,
        0.0
      )
    }
  }
  ```
- 스트리밍 `Dataset`의 상태 기반 변환을 위해 `mappingFunction`을 사용
  ```scala
  val weatherEventsMovingAverage = weatherEvents
      .groupByKey(record => record.stationId)
      .mapGroupsWithState(GroupStateTimetout.ProcessingTimeTimeout)(mappingFunction)
  ```
  - 먼저 도메인의 주요 식별자로, `Group` 생성
    - 이 예시에서는 `stationId`를 의미
  - `groupByKey` 작업은 `[map|flatMap]GroupWithState` 작업의 **진입점**이 되는
    - 중간 구조인 `KeyValueGroupedDataset`을 생성
  - 매핑 기능 외에도 **타임아웃 유형** 제공 필요
    - 이 유형은 `ProcessingTimeTimeout`이거나 `EventTimeTimeout`일 수 있음
  - **상태 관리**를 위한 **이벤트 타임스탬프**에 의존하지 않기 때문에 `ProcessingTimeTimeout`을 선택

### 쿼리 결과
- 콘솔 싱크를 사용한 쿼리 결과 도출
- 코드
  ```scala
  val outQuery = weatherEventsMovingAverage.writeStream
      .format("console")
      .outputMode("update")
      .start()
  ```

## 13.4. FlatMapGroupsWithState 사용
- `mapGroupsWithState` 사용시의 맹점
  - 스트림 처리를 시작하고, **이동 평균**을 계산하는 데 필요한 것으로 간주되는 모든 요소를 수집하기 전
  - `mapGroupsWithState` 작업에서 `0`이 없는 값을 생성
- `mapGroupsWithState`는 모든 **트리거 간격**에서 처리되는 각 **그룹**에 대해
  - **단일 레코드**를 생성하기 위한 **상태 처리 기능**이 필요
- 이는 각 **키**에 해당하는 **새로운 데이터**의 도착이
  - 자연스럽게 상태를 갱신한다면 문제 x
- 단, **상태 로직**이 결과를 생성하기 전 **이벤트가 발생해야 하는 경우**가 존재
  - 현재 예제에서는 **그들의 대한 평균 계산 전 `n`개 요소가 필요**
  - 다른 시나리오에서는 **단일 수신 이벤트**가 여러 **임시 상태**를 완료하여
    - 둘 이상의 결과 생성 가능
      - 예시: 단일 대중교통 목적지 도착시, `승객의 여행상태`가 업데이트 됨
- `flatMapGroupsWithState`는 **상태 처리 함수**가 `0`개 이상의 요소를 포함할 수 있는
  - 결과 이터레이터를 생성하는 `mapGroupsWithState`의 일반화

### 온라인 예시
- `mapgroupswithstate-n-moving-agerage` 노트북 참고

#### CODE.13.2. 카운트 기반 이동 평균에 FlatMapGroupsWithState 사용
- 결과 이터레이터를 반환하려면 **매핑 함수**를 업데이트 해야 함
- 이터레이터는 `평균`을 계산할 충분한 값이 없을 때 원소를 `0`개 포함하고
  - 그렇지 않을 경우 값을 포함
```scala
def flatMappingFunction(
  key: String,
  values: Iterator[WeatherEvent],
  state: GroupState[FIFOBuffer[WeatherEvent]]
): Iterator[WeatherEventAverage] = {
  val ElementCountWindowSize = 10

  // 현재 상태값을 받거나, 상태값이 존재하지 않는 다면 새롭게 생성
  val currentState = state.getOption
    .getOrElse(
      new FIFOBuffer[WeatherEvent](ElementCountWindowSize)
    )
  
  // 새로운 이벤트가 발생하면 상태값 변화
  val updatedState = values.foldLeft(currentState) {
    case (st, ev) => st.add(ev)
  }

  // 최신화된 상태로 상태값을 갱신
  state.update(updatedState)

  // 충분한 데이터가 있을 때만, 상태에서 WeatherEventAverage 생성
  // 그 전까지는 empty 값을 결과로 반환
  val data = updatedState.get
  if (data.size == ElementCountWindowSize) {
    val start = data.head
    val end = data.last
    val pressureAvg = data
      .map(event => event.pressure)
      .sum / data.size
    val tempAvg = data
      .map(event => event.temp)
      .sum / data.size
    
    Iterator(
      WeatherEventAverage(
        key,
        start.timestamp,
        end.timestamp,
        pressureAvg,
        tempAvg
      )
    )
  } else {
    Iterator.empty
  }

  val weatherEventsMovingAverage = weatherEvents
    .groupByKey(record => record.stationId)
    .flatMapGroupsWithState(
      OutputMode.Update,
      GroupStateTimeout.ProcessingTimeTimeout
    )(flatMappingFunction)
}
```
- `flatMapGroupsWithState`를 사용하면, 더 이상 인공적인 제로 레코드 생성 필요 x
- 또한, 상태 관리 정의는
  - 결과를 생성하기 위해 `n`개 요소를 갖는 것에 엄격함

### 13.4.1. 출력 모드
- `map`과 `flatMapGroupsWithState` 작업간 결과에서의 **카디널리티** 차이는
  - 실제 `API 차이`와 거의 같지 않지만
  - 더 큰 결과를 초래
- `flatMapGroupsWithState`에는 **출력 모드**의 추가 사양이 필요
  - **상태 기반 작업**의 **레코드 생산 의미**에 대한 정보를 **다운스트림 프로세스**에 제공하는 데 필요
- 결과적으로 **구조적 스트리밍**은
  - **다운스트림 싱크**에 대해 허용된 **출력 작업**을 계산하는데 도움이 됨

#### flatMapGroupsWithState에 지정된 출력 모드
- `update`
  - 생성된 레코드가 **최종이 아님**
  - 나중에 새로운 정보로 업데이트 될 수 있는 **중간 결과**
  - 이전 예제에서, 키에 대한 새 데이터가 도착하면, 새 데이터 포인트 생성
  - **다운 스트림 싱크**는 `update`를 사용해야 하며, 어떤 집계도 `flatMapGroupsWithState`작업을 따를 수 없음
- `append`
  - 그룹에 대한 결과를 생성하는 데 필요한 **모든 정보를 수집했으며**
  - 들어오는 이벤트가 **해당 결과를 변경하지 않음**을 의미
  - 다운스트림 싱크는 `append`모드를 사용하여 작성해야 함
  - `flatMapGroupsWithState`를 적용하면
    - 최종 레코드가 생성되므로, 해당 결과에 추가 집계를 적용할 수 있음

### 13.4.2. 시간 경과에 따른 상태 관리
- 시간 경과에 따른 **상태 관리**의 중요한 요건은
  - 안정적인 **작업셋**(working set)을 확보하는 것
- 공정에 의해 요구되는 **메모리**는
  - 시간이 지남에 따라 **경계**를 이루며
  - **변동**을 허용할 수 있도록 **가용 메모리 아래** 안전한 거리에 머무름
- `CHAP.12`에서 보았던 **시간 기반 윈도우**와 같은 관리되는 상태 기반 집계에서
  - **구조적 스트리밍**은 **사용되는 메모리양 제한**을 위해
  - 내부적으로 **상태** 및 만료된 것으로 간주되는 **이벤트를 제거**하는 메커니즘 관리
  - `map|flatMapGroupsWithState`에서 제공하는 **사용자 지정 상태 관리 기능**을 사용할 때
    - 이전 상태를 제거하는 책임을 져야 함
- 구조적 스트리밍은 **특정 상태의 만료 시기**를 결정하는데 사용할 수 있는
  - **시간** 및 **타임아웃 정보**를 노출

#### 만료시기 결정 첫번째 단계: 사용할 시간 기준 정하기
- 타임아웃은 **이벤트 시간** 또는 **처리 시간**을 기준으로 함
- 그 선택은 구성 중인 특정 `map|flatMapGroupsWithState`에 의해 처리되는 상태에 **전역적**
- 타임아웃 유형은 `map|flatMapGroupsWithState`를 호출할 때 지정

#### 처리시간을 기준으로 처리
  ```scala
  val weatherEventsMovingAverage = weatherEvents
    .groupByKey(record => record.stationId)
    .mapGroupsWithState(GroupStateTimeout.ProcessingTimeTimeout)(mappingFunction)
  ```

#### 이벤트 시간을 사용하기
- 이벤트 시간을 사용하려면 **워터마크 정의** 필요
- 이 정의는 이벤트의 **타임스탬프 필드**와 **워터마크**의 구성된 지연으로 이루어짐
  ```scala
  val weatherEventsMovingAverage = weatherEvents
    .withWatermark("timestamp", "2 minutes")
    .groupByKey(record => record.stationId)
    .mapGroupsWithState(GroupStateTimeout.EventTimeTimeout)(mappingFunction)
  ```

#### 타임아웃 타입
- 타임아웃 타입은 **시간 참조**의 **전역적인 소스** 선언
- 타임아웃이 필요하지 않는 경우를 위한 `GroupStateTimeout.NoTimeout` 옵션도 존재
- `Timeout`의 실젯값은 `GrouopState`에서 사용한 `state.setTimeoutDuration`또는
  - `state.setTimeoutTimestamp` 메서드를 사용하여, 각 그룹별로 관리
- 상태가 만료되었는지 확인하기 위해 `state.hasTimedOut`을 확인
- 타임아웃 상태가 되면, 타임아웃된 그룹에 대한 **빈 값의 이터레이터**가 포함된
  - `(flat)MapFunction`에 대한 호출이 발생
- 타임아웃 기능 사용
  - 상태를 이벤트로 변환하는 것 포함
  ```scala
  def stateToAverageEvent(
    key: String,
    data: FIFOBuffer[WeatherEvent]
  ): Iterator[WeatherEventAverage] = {
    if (data.size = ElementCountWindowSize) {
      val events = data.get
      val start = events.head
      val end = events.last
      val pressureAvg = events
        .map(event => event.pressure)
        .sum / data.size
      val tempAvg = events
        .map(event => event.temp)
        .sum / data.size
      Iterator(
        WeatherEventAverage(
          key,
          start.timestamp,
          end.timestamp,
          pressureAvg,
          tempAvg
        )
      )
    } else {
      Iterator.empty
    }
  }
  ```
- 데이터가 유입되는 일반적인 시나리오뿐 아니라,
  - 타임아웃의 경우에도
  - **상태를 변환**하기 위해 **새로운 추상화**를 사용할 수 있음

#### CODE.13.3. flatMapGroupsWithState에서 타임아웃 사용
```scala
def flatMappingFunction(
  key: String,
  values: Iterator[WeatherEvent],
  state: GroupState[FIFOBuffer[WeatherEvent]]
): Iterator[WeatherEventAverage] = {
  // 우선 상태에서 시간 초과 확인
  if (state.hasTimedOut) {
    // 상태값이 timeout이라면 값이 비어 있음
    // 이 검증은 단지 요청을 설명하기 위한 것
    assert(
      values.isEmpty,
      "When the state has a timeout, the values are empty"
    )
    val result = stateToAverageEvent(key, state.get)
    // 시간 초과 상태(timeout) 제거
    state.remove()
    // 현재 상태를 출력 레코드로 변환한 결과를 내보냄
    result
  } else {
    // 현재 상태값을 받거나 상태값이 존재하지 않는다면 새롭게 생성
    val currentState = state.getOption.getOrElse(
      new FIFOBuffer[WeatherEvent](ElementCountWindowSize)
    )
    // 새로운 이벤트 발생하면 상탯값을 변화시킴
    val updatedState = values.foldLeft(currentState) {
      case (st, ev) => st.add(ev)
    }
    // 최신화된 상태로 상태값 갱신
    state.update(updatedState)
    state.setTimeoutDuration("30 seconds")
    // 충분한 데이터가 있을 때만 상태에서 WeatherEventAverage 생성
    // 그 전까지는 empty 값을 결과로 반환
    stateToAverageEvent(key, updatedState)
  }
}
```

#### 타임아웃이 실제로 시간 초과되는 경우
- 구조적 스트리밍에서 **타임 아웃**의 의미는
  - 시계가 **워터마크**를 지나기 전에 **이벤트**가 시간초과 되지 않도록 보장
  - 이것은 우리의 **타임아웃 직관**에 따른 것이며
  - 설정된 **만료 시간**전에는 타임아웃되지 않음
- 타임아웃 시맨틱이 **일반적인 직관**에서 벗어난 곳은
  - **만료 시간**이 지난 후, **타임아웃 이벤트가 실제로 발생할 때**
- 현재 타임아웃 처리는 **새 데이터 수신**에 바인드
  - 따라서 **잠시 침묵하고 처리할 새 트리거**를 생성하지 않는 스트림은, 타임아웃도 생성하지 않음
- 현재 타임아웃 시맨틱은 **이벤트 측면**에서 정의
- **타임아웃 이벤트**는 실제 타임아웃이 발생한 후
  - 타임아웃 이베트가 `얼마나 오래 발생했는지`에 보증 없이, 상태가 만료된 후 **트리거**
- 공식적으로 말하면 **타임아웃이 언제 발생할지에 대한 엄격한 상한은 없음**
- 주의
  - `사용 가능한 새 데이터가 없는 경우에도 타임아웃이 발생하도록 작업 진행 중`

## 13.5. 요약
- 구조적 스트리밍의 **임의 상태 기반 처리 API**
- 생선된 이벤트 및 지원되는 출력 모드와 관련하여
  - `maapGroupsWithState`와 `flatMapGroupsWithState`의 세부 사항과 차이점
- 타임아웃 설정
- 이 장에서 살펴본 API는
  - 일반 `SQL`과 유사한 구성보다 사용하기 더 복잡하나,
  - **스트리밍 사용 사례 개발**을 처리하기 위한 **임의의 상태 관리**를 구현할 수 있는 강력한 도구셋 제공