# [CHAP.09] 구조적 스트리밍 작동
- 사물 인터넷에서 영감을 받은 스트리밍 프로그래밍 제작
  - `Structured-Streaming-in-action` notebook
  - `https://github.com/stream-processing-with-spark
- 스트리밍 소스로서, `Apache kafka의 센서 판독 스트림`을 소비하기
- `parquet` 형태로 저장

## 9.1. 스트리밍 소스 소비하기
- 코드
  ```scala
  val rawData = sparkSession.readStream
          .format("kafka")
          .option("kafka.bootstrap.servers", kafkaBootstrapServers)
          .option("subscribe", topic)
          .option("startingOffsets", "earliest")
          .load()
  ```
- 기존 `SparkSession`을 사용
- 정적 `Dataset` 생성과 거의 동일
- `sparkSession.readStream`은 `floud API`를 사용하여 **스트리밍 소스**를 구성하는데
  - 필요한 정보를 수집하기 위해
  - 빌더 패턴을 구현하는 클래스인 `DataStreamReader`를 리턴
- 이 `API`에서 **소스 공급자**를 지정할 수 있는 `format` 옵션을 찾음
- 옵션 설명
  - `kafka.bootstrap.servers`
    - 쉼표로 구분된 `host:port` 주소 목록으로 접속할, 일련의 **부트스트랩 서버**의 집합
  - `subscribe`
    - 구독할 `topic`
  - `startingOffsets`
    - 이 app이 시작할 때 적용되는 오프셋 재설정 정책
- `load()`는
  - `DataStreamReader`를 평가하고,
  - 결과값에서 볼 수 있는 것처럼 `DataFrame`을 결과로 생성
- `DataFrame`은 알려진 스키마가 있는 `Dataset[Row]`의 별칭
  - 일반 데이터셋처럼 **스트리밍 데이터셋** 사용 가능
- `show()`나 `count()`와 같은 모든 작업이
  - 스트리밍 컨텍스트에서 의미가 없기 때문에
  - 일부 예외가 적용되긴 하나,
    - 구조적 스트리밍과 함께 본격적인 `Dataset API`를 사용할 수 있음
- 스트리밍 종류인지 여부 파악
  ```scala
  rawData.isStreaming
  // Boolean = true
  ```
- 연결된 스키마 탐색 방법
  ```scala
  rawData.printSchema()
  ```
- 일반적으로 **구조적 스트리밍**에는
  - 소비된 스트리밍에 대한 스키마의 명시적 선언이 필요
- `kafka`의 특정 케이스에서 결과 `Dataset`의 스키마는 고정되어 있으며
  - 스트림의 내용과 무관함
- 카프카 소스에 고유한 필드 집합인
  - `key, value, topic, partition, offset, timestamp` 및 `timestampType`으로 구성
- 대부분의 경우, 스트림의 `payload`가 상주하는 `value` 필드를 주로 활용

## 9.2. 애플리케이션 로직
- 업무의 목적
  - 수신되는 `IoT` 센서 데이터를
  - 구성이 알려진 모든 센서가 포함된 **참조 파일**과 연관시키는 것
- 보고된 데이터를 해석할 수 있는 **특정 센서 파라미터**로
  - 각 수신 레코드를 보강할 수 있음
  - 이후 올바르게 처리된 모든 레코드를 `parquet` 파일에 저장함
- 알려지지 않는 센서에서 나온 데이터는 나중에 분석하기 위해 별도의 파일로 저장
- 구조적 스트리밍을 사용하여 Dataset 작업의 관점에서 Job 구현 가능
  ```scala
  val iotData = rawData.select($"value").as[String].flatMap{record => 
    val fields = record.split(",")
    Try {
      SensorData(fields(0).toInt, fields(1).toLong, fields(2).toDouble)
    }.toOption
  }

  val sensorRef = sparkSession.read.parquet(s"$workDir/$referenceFile")
  sensorRef.cache()

  val sensorWithInfo = sensorRef.join("iotData", Seq("sensorId"), "inner")

  val knownSensors = sensorWithInfo
    .withColumn("dnvalue", $"value"*($"maxRange"-$"minRange")+$"minRange")
    .drop("value", "maxRange", "minRange")
  ```
- 첫 단계에서는 `CSV` 형식의 레코드를 다시 `SensorData` 항목으로 변환
- `value` 필드를 `String`으로 추출하여 얻은 유형이
  - 지정된 `Dataset[String]`에 스칼라 기능 연산을 적용
- 이후 정적 데이터셋 `inner` 조인에 **스트리밍 데이터셋**을 사용하여
  - `sensorId`를 키로 하여 센서 데이터를 해당 참조와 상관
- app을 완성하기 위해, 기존 데이터의 `min-max` 범위를 사용하여
  - 센서 판독값의 실젯값을 계산

## 9.3. 스트리밍 싱크에 쓰기
- 데이터를 `parquet` 형식 파일에 쓰기
- 구조적 스트리밍에서는 `write`작업이 중요
- **선언된 변환**이 **스트림**에서 완료되었음을 표시하고,
  - `write`모드를 정의하며 `start()`를 호출하면, **연속 쿼리 처리**가 시작됨
- 구조적 스트리밍에서 **모든 작업**은
  - **스트리밍 데이터**로 수행하려는 작업에 대한 **늦은 선언**
- `start()`를 호출할때만
  - 스트림의 **실제 소비**가 시작되고,
  - 데이터에 대한 **쿼리 작업**이 **실제 결과**로 **구체화**
- 예시 코드
  ```scala
  val knownSensorQuery = knownSensors.writeStream
      .outputMode("append")
      .format("parquet")
      .option("path", targetPath)
      .option("checkpointLocation", "/tmp/checkpoint")
      .start()
  ```
- 코드 설명
  - `writeStream`
    - `fluent`(유연한) 인터페이스를 사용하여
    - 원하는 **쓰기 작업**에 대한 옵션을 구성할 수 있는 **빌더 객체** 생성
  - `format`을 사용하여, 결과 다운스트림을 구체화할 싱크를 지정
    - 이 경우, 내장 `FileStreamSink`와 `parquet` 형식을 사용
  - `mode`
    - 구조적 스트리밍에서 새로운 개념
    - 이론적으로, 스트림에서 볼 수 있는 모든 데이터에 접근할 수 있음을 감안할 때
    - 해당 데이터의 다른 보기를 생성할 수 있음
  - `append`
    - 스트리밍 계산의 영향을 받는 새 레코드가 출력으로 생성됨
- `start`호출의 결과는 `StreamingQuery` 인스턴스
- 이 객체는 쿼리의 **실행을 제어**하고 **실행 중인 스트리밍 쿼리의 상태**에 대한 정보를 요청하는 방법 제공
- `knownSensorsQuery.recentProgress`를 호출한 결과로
  - `StreamingQueryProgress`를 볼 수 있음
- `numInputRows`에 `0`이 아닌 값이 표시되면
  - `job`이 data를 소비하고 있음을 확신할 수 있음