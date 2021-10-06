# [CHAP.07] 구조적 스트리밍 소개
- **Spark SQL**의 **데이터셋 추상화**는 **유휴 데이터**를 분석하는 방법중 하나
- 본질적으로 **구조화 된 데이터**에 유용함
  - 정해진 스키마
- `Spark DataSet API`는
  - `SQL`과 유사한 **API의 표헌성**과 **스칼라 컬렉션** 및 
  - **RDD 프로그래밍 모델**을 연상시키는 `type-safe` 컬렉션 작업 결합
- 동시에 `python pandas` 및 `R dataframe`과 유사한 `Dataframe API`는
  - **함수형 패러다임**에서 주로 개발하던
  - 초기 핵심적인 **데이터 엔지니어**를 넘어, 스파크 사용자의 대상을 넓힘
- 이 높은 수준의 **추상화**는
  - 더 넓은 범위의 전문가가
  - 친숙한 API를 사용하여 빅데이터 분석 교육이 시작됨
    - 최신 데이터 엔지니어링 및 데이터 과학 실습 지원
- 데이터가 정착(settle down)될 때까지, 기다리지 않고
  - 원래 **스트림** 형태인 동안
  - 동일한 **데이터셋** 개념을 데이터에 적용할 수 있다면?
- 구조적 스트리밍 모델은 **이동 중에 데이터를 처리하기 위한** 데이터셋 SQL 지향 모델의 확장
  - 데이터가 `source stream`으로 부터 도착, 정해진 **스키마**가 있다고 가정
  - 이벤트 스트림은 **무한한 테이블**에 **추가된 행**으로 보여질 수 있음
  - 스트림에서 결과를 얻기 위해, 연산을 **해당 테이블에 대한 쿼리**로 표현
  - 동일한 쿼리를 **업데이트 테이블**에 지속적으로 적용하여
    - 처리된 이벤트의 **출력 스트림**을 생성
  - 결과 이벤트는 출력 `sink`에 제공
  - `sink`는 스토리지 시스템, 다른 스트리밍 백엔드 또는
    - 처리된 데이터를 사용할 준비가 된 app일 수 있음
- 이 모델에서 이론적으로 `unbounded table`은
  - 정의된 **제약 조건이 있는 실제 시스템**에서 구현
  - 모델 구현시, 잠재적으로 무한한 데이터 유입을 처리하기 위해
    - 특정 고려 사항 및 제한 사항이 필요
- 위 문제를 해결하기 위해 구조적 스트리밍은 아래와 같은 새로운 개념을, `DataSet/DataFrame API`에 도입
  - **이벤트 시간 지원**
  - **watermarking**
  - 과거 데이터가 얼마 오래 저장되었는지 결정하는 **다양한 출력 모드**
- 개념적으로 **구조적 스트리밍** 모델은
  - **일괄 처리**와 **스트리밍 처리**사이의 경계를 흐리게 하여
  - 빠르게 움직이는 데이터 상에서 **분석에 대한 추론 부담**을 상당히 제거

## 7.1. 구조적 스트리밍의 첫 걸음
- 간단한 `웹로그 분석` 사례를 예시로, 구조적 스트리밍 보기
- 아파치 스파크의 **기존 배치 분석**을 동일한 사례에 적용하기
- 두가지 주요 목표
  - 대부분 스트리밍 데이터 분석은 **정적 데이터 샘플**을 연구하는 것으로 시작
    - 데이터 파일 연구, 데이터가 어떻게 보이는지, 어떤 패턴을 보이는지, 직관적으로 파악
      - 그 데이터에서 의도한 지식을 추출하기 위해
      - **과정 정의**
    - 데이터 분석 잡을 정의하고, `테스트`한 이후에야
    - **데이터 분석 논리**를 이동중인 데이터에 적용할 수 있는 **스트리밍 프로세스**로 변환 가능
  - 현실적인 관점에서, 아파치 스파크가 `배치 및 스트리밍 분석`에 **균일한 API**를 사용하여
    - 배치 탐색에서 **스트리밍 어플리케이션**으로 전환하는 **여러 측면 단순화**

## 7.2. 배치 분석
- `아카이브 로그 파일로 작업`
  - 모든 데이터에 한 번에 액세스 가능
- 예시 링크
  - https://github.com/stream-processing-with-spark
- 간단히 배치 분석 진행
  - `json` 인코딩 된 로그 파일 로드
  - 유형이 지정되었기 때문에, 관련한 스키마 `case class` 지정
    ```scala
    import java.sql.Timestamp
    case class WebLog(host: String,
                      timestamp: Timestamp,
                      request: String,
                      http_reply : Int,
                      bytes: Long
                      )
    ```
    - 스파크 내부적으로 지원되며, 다른 옵션에 필요할 수 있는 **추가적인 캐스트가 필요하지 않기 떄문**에
      - 타임스탬프 유형으로 `java.sql.Timestamp`를 사용
