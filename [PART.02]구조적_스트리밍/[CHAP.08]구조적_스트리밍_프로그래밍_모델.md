# [CHAP.08] 구조적 스트리밍 프로그래밍 모델
- **구조적 스트리밍**은
  - `Spark SQL`의 `DataFrame`과
  - `Spark SQL`의 `Dataset` API 위에 놓인 기초를 기반으로 함
- 이러한 `API`를 확장하여
  - `Catalyst Query Optimizer` 엔진과 `Tungsten` 프로젝트에서 제공하는 **낮은 오버헤드 메모리 관리 코드 생성**
  - 기본 쿼리 최적화 뿐만 아니라 `Spark SQL`이 도입한 **고급 언어의 특성**을 계승
- 구조적 스트리밍은, `Spark SQL`에 대해 지원되는 **모든 언어에서 사용 가능**
- `Spark SQL`에 사용한 **중간 쿼리 표현** 덕분에
  - 프로그램의 성능은, 사용된 `Language Binding`에 상관없이 동일
- 구조적 스트리밍은 모든 `윈도우` 및 `집계 작업`에서의 **Event time**을 지원하고
  - **처리 시간**(processing time)이라는 **처리 엔진**에 들어가는 시간이 아닌
  - `이벤트가 생성된 시간을 사용하는 로직`을 쉽게 사용 가능
- spark는 `class batch`와 `stream` 기반의 데이터 처리 간의 개발 경험을 통합함\

## 8.1. 스파크 초기화
- `SparkSession`이 구조적 스트리밍을 사용하는 
  - `batch` 및 `streaming` application의 단일 entry-point
- 따라서, spark 작업을 생성하기 위한 entry-point는
  - spark batch API를 사용할때와 동일

#### CODE.8.1. 로컬 스파크 세션 생성
```scala
val spark = SparkSession
  .builder()
  .appName("StreamProcessing")
  .master("local[*]")
  .getOrCreate()
```

#### spark shell 사용하기
- `spark shell` 사용시, SparkSession은 `spark`에서 제공
- 따라서 **구조적 스트리밍**을 사용하기 위해, 추가적인 컨텍스트 생성 필요 x

## 8.2. 소스 : 스트리밍 데이터 수집
- **구조적 스트리밍**에서 `source`는 
  - 스트리밍 데이터 생산자의 데이터를 사용할 수 있는 추상화
- `source`는 직접 작성되지 않음
- `sparkSession`은 `API`를 노출하여
  - `format`이라는 스트리밍 소스를 지정
  - 이후 빌더 메서드인 `readStream`을 사용

#### CODE.8.2. 파일 스트리밍 소스
```scala
val fileStream = spark.readStream
  .format("json")
  .schema(schema)
  // json 형식을 준수하지 않거나, 제공된 스키마와 일치하지 않는 라인 제거
  .option("mode", "DROPMALFORMED") 
  // 지정된 경로 모니터리 
  .load("/tmp/datasrc")
```
- 이면에서는 `spark.readStream`에 대한 호출이 `DataStreamBuilder` 인스턴스를 생성
  - 빌더 메서드 호출을 통해 제공되는 **다양한 옵션 관리**
- `DataStreamBuilder`의 인스턴스에 대한 호출 `load(...)`는
  - 빌더에 제공된 **옵션의 유효성**을 확인하고
  - 모든 항목이 확인된다면 스트리밍 `DataFrame`을 반환
- 스트리밍 소스를 로드하는 것은 느림
- 스트리밍 `DataFrame` 인스턴스에 구현된 스트림의 표현
  - 특정 비즈니스 로직을 구현하기 위해, 적용할 일련의 변환을 표현하는데 사용
- 스트리밍 `DataFrame`을 만들면
  - 스트림이 구체화될 때까지 실제로 사용되나
  - 처리하는 데이터가 없음
- 나중에 확인할 `query`가 필요함

#### ReadStream & WriteStream
- spark API의 대칭성
- readStream
  - 스트림 소스를 선언하는 옵션
- writeStream
  - 출력 싱크 및 출력 모드 지정
- `DataFrame API`의 `read/write`와 대응
- `read/write` 배치 작업
- `readStream/writeStream` : 스트리밍 작업

### 8.2.1. 사용 가능한 소스
- `spark 2.4.0`부터는 다음과 같은 스트리밍 소스 지원
- `json, orc, parquet, csv, text, textFile`
  - 파일 기반 스트리밍 소스
  - 파일 시스템에서 경로를 모니터링하고, 그 안에 원자적으로 배치된 파일 사용
  - 찾은 파일은 지정된 `formatter`에 의해 파싱
    - e.g. `json`이 제공되는 경우, 제공된 **스키마 정보**를 사용하여 `spark json` 판독기가 파일을 처리
