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