- 스키마 정의를 가지고, json을 지정한 데이터 구조로 변환
  ```scala
  val logsDirectory = ???
  val rawLogs = sparkSession.read.json(logsDirectory)
  val preparedLogs = rawLogs.withColumn("http_reply", $"http_reply".cast(IntegerType))
  val weblogs = preparedLogs.as[WebLog]
  ```
- 각 질문별 연산
  ```scala
  // 레코드 수 계산
  val recordCount = weblogs.count

  // 하루에 가장 인기 있는 URL
  // Timestamp -> day 기준
  // 새로운 dayOfMonth 열과 요청 URL을 기준으로 그룹과, 이 집계를 카운트
  // 상위 URL을 먼저 얻기 위해, 내림차순을 사용하여 정렬
  val topDailyURLs = weblogs.withColumn("dayOfMonth", dayofmonth($"timestamp"))
                            .select($"request", $"dayofMonth")
                            .groupBy($"dayOfMonth", $"request")
                            .agg(count($"request").alias("count"))
                            .orderBy(desc("count"))
  topDailyURLs.show()

  // 가장 많은 트래픽을 발생시키는 콘텐츠 페이지
  // html을 필터링 한 다음, 상위 집계 활용
  // request 필드는 [HTTP VERB] URL [HTTP_VESERION]
  // URL을 추출하여 .html, .htm 또는 확장자 없음(디렉터리)로 끝나는 URL만 보존
  val urlExtractor = """^GET (.+) HTTP/\d. \d""".r
  val allowedExtensions = Set(".html", ".htm", "")
  val contentPageLogs = weblogs.filter(log => 
    log.request match {
      case urlExtractor(url) =>
        val ext = url.takeRight(5).dropWhile(c => c != ".")
        allowedExtensions.contains(ext)
      case _ => false
    }
  )

  // .html, .htm 및 디렉터리만 포함하는 이 새로운 데이터셋을 사용하여, 이전과 동일한 top-k 함수 적용
  val topContentPages = contentPageLogs
        .withColumn("dayOfMonth", dayofmonth($"timestamp"))
        .select($"request", $"dayOfMonth")
        .groupBy($"dayOfMonth", $"request")
        .agg(count($"request").alias("count"))
        .orderBy(desc("count"))
  
  topContentPages.show()
  ```

## 7.3. 스트리밍 분석
- 스트리밍 분석의 당위성을 부여하는 요인
  - 다양한 수준에서 의사결정을 내리는데, 도움이 될 수 있는 시기 적절한 정보를 보유하려는 수요
- 데이터 예시로, TCP 서버를 사용하여, 로그를 실시간으로 전송하는 **웹 시스템**을 시뮬레이션
- 예시 코드 : `weblog_TCP_server`, `streaming_weblogs` notebook

### 7.3.1. 스트림에 연결하기
- `TCP` 소켓을 통하여 서버에 연결하기 위해 `TextSocketSource` 구현을 사용
- 소켓 연결은, 서버의 호스트와 연결을 수신하는 **포트**에 의해 정의
  ```scala
  val stream = sparkSession.readStream
                .format("socket")
                .option("host", host)
                .option("port", port)
                .load()
  ```
- 스트림 생성이, 배치 처리 사례에서의 정적 데이터 소스 선언과 매우 유사
- `readStream`을 사용하여 스트리밍 소스에 필요한 파라미터 전달
- 구조적 스트리밍의 세부 사항을 살펴볼 때,
  - API는 기본적으로 **정적 데이터**에 동일한 **데이터프레임** 및 **데이터셋 API**

### 7.3.2. 스트림에서 데이터 준비하기
- 소켓 소스는 스트림에서 수신된 데이터를 포함하는
  - 하나의 열(값)으로 스트리밍 데이터프레임을 생성
- 소켓 소스의 경우, 일반 텍스트
- 원시 데이터를 WebLog로 변환하려면, 스키마가 먼저 필요
- 스키마는 텍스트를 `JSON` 객체로 파싱하는데 필요한 정보를 제공
- 이는 **구조적 스트리밍**에 대해 이야기할 때의 **구조(scheme)**이다
- 데이터 스키마를 정의한 후, 다음 단계에 따라 데이터셋 생성
  ```scala
  case class webLog(...)

  // case class에서 스키마 확보
  val webLogSchema Encoders.product[WebLog].schema
  // spark sql에 내장된 json 지원을 사용하여, 텍스트를 json으로 변환
  val jsonStream = stream.select(from_json($"value", webLogSchema) as "record")
  // 데이터셋 API를 사용하여, JSON 레코드를 WebLog 오브젝트로 변환
  val webLogStream: Dataset[WebLog] = jsonStream.select("record.*").as[WebLog]
  ```
  - 위 결과로 `WebLog` 레코드의 **스트리밍 데이터셋**을 얻게됨