- `socket`
  - 소켓 연결을 통한 **텍스트 데이터**
  - `TCP`서버에 대한 클라이언트 연결 설정
- `kafka`
  - 카프카에서 데이터 검색, 카프카 소비자 생성
- `rate`
  - `rowsPerSecond` 옵션
  - 지정된 비율로 **행 스트림 생성**
  - **주로 테스트 소스**로 사용

## 8.3. 스트리밍 데이터 변환
- `load` 호출의 결과는 **스트리밍 DataFrame**
- 이 `스트리밍 DataFrame`을 만든 후에는
  - 데이터셋 또는 데이터프레임 API를 사용하여
  - 스트림의 데이터에 적용할 **논리**를 표현할 수 있음

#### 주의
- `DataFrame`은 `Dataset[Row]`의 별칭
- `DataSet API`는
  - 유형이 지정된 인터페이스를 제공하지만
- `DataFrame`의 사용은
  - 유형이 지정되지 않음
- 파티썬과 같은 **동적 언어**에서 **구조적인 API**를 사용하는 경우
  - `DataFrame API`를 유일하게 사용할 수 있음
- **형식화된 데이터셋**에서 작업을 사용할 때, **성능에 영향**
- `Dataframe API`에서 제공하는 `SQL`을
  - `query planner`가 이해하고 추가로 최적화할 수 있으나
- `Dataset API`에서 제공되는 `closure`는
  - `query planner`에 불투명하므로, `DF`에 비해 느리게 실행될 수 있음
  - **closure** : 외부 함수에 접근할 수 있는 **내부 함수**

#### CODE.8.3. 필터와 프로젝션 // 센서 네트워크 데이터 예시
```scala
val highTempSensors = sensorStream
        .select($"deviceId", $"timestamp", $"sensorType", $"value")
        .where($"sensorType" === "temperature" && $"value" > threshold)
```

#### CODE.8.4. 시간에 따른 센서 유형별 평균
- 데이터 집계 및 시간에 따라 **그룹**에 작업 적용 가능
- 이벤트 자체의 `timestamp` 정보를 사용하여
  - `1m`마다 슬라이드되는 `5m` 시간 윈도우 정의 가능
- `구조적 스트리밍 API`가 배치 분석을 위한 `Dataset API`와 실질적으로 동일
  - 스트림 처리에 특정한 몇 가지 추가 조항만 존재
- 코드
  ```scala
  val avgBySensorTypeOverTime = sensorStream
          .select($"timestamp", $"sensorType", $"value")
          .groupBy(window($"timestamp", "5 minutes", "1 minute"), $"sensorType")
          .agg(avg($"value"))
  ```

### 8.3.1. 데이터프레임 API에서의 스트리밍 API 제한
- 표준 데이터프레임 및 데이터셋 API에서 제공하는 일부 작업은
  - **스트리밍 컨텍스트**에서 의미가 없음
- `streaming.count`는 스트림에서 사용하기 적합하지 않음
- 스트림상 직접적으로 지원되지 않는 API
  - `count, show describe, limit take(n), distinct, foreach, sort, 누적된 여러 값 집계`
- `stream-stream`과 `stream-stream`의 join은 부분적으로 지원

#### 제약 사항의 이해
- `count`, `limit`과 같은 작업은 스트림에서 의미는 없음
  - 단, 일부 다른 스트림 작업은 **연산**이 어려움
- `distinct`가 그 예시
  - 임의의 스트림에서 **중복 필터링**하는 연산
  - 지금까지 본 모든 데이터를 가지고, // 무한한 메모리 필요
  - 새로운 레코드를 각 레코드와 비교해야 함 // `O(n^2)`
- 요소 `n`이 증가함에 따라, 엄두를 못낼 정도로 복잡도 증가

#### 집계 스트림에 대한 작업
- 지원되지 않는 일부 작업은
  - 집계함수를 **스트림**에 적용한 후에 정의 됨
- 스트림을 계산할 수는 없지만
  - `분당 수신한 메세지`를 `count`하거나
- 특정 유형의 기기수를 `count`할 수 있음

##### CODE.8.5. 시간이 지남에 따른 센서 유형 수
```scala
val avgBySensorTypeOverTime = sensorStream
        .select($"timestamp", $"sensorType")
        .groupBy(window($"timestamp", "1 minutes", "1 minutes"), $"sensorType")
        .count()
```
- 출력모드 `complete`를 사용한 쿼리로 제한
- 집계된 데이터에 대한 정렬(`sort`)를 정의할 수 있음