### 7.3.3. 스트리밍 테이터셋에 대한 작업
- `webLogStream`은 배치 분석과 동일하게 `Dataset[WebLog]` 유형
- 배치 버전과의 차이는, `webLogStream`이 **스트리밍 데이터셋** 이라는 것
- 객체를 쿼리하여 확인 가능
  ```scala
  webLogStream.isstreaming // true
  ```
- `count`수행시 `AnalysisException` 발생
  ```scala
  val count = webLogStream.count()
  ```
- 이는 정적 데이터셋 또는 데이터프레임에서 사용한 직접 쿼리에
  - 이제는 두 가지 수준의 **상호작용**이 필요하다는 것을 의미
- **스트림의 변환**을 선언한 다음 **스트림 프로세스**를 시작해야 하

### 7.3.4. 쿼리 작성하기
- 관심 있는 기간을 정의하기 위해 **타임스탬프**에 대한 **창**을 만듦
- 구조적 스트리밍은
  - 데이터가 처리되는 시간이 아닌 **데이터가 생성된 타임스탬프**(Event Time)에서 **해당 시간 간격 정의 가능**
- 윈도우 정의는 `5min`의 이벤트 데이터로 구성
- 타임라인이 시뮬레이션된 경우
  - `5min`은 시계 시간보다 훨씬 빠르거나, 느릴 수 있음
- 구조적 스트리밍이 이벤트 다임스탬프 정보를 사용하여
  - 이벤트 타임라인을 추적하는 방법을 분명 이해할 수 있음
- `URL`을 추출하고, `.html`, `.htm` 또는 디렉터리와 같은 콘텐츠 페이지를 선택해야 함
  ```scala
  // weblog, request에서 접근한 URL을 표현하는 정규 표현식
  val urlExtractor = """^GET (.+) HTTP/\d. \d""".r
  val allowedExtensions = Set(".html", ".htm", "")

  val contentPageLogs: String => Boolean = url => {
    val ext = url.takeRight(5).dropWhile(c => c != '.')
    allowedExtensions.contains(ext)
  }

  val urlWebLogStream = webLogStream.flatMap { weblog => 
    weblog.request match {
      case urlExtractor(url) if (contentPageLogs(url)) =>
        Some(weblog.copy(request = url))
      case _ => None
    }
  }

  // 인기 URL을 계산하기 위한, 윈도우 쿼리 정의
  val rankingURLStream = urlWebLogStream
          .groupBy($"request", window($"timestamp", "5 minutes", "1 minute"))
          .count()
  ```

### 7.3.5. 스트림 처리 시작하기
- 위에서 작업한 내용은 **스트림이 수행할 프로세스**를 정의하기 위한 것
- 구조적 스트리밍 job을 시작하기 위해 `sink` 및 `outputMode`를 정의해야 함
- 구조적 스트리밍에서 도입된 두가지 새로운 개념
  - **sink**
    - 결과 데이터를 구체화할 위치
    - e.g. 파일시스템의 파일, 인메모리 테이블, 카프카와 같은 다른 스트리밍 시스템
  - **outputMode**
    - 결과 전달 방법
    - 매번 모든 데이터를 볼 것인가?
    - 오직 업데이트가 발생할 때만 볼 것인가?
    - 아니면 단지 새 레코드만 볼것인가?
- 위 옵션은 `writeStream` 작업에 제공
- 스트림 소비를 시작하고, 선언된 계산을 구체화 하고,
  - 출력 싱크에 생성하는 스트리밍 쿼리를 생성
- `memory` 싱크와, `complete` 출력 모드 사용
  - 새 레코드가 추가될 때마다, 완전히 업데이트 된 테이블을 갖도록 함
    ```scala
    val query = rankingURLStream.writeStream
        .queryName("urlranks")
        .outputMode("complete")
        .format("memory")
        .start()
    ```
- `memory` 싱크는, `queryName` 옵션에 주어진 것과, 동일한 이름의 **임시 테이블**에 데이터를 출력
- `spark SQL`에 등록된 테이블을 쿼리하여 이를 확인할 수 있음
  ```scala
  spark.sql("show tables").show();
  ```
- 표현식에서, 쿼리는 `StreamingQuery` 타입이며
  - 쿼리 수명 주기를 제어하는 핸들러

### 7.3.6. 데이터 탐색
- 생산자 측에서 **로그 타임라인**을 **가속화**하고 있다고 가정하면,
  - 몇 초 후에 다음 명령을 실행하여
  - 첫 번째 윈도우의 결과를 볼 수 있음
- `처리 시간`(몇 초)에 어떻게 **이벤트 시간**(수백분의 로그)에서 분리되는지 확인
  ```scala
  urlRanks.select($"request", $"window", $"count").orderBy(desc("count"))
  ```

## 7.4. 요약
- 스트리밍 어플리케이션의 개발 과정
- 구조화된 배치와 스트리밍 API가 얼마나 가까운지 알 수 있지만
  - 일반적인 배치 작업이, 스트리밍 컨텍스트에 적용되는 것도 관찰