#### 스트림 중복 제거
- 스트림에서 `distinct`는 구현하기 어려움
- 단, 스트림의 요소가 **이미 표시**되었을 때 알려주는 키를 정의한다면
  - 이를 사용하여 복제본 제거 가능
- 예시
  ```scala
  stream.dropDuplicates("<key-column>") ...
  ```

#### 해결 방안
- 일부 작업은 **배치 모델**에서와 동일한 방식으로 지원되지는 않으나,
  - 동일한 기능을 달성하는 다른 방법이 존재

##### foreach
- `foreach`를 스트림에 직접 사용은 불가
- 동일한 기능을 제공하는 `foreach sink`가 존재
- `sink`는 스트림의 **출력 정의**에 지정

#### show
- `show`에서는 **쿼리를 즉시 구체화** 하므로
- 스트리밍 데이터셋에서는 사용 불가능하지만,
  - `console sink`를 사용하여 데이터를 화면에 출력 가능

## 8.4. 싱크: 결과 데이터 출력
- 스트림을 시작하기 전, 먼저 **출력 데이터**의 
  - **위치**와 **방법**을 정의해야 함
    - 스트리밍 싱크와 관련된 **위치**: 스트리밍 데이터의 수신측
    - 출력 모드를 언급하는 **방법**: 스트림에서 결과 레코드를 처리하는 방법
- `API` 관점에서,
  - 스트리밍 데이터 프레임, 데이터셋에서 `writeStream`을 호출하여 스트림 구체화
- **스트리밍 데이터셋**에서
  - `writeStream`을 호출하면 `DataStreamBuilder`가 생성됨
  - 이는 스트리밍 프로세스의 출력 동작을 구현하는 **메서드**를 제공하는 **빌더 인스턴스**

#### CODE.8.6. 파일 스트리밍 싱크
```scala
val query = stream.writeStream
      .format("json")
      .queryName("json-writer")
      .outputMode("append")
      .option("path", "/target/dir")
      .option("checkpointLocation", "/checkpoint/dir")
      .trigger(ProcessingTime("5 seconds"))
      .start()
```

### 8.4.1. format
- `format` 메서드는 `내장형 싱크`또는 `사용자 정의 싱크`의 정식 이름을 제공하여 출력 싱크 지정 가능
- `spark 2.4.0` 기준 `sink`
  - `console`
    - 표준 출력
    - `numRows` 옵션으로 구성할 수 있는 여러행 노출
  - `file`
    - fileSystem에 쓰는 `파일 기반` 및 `특정 형식`의 싱크 제공
    - `csv, hive, json, orc, parquet, avro, text`등의 형식 이름 제공 가능
  - `kafka`
    - `하나 이상의` 카프케 주제에 쓸 수 있는 `producer sink`
  - `memory`
    - 제공된 **쿼리 이름**을 **테이블 이름**으로 사용하여
      - 메모리 내 **테이블 생성**
    - 이 테이블은 스트림 결과로 **지속적인 업데이트**
  - `foreach`
    - `한 번에 한 요소씩` 스트림 콘텐츠에 접근
    - 프로그래밍 인터페이스 제공
  - `foreachBatch`
    - 구조적 스트리밍 실행의 각 **기본 마이크로 배치**에 해당하는
      - 완전한 `dataFrame`에 액세스 할 수 있는
      - 프로그래밍 방식 싱크 인터페이스

### 8.4.2. outputMode
- `outputMode`는 **스트리밍 쿼리의 출력**에 레코드가 추가되는 방식 지정
- `append, update, complete`

#### append
- default
- 출력 스트림에 **최종 레코드**만 추가
- 수신 스트림의 새 레코드가 해당 값을 수정할 수 없는 경우 **최종 레코드**로 간주
- `projection, filtering, mapping`을 적용한 **선형 변환**의 경우, 항상 이러한 현상 발생
- 각 결과 레코드가 **한 번만 출력**

#### update
- 마지막 **트리거** 이후의 `새 레코드` 및 `업데이트된 레코드`를 출력 스트림에 추가
- 새 레코드가 도착하면 **집계된 값**이 변경되는 **집계 컨텍스트**에만 의미가 있음
- `하나 이상의 수신 레코드`가 **단일 결과**를 변경하면
  - 트리거 간격 사이의 **모든 변경 사항**이 하나의 **출력 레코드**로 수집

#### complete
- 스트림의 완전한 내부 표현
- 집계와도 관련이 있음
- `집계되지 않은` 스트림의 경우,
  - 지금까지 본 `모든 레코드`를 기억해야 함 => **비현실적**
- 실제적인 관점에서의 `complete` 모드는
  - **국가별 방문자 수**와 같은, **낮은 카디널리티 기준**(국가수는 한정됨)
  - 이에 따라 값을 집계하는 경우에만 권장

#### append 시멘틱의 이해
- 스트리밍 쿼리에 **집계**가 포함된 경우, **마지막의 정의**는 중요하지 않음
- `집계 연산`에서 새 **수신 레코드**는
  - 사용된 **집계 기준**을 준수할 때, **기존 집계 값**을 변경할 수 있음
- **정의**에 따라 값이 **최종**임을 알 때까지
  - `append`를 사용하여 레코드를 출력할 수 없음
- 따라서 **집계 쿼리**와 조합하여
  - `append` 출력 모드를 사용하는 것은
  - **이벤트 시간**을 사용하여 **집계가 표현되는 쿼리**로 제한되며
  - **워터마크**를 정의
- 이 경우 **워터미크**가 만료되자마자, `append`가 이벤트를 출력하므로
  - 집계된 값을 **변경할 수 있는 새 레코드**는 없는 것으로 간주 됨
- 결과적으로 `append` 모드의 **출력 이벤트**는
  - **집계 시간 윈도우**와 **워터마크 오프셋**에 의해 지연

### 8.4.3. queryName
- 일부 싱크에서 사용하고, `spark console`의 **작업 설명**에 표시되는 **쿼리의 이름** 제공 가능

### 8.4.4. option
- 소스 구성과 유사하게, `key-value`쌍의 구성을 **스트림**에 제공 가능
- 각 싱크는 이 방법을 사용하여, 커스터마이징 할 수 있음
- 싱크를 구성하는데 필요한 만큼 `.option(...)`호출 추가 가능

### 8.4.5. options
- 설정하려는 **모든 `key-value`**구성 파라미터를 포함하는 `Map[Stirng, String]`을 사용
- `option`의 대안
- 싱크의 구성으로 전달될 설정을 **미리 알지 못하는**
  - 외부화된 구성 모델에 더 친숙

### 8.4.6. trigger
- 선택적 `trigger`옵션을 사용하면
  - **결과를 생성할 빈도**를 지정할 수 있음
- 기본적으로 **구조적 스트리밍**은
  - 입력을 처리하고, 가능한 한 **빨리 결과**를 생성
- 트리거가 지정되면 **트리거 간격마다 출력 생성**
- `org.apache.spark.sql.streaming.Trigger`는 지원되는 트리거를 다음과 같이 제공

#### `ProcessingTime(<interval>)`
- 쿼리 결과의 **빈도**를 나타낼 수 있는 **시간 간격**

#### `Once()`
- 스트리밍 잡을 **한 번 실행**할 수 있는 특정 트리거
- **테스트**시에 유용하며
- 정의된 **스트리밍 잡**을 **단일 샷 배치 작업**으로 적용 가능

#### `Continuous(<checkpoint-interval>)`
- **짧은 지연 시간 처리**를 위해 **실행 엔진**을
  - 실험용의 `continuous`로 전환
- `checkpoint-interval` 파라미터는
  - **데이터 복원력**에 대한 **비동기 체크포인트 빈도**를 나타냄
- `ProcessingTime` 트리거의 **배치 간격**(batch-interval)과 혼동하면 안됨

### 8.4.7. start()
- 스트리밍 연산을 **구체화**하려면 **스트리밍 프로세스**를 시작해야 함
- `start()`는
  - 전체 잡 설명을 **스트리밍 연산으로 구체화**하고
  - 내부 스케줄링 프로세스를 시작하여
  - `src`에서 데이터를 소비하고 처리하여 `sink`로 생성
- 각 **쿼리의 개별 수명 주기**를 관리하는 `Streaming Query` 객체를 반환
- 이와 동일한 `SparkSession`내에서
  - 서로 **독립적**으로 여러 쿼리를 **동시에 시작**하고 **중지**할 수 있음

## 8.5. 요약
- 각 스트리밍 프로그램은 **소스**와 **현재 사용 가능한 소스**를 정의함으로써 시작
- 스트리밍 데이터 변환에 대부분의 `DataSet/DataFrame API` 사용 가능
- `Batch API`의 몇몇 일반적인 작업은 `Streaming Mode`에서 의미가 없음
- `Sink`는 스트림 **출력**의 구성 가능한 정의
- `Stream`에서 **출력 모드**와 **집계 작업**과의 관계
- 모든 변환이 **지연**되고 시스템을 통해 데이터가 흐르도록 **스트림을 시작**해야 함
