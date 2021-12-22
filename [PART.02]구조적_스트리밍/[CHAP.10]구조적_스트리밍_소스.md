# [CHAP.10] 구조적 스트리밍 소스

## 10.1. 소스의 이해
- 구조적 스트리밍에서 **소스**
  - 스트리밍 데이터 공급자를 나타내는 **추상화**
- 스트리밍 데이터는
  - 시간이 지남에 따라 **연속적으로 발생**하는 **이벤트 흐름**
  - 단조롭게 증가하는 **카운터**로 인덱스된 **시퀀스**로 볼 수 있다는 것이
    - **소스 인터페이스**의 기본 개념
- **오프셋**은
  - 외부 소스에서 데이터를 요청하고
  - 이미 소된 데이터를 나타내는데 사용
- 구조적 스트리밍은
  - **외부 시스템**에서 현재 **오프셋**을 요청하고
  - 이를 마지막으로 처리한 **오프셋**과 비교하여 **처리할 데이터가 있는 시기**를 파악
- 처리할 데이터는 `start`, `end`사이에 **배치**를 가져와 요청
- 소스는 **주어진 오프셋**을 커밋하여 데이터를 처리했다는 정보를 받음
- 그 **소스 계약**은
  - **커밋된 오프셋**보다 작거나 같은오프셋을 가진 모든 데이터가, 처리되었으며
  - 후속 요청이 `커밋된 오프셋보다 큰 오프셋`만 규정하도록 보장
- 위 보장이 제공되면, 소스는 `처리된 데이터를 삭제`하여 **시스템 리소스**를 확보
- 오프셋 기반 처리 순서
  ```scala
  t1. getOffset = 21
  t2. getBatch(17, 21) = DataFrame(...)
  t3. commit(21)
  ```
  - t1: `getOffset`을 호출하여 `source`의 현재 오프셋을 얻음
  - t2: `getBatch(sttart, end)`를 호출하여
    - 알려진 오프셋까지의 배치를 얻음
    - 그 동안 새로운 데이터가 도착했을 수 있음
  - t3: 오프셋을 `commit`하고 `source`는 해당 레코드를 제거
- 위 프로세스는 **지속적으로 반복**하여 스트리밍 데이터 확보
- **최종 오류를 복구하기위해**
  - 오프셋은 종종 외부 저장소에 `checkpoint`됨
- 오프셋 기반 상호작용 외에도, 소스는 **두가지 요구 사항**을 충족해야 함
  - 소스를 **동일한 순서**로 재생해야 함
  - 소스는 **스키마**를 제공해야 함

### 10.1.1. 신뢰할 수 있는 소스는 지속 가능해야 함
- 구조적 스트리밍에서 **재생 가능성**(replayability)는
  - 이미 요청되었지만, 아직 커밋되지 않은 스트림의 일부를 요청
- 댜시 받고자 하는 오프셋 범위로 `getBatch`를 호출하여 이루어짐
- 소스는 `구조적 스트리밍 프로세스`가 완전히 **실패**한 후에도
  - **커밋되지 않은 오프셋 범위를 생성**할 수 있을 때, 신뢰할 수 있는 것으로 간주
- 복구 프로세스에서는
  - 마지막으로 알려진 **체크포인트**에서, `offset`이 복원되고,
  - 소스에서 다시 요청됨
- 스트리밍 프로세스 외부에 데이터를 안전하게 저장하려면
  - 소스 구현을 지원하는 **실제 스트리밍 시스템**이 필요
- 소스에서 **재생성**을 요구함으로써, 구조적 스트리밍은 소스에 **복구 책임을 위임**
- 이는 신뢰할 수 있는 소스만 **구조적 스트리밍**과 함께 작동하여
  - **강력한 end-to-end 전달 보장**을 생성함을 의미

### 10.1.2. 소스는 스키마를 제공해야 함
- **구조화된 API**의 특징
  - 서로 다른 수준에서 데이터를 처리하기 위해 **스키마**에 의존
- 불투명한 문자열 또는 바이트 배열 `blob`을 처리하는 것과 달리
  - **스키마 정보**는 **필드**와 **유형**의 관점에서 데이터가 어떻게 생성되는지를 알게 해줌
- 스키마 정보를 사용하여 **쿼리 계획**에서
  - 데이터, 스토리지 및 접근에 대한 `내부 바이너리 표현`에 이르기까지
  - 스택의 여러 수준에서 최적화 추진 가능
- 소스는 생성하는 데이터를 설명하는 **스키마 정보**를 제공해야 함
- 일부 소스 구현에서는 이 스키마를 구성하고, 이 구성정보를 활용하여
  - 수신 데이터를 자동으로 파싱, 유효한 레코드 반환 가능
- `JSON, CSV`파일과 같은 많은 **파일 기반 스트리밍 소스**가 이 모델을 따르며,
  - 이 모델에서는 사용자가 올바른 **파싱**을 위해, 파일 형식에서 사용되는 **스키마 제공**필요
- 일부 다른 소스는 모든 레코드의 **메타데이터** 정보를 표시하고
  - `payload parsing`을 app에 남겨두는 **고정된 내부 스키마**를 사용
- `스키마 중심의 스트리밍 어플리케이션`을 만드는 것은
  - 데이터가 시스템을 통과하는 방식을 `전체적으로 이해`하고
  - 다중 프로세스 스트리밍 파이프라인의 여러 단계를 공식화하기 때문에 바람직

#### 스키마 정의하기
- 스키마 정의를 생성하기 위해 `Spark SQL API`를 재사용
- 프로그래밍 방식으로 `case class`정의에서 유추하거나
  - 기존 데이터셋에 로드된 스트림의 내용을 정의하는 내용을 참고하는 등의 방법

##### 프로그래밍 방식
- `StructType`, `StructField` 클래스를 이용하여, 스키마 표현 작성
- `id, type, location coordinate`를 가진 궤도 차량을 나타내기 위해, 다음과 같이 정의 가능
- 코드
  ```scala
  val schema = StructType(
    List(
      StructField("id", StringType, true),
      StructField("type", StringType, true),
      StructField("location", StructType(List(
        StructField("latitude", DoubleType, false),
        StrcutField("longitude", DoubleType, false)
      )), false)
    )
  )
  ```
  - `StructField`는 중첩된 `StructType`을 포함할 수 있음
    - 임의의 깊이와 복잡도의 스키마 생성 가능

##### 추론을 이용한 방식
- scala에서는 스키마를 임의의 `case class` 조합을 사용하여 표현 가능
- 단일 `case class`나 `case class` 계층 구조가 제공되면
  - `case class`에 대한 `Encoder`를 작성하고
  - 해당 `encoder` instance에서 **스키마를 가져와** 스키마 표시 계산 가능
- 코드
  ```scala
  case class Coordinates(latitude: Double, longitude: Double)
  case class Vehicle(id: String, `type`:String, location: Coordinates)
  // Encoder로부터 Encoder와 schema 가져오기
  val schema = Encoders.product[Vehicle].schema
  ```

##### 데이터셋에서 추출
- 실용적인 방법: 샘플 데이터 파일을 `parquet`와 같은 **스키마 인식 형식**으로 유지하기
- 스키마 정의를 얻기 위해 **샘플 데이터셋**을 로드하고
- 로드된 `DataFrame`에서 스키마 정의를 가져옴
- 코드
  ```scala
  val sample = spark.read.parquet(...)
  val schema = sample.schema
  ```

#### 스키마 정리
- 스키마를 정의하는 **프로그래밍 방식**은 강력하지만
  - 노력이 필요하고, 유지관리가 복잡하여 오류에 이르는 경우가 있음
- `prototype`단계에서는 데이터 셋을 로드하는 것이 실용적일 수 있으나,
  - 샘플 데이터셋을 최신 상태로 유지해야할 경우, 실수로 복잡해질 수 있음
- 사례마다는 다를 수 있으나, `scala`를 사용할 때는
  - 가능하면 **추론을 이용한 방식**을 사용하는 것이 좋음

## 10.2. 사용 가능한 소스
- 파일
  - 파일로 저장된 데이터 수집 가능
  - 대부분의 경우 데이터는 스트리밍 모드에서 추가로 처리되는 레코드로 변환
  - `JSON, CSV, PARQUET, ORC` 및 일반 텍스트 형식 지원
- 카프카
  - 스트리밍 데이터 사용 가능
- 소켓
  - TCP 서버에 연결하고, **텍스트 기반의 데이터 스트림**을 사용할 수 있는 `TCP Socket Client`
  - 스트림은 `UTF8` 캐릭터셋으로 인코딩 되어야 함
- 레이트
  - 내부적으로 발생한 레코드 `(timestamp, value)`의 스트림을, 구성 가능한 **생산 속도**로 생성
  - 일반적으로 `학습 및 테스트` 목적으로 수행
- 신뢰 여부
  - 구조화된 스트리밍 프로세스가 실패하더라도, **오프셋에서 재생 기능을 제공할 때**
  - 신뢰할 수 있는(reliable)
    - 파일, 카프카
  - 신뢰할 수 없는(unreliable)
    - 소켓, 레이트
- `신뢰할 수 없는` 소스는, 데이터 손실이 용인될 수 있는 경우에만 **운영에서 사용 가능**
- 이 책에서는 `custom source`를 개발하기 위한 `public API`는 없으나
  - 앞으로 지원이 가능할 수 있음
  

## 10.3. 파일 소스
- 파일시스템의 모니터링된 **디렉터리**에서 파일을 읽음
- 파일 기반 핸드오버는
  - **배치 기반 프로세스**를 스트리밍 시스템과 연결하기 위해 일반적으로 사용하는 방법
  - 배치 프로세스는 **파일 형식**으로 출력을 생성
    - 파일 소스에 적합한 구현이, 이러한 파일을 선택하고 스트리밍 모드에서 추가 처리를 위해
    - 해당 콘텐츠를 레코드의 스트림으로 변환할 수 있는 **공통 디렉터리**에 출력을 떨굼

### 10.3.1. 파일 형식 지정하기
- `readStream` 빌더에서 `.format(<format_name>)` 메서드와 함께 제공되는 **지정된 형식**을 사용하거나
  - `DataStreamReader`에서 사용할 형식을 나타내는 **전용 메서드**를 사용하여 읽음
  - `readStream.parquet('/path/to/dir/)`과 같은 형식
- 지원되는 각 형식에 해당하는 **전용 메서드**를 사용하는 경우
  - 메서드 호출은 **빌더의 마지막 호출**로 수행해야 함

#### CODE.10.1. FileStream 구성하기
- 아래 3개의 코드는 모두 동일한 동작 수행
```scala
// 형식, 로드 경로 사용
val fileStream = spark.readStream
      .format("parquet")
      .schema(schema)
      .load("hdfs://data/exchange")

// 형식, 경로 옵션 사용
val fileStream = spark.readStream
      .format("parquet")
      .option("path", "hdfs://data/exchange")
      .schema(schema)
      .load()

// 전용 메서드 사용
val fileStream = spark.readStream
      .schema(schema)
      .parquet("hdfs://data/exchange")
```

#### spark 2.3.0에서 지원하는 파일 기반 형식
- 정적 데이터프레임, 데이터셋, `SQL API`에서 지원하는 것과 동일
  - CSV, JSON, PARQUET, ORC, TEXT, TEXTFILE

### 10.3.2. 공통 옵션
- 특정 형식과 관계없이
  - 파일 소스의 **일반적인 기능**은
  - 특정 `URL`로 식별되는 **공유 파일 시스템**에서 **디렉터리**를 모니터 하는 것
- 모든 파일 형식은 **파일 유입을 제어**하고
  - 파일의 **에이징 기준을 정의**하는 공통적인 일련의 옵션 정의

#### maxFilesPerTrigger(Default: X)
- 각 쿼리 **트리거**에서 소비될 **파일 수**를 의미
- 시스템에서 데이터 유입을 제어하는데 도움이 됨

#### latestFirst(Default: false)
- `true`, 최신 파일이 가장 먼저 처리
- 최신 데이터가 **이전 데이터**보다 우선순위가 높을 때 사용

#### maxFileAge(Default: 7days)
- 디렉터리에 있는 파일에 대한 **임계값**을 지정
- **임계값**보다 오래된 파일은 처리할 수 없으며, 효과적으로 무시됨
- 이 값은 `시스템 시계`가 아니라
  - 디렉터리의 **가장 최근 파일**과 관련된 것
- 예시
  - `maxFileAge=2days`
  - 가장 최근 파일이 **어제**인 경우
    - 파일이 오래되었다고 판별하는 임계값은 **`3days`보다 오래된 것을 의미**
- 이 특성은 **이벤트 시간**의 **워터마크**와 유사

#### fileNameOnly(Default: false)
- `true`, 두 파일의 이름이 같은 경우 동일한 것으로 간주
  - 그렇지 않을 경우 `전체 경로를 고려`

#### 유의 사항
- `latestFirst=true, maxFilesPerTrigger`가 구성될 경우
- 시스템이 최근에 발견한 파일에 **우선순위**를 부여하므로
  - 처리하기에 유효한 파일이 임곗값보다 오래된 조건이 있을 수 있기 때문에
  - `maxfileAge`는 무시됨
- 위 경우에 **에이징 정책**을 사용할 수 없음

### 10.3.3. 일반적인 텍스트 파싱 옵션(CSV, JSON)
- 구성 가능한 **파서**를 사용하여,
  - 각 파일의 텍스트 데이터를 **구조화된 데이터**로 변환
- `upstream process`가
  - 예상되는 형식을 충족하지 않는 레코드를 작성하는 것도 가능
  - 레코드는 **손상된 것**으로 간주
- 잘못된 데이터가 수신될 때, 스트리밍 프로세스가 `실패하지 않아야 함`
- 비즈니스 요구사항에 따라
  - 잘못된 레코드를 삭제하거나
  - 손상된 것으로 간주되는 데이터를, 별도의 오류 처리 흐름으로 라우팅 가능

#### 파싱 오류 처리
- 파서 동작을 구성하여 `손상된 것`으로 간주하는 레코드 처리 가능

##### mode(default: `PERMISSIVE`)
- parsing 중에 `손상된 레코드`가 처리되는 방식 제어
- `PERMISSIVE`
  - 손상된 레코드의 값이, 스키마에 있어야 하는 `columnNameOfCorruptRecord` 옵션으로 구성된 **특수 필드**에 삽입
  - 다른 모든 필드는 `null`로 설정
  - 필드가 존재하지 않으면, 레코드는 **삭제**됨
    - `DROPMALFORED`와 동일 동작
- `DROPMALFORMD`
  - 손상된 레코드가 **삭제**됨
- `FAILFAST`
  - 손상된 레코드가 발견되면 **예외** 발생
  - 에러의 전파로 **스트리밍 프로세스가 중단**
  - 권장하지 않음

##### columnNameOfCorruptRecord(default: "_corrupt_record")
- 조작된 레코드의 문자열 값을 포함하는 **특수 필드의 구성 허용**
- `spark.sql.columnNameOfCorruptRecord`를 설정하여 필드 구성 가능
- `spark.sql.columnNameOfCorruptRecord`와 이 옵션이 모두 설정된 경우
  - **이 옵션이 우선됨**

#### 스키마 추론
##### inferSchema(default: `false`)
- 스키마 추론이 지원되지 않음
- 스미카가 제공되는 것이 필수

#### 날짜와 시간 형식
##### dateFormat(default: `yyyy-MM-dd`)
- `date`를 파싱하는데 사용되는 패턴 구성
- `java.text.SimpleDateFormat`을 따름

##### timestampFormat(default: `yyyy-MM-dd'T'HH:mm:ss.SSSXXX`)
- `timestamp` 필드를 파싱하는데 사용되는 패턴 구성
- `java.text.SimpleDateFormat`에 정의된 형식을 따름

### 10.3.4. JSON 파일 소스 형식
- `JSON`으로 인코딩된 텍스트 파일 사용 가능
  - 파일의 **각 줄**은 유효한 `JSON`객체를 의미
- 제공된 스키마를 제공하여 `JSON`을 파싱
- 스키마를 따르지 않는 레코드는 **유효하지 않은 것으로 간주**
- 유효하지 않은 레코드 처리시, 제어할 수 있는 몇가지 옵션 존재

#### JSON 파싱 옵션
- 기본적으로 `JSON file source`는
  - `JSON Lines specification`을 따를 것으로 기대
- 파일에 있는 각각의 `row`는
  - 지정된 스키마를 준수하는 유효한 `JSON document`로 기대
- 각 줄은 `\n`으로 구분되어야 하며,
  - 후행 공백이 무시되므로, `\r\n`도 지원
- 표준을 완전히 준수하지 않는 데이터를 처리하기 위해
  - `JSON Parser`의 **허용 오차**를 조정할 수 있음
- 손상된 것으로 간주되는 레코드를 처리하기 위해
  - 동작을 변경할 수 있음

##### allowComments(default: `false`)
- `enabled`일 경우,
  - 파일에서 `java/c++` 스타일의 주석이 허용되며, 해당 행은 무시됨
    ```json
    // java syntax comments
    {"id":4, "name":"test"}
    ``` 
- `false`일 경우, `JSON 파일의 주석`은 `손상된 레코드`로 간주
  - 모드 설정에 따라 처리

##### allowNumericLeadingZeros(default: `false`)
- `enabled` 상태인 경우
  - `0`으로 시작되는 숫자가 허용됨
- `false`일 경우
  - 선행 `0`은 유효하지 숫자값으로 간주
  - 손상된 것으로 간주. 모드 설정에 따라 처리

##### allowSingleQuote(default: `true`)
- `'`을 사용하여 필드 표시 가능
- 사용 가능한 경우 `'`와 `"`가 모두 허용
- 이 설정에 관계없이, 따옴표 문자는 중첩 불가능
  - 값 내에 사용될때 `escape`되어야 함

##### allowUnquotedFiledNames(default: `false`)
- 따옴표 없는 `JSON field name`을 허용
- 이 옵션을 사용할 경우
  - 필드 이름에 **공백**은 불가

##### multiLine(default: `false`)
- `enabled` 상태인 경우
  - `JSON row`를 파싱하는 대신
  - 각 파일의 **콘텐츠**를 하나의 유효한 `JSON document`로 간주
  - 해당 스키마에 따라 **레코드**로 파싱
- 파일의 생산자가 완전한 `JSON document` 파일로 출력할 수 있는 경우
  - 이 옵션을 사용하도록 하기
- 이 경우, 레코드를 그룹화 하기 위해 **최상위 배열** 사용
- 예시
  ```json
  [
    {"firstname":"Alice", "last name": "Wonderland"},
    {"firstname":"Caraline", "last name": "Spin"}
  ]
  ```

##### primitiveAsString(default: `false`)
- `enabled` 상태인 경우
  - 기본값 유형은 **문자열**로 간주
- 이를 통해 **혼합 유형의 필드**가 있는 `document`를 읽을 수 있지만
  - 모든 값을 `String`으로 읽음
- 예시
  ```json
  [
    // `15`와 `unknown` 모두 `String` 유형
    {"firstname":"Alice", "last name": "Wonderland", "age":15},
    {"firstname":"Caraline", "last name": "Spin", "age": "unknown"}
  ]
  ```

### 10.3.5. CSV 파일 소스 형식
- `CSV`는 값이 `,`로 구분되어 있음을 나타내지만
  - 종종 **분리 문자**를 자유롭게 지정 가능
- 데이터가 **일반 텍스트**에서
  - **구조화된 레코드**로 변환되는 방식을 제어하는 데
  - 사용할 수 있는 **많은 구성 옵션**이 있음
- 서식 관련 옵션에 대해서는 **최신 문서**를 살펴 볼 것

#### CSV 파싱 옵션

##### comment(default: `""`)
- **주석**으로 간주되는 줄을 표시하는 문자 구성
- `option("comment", "#")`을 사용하면
  - `#`로 시작하는 주석을 포함하여 파싱 가능

##### header(default: `false`)
- 스키마가 제공되어야 할 경우 **헤더 행은 무시**되며 효과가 없음

##### multiline(default: `false`)
- 각 파일의 모든 행에 걸쳐있는 **하나의 레코드**로 간주

##### quote(default: `"`)
- **열 구분 기호**를 포함하는 값을 묶는데 사용하는 문자

##### sep(default: `,`)
- 각 행의 필드를 구분하는 문자 구성

### 10.3.6. parquet 파일 소스 형식
- `apache parquet`은 **컬럼 지향적**인 **파일 기반 데이터 저장 형식**
- **내부 표현**은
  - 원본 행을 **압축 기술**을 사용하여 **저장된 열 청크**로 나눔
- 결과적으로 **특정 열**이 필요한 쿼리는
  - 전체 파일을 읽을 필요 없으며,
  - 관련 부분을 **독립적**으로 처리 및 검색 가능
- `parquet`은 복잡한 중첩 데이터 구조를 지원하고
  - **데이터의 스키마 구조**를 유지
- **향상된 쿼리 기능**, 스토리지 공간의 효율적인 사용 및
  - 스키마 정보의 보존으로 인해 **parquet**은 크고 복잡한 데이터셋을 저장하는 데 널리 사용하는 형식

#### 스키마 정의
- `parquet` 파일에서 **스트리밍 소스**를 작성하려면
  - **데이터 스키마** 및 **디렉터리 위치**를 제공하는 것으로 충분
- 스트리밍 선언 중에 제공되는 스키마는 **스트리밍 소스 정의 기간**동안 고정
  
##### CODE.10.4. parquet 소스 예제 구축하기
  ```scala
  // 형식과 로드 경로 사용
  val fileStream = spark.readStream
      .schema(schema)
      .parquet("hdfs://data/folder")
  ```

### 10.3.7. 텍스트 파일 소스 형식
- **구성 옵션**을 사용하면
  - 텍스트를 **한 줄씩** 또는
  - 전체 파일을 **단일 텍스트 blob**으로 수집 가능
- 이 소스에서 생성된 데이터의 스키마는 `StringType`

#### 텍스트 흡수 옵션
- 전체 텍스트 옵션을 사용하여 텍스트 파일을 **전체적으로 읽을 수 있도록** 지원

##### wholetext(default: `false`)
- `true`면 전체 파일을 `단일 텍스트 blob`으로 읽음
- 그렇지 않으면 **표준 줄 구분 기호**(`\n, \r\n, \r`)을 사용하여
  - 텍스트를 줄로 나누고, **각 줄을 레코드**로 간주

#### text와 textFile
- **텍스트 형식 사양**을 **종료 메서드 호출** 또는 **형식 옵션**으로 사용할 수 있음
- **정적**으로 유형이 지정된 `dataset`을 얻으려면
  - `StreamBuilder`의 마지막 호출로 `textFile`을 사용해야 함
- 텍스트 형식은 두가지 API 대안을 지원

##### text
- `StringType` 형식의 단일 `value` 필드를 사용하여
  - 동적으로 형식화된 `dataframe`을 반환

##### textFile
- 정적으로 타입이 지정된 `Dataset[String]`을 반환

##### CODE.10.5. 텍스트 형식 API 사용법
```scala
// 형식으로 지정된 text
val fileStream = spark.readStream.format("text").load("hdfs://data/folder")

// 전용 메서드를 통해 지정된 text
val fileStream = spark.readStream.text("hdfs://data/folder")

// 전용 메서드를 통해 지정된 textFile
val fileStream = spark.readStream.textFile("/tmp/data/stream")
```

## 10.4. 카프카 소스
- **분산 로그**를 개념을 기반으로 하는 pub/sub(publish/subscribe) 시스템
- kafka에서 조직 단위는 topic
- 카프카의 **구조적 스트리밍 소스**는 `subscriber`역할을 구현하며,
  - 하나 이상의 토픽에 게시된 데이터를 사용할 수 있음
  - 이는 **신뢰할만한 소스**
    - 스트리밍 프로세스의 **부분** 또는 **전체 실패** 및 **재시작**의 경우에도
      - **데이터 전송 시멘틱**이 보장됨

### 10.4.1. 카프카 소스 설정
- `SparkSession`에서 `createStream` builder와 `format("kafka")` 메서드를 함께 사용 
- 카프카에 연결하려면 **카프카 브로커**의 주소와
  - 연결하려면 **토픽**이라는 **두 가지 필수 파라미터**가 필요
- 카프카 브로커의 주소는
  - `kafka.bootstrap.servers` 옵션을 통해 쉼표로 구분된 `host:post`쌍 목록을 포함하는 `String`으로 제공

#### CODE.10.6. kafka 소스 생성하기
```scala
val kafkaStream = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "host1:port1,host2:port2,host3:port3")
      .option("subscribe", "topic1")
      .option("checkpointLocation", "hdfs://spark/streaming/checkpoint")
      .load()

// sql.DataFrame = [key: binary, value: binary, ...]
```
- `key, value, topic, partition, offset, timestamp, timestampType`
  - 7개 필드가 있는 `DataFrame`
- 이 스키마는 **카프카 소스**로 고정
  - 카프카의 원시 `key, value` 및 사용된 **각 레코드의 메타데이터**를 제공
- 일반적으로는 `key, value`에만 관심
  - 모두 `ByteArray`로 표시되는 이전 `payload`를 포함
- `StringSerializer`를 사용하여 카프카에 데이터를 쓰면
  - 예제의 **마지막 표현식**에서와 같이 값을 `String`으로 캐스팅하여 해당 데이터를 다시 읽을 수 있음
- **텍스트 기반 인코딩**은 일반적인 방법이나 데이터를 교환하는 가장 **공간 효율적**인 방법은 아님
- 스키마 인식 `AVRO`형식과 같은 **다른 인코딩**은
  - **스키마 정보 임베딩**의 추가 이점으로 인해 더 나은 공간 효율성을 제공할 수 있음
- `topic, partition, offset`과 같은 메세지의 추가적인 **메타데이터**보다 복잡한 시나리오에서 사용 가능
- 레코드를 생성한 `topic`이 포함되며 동시에
  - 여러 토픽을 구독하는 경우 **레이블 또한 판별자**로 사용할 수 있음

### 10.4.2. 토픽 구독 메서드 선택하기
- 토픽을 지정하는 **세 가지 방법**
  - subscribe
  - subscribePattern
  - assign
- 위 옵션중에 **하나만** 포함되어야 함
- 토픽과 **구독할 파티션**을 선택할 수 있는 **다양한 수준**의 **유연성**을 제공함

#### subscribe
- 단일 **토픽**또는 **쉼표**로 구분된 토픽 목록(`topic1, topic2, ..., topic n`)
- 이 메서드는 각 **토픽**을 구독하고 
  - 모든 **토픽**의 **통합 데이터**가 포함된 **단일 통합 스트림**을 작성
- `.option("subscribe", "topic1, topic3")`

#### subscribePattern
- `subscribe` 동작과 유사하지만 **토픽**은 **정규 표현식** 패턴으로 지정
- `.option("subscribePattern"), " factory[\\d]+Sensors")`

#### assign
- 토픽당 **특정 파티션**의 세부 스펙을 사용할 수 있음
- `TopicPartition`으로 알려짐
- 토픽당 파티션 `JSON`객체를 사용하여 표시되며,
  - 각 키는 **토픽**이며 그 값은 **파티션 배열**
- `.option("assign", """{"sensors": [0,1,3]}""")`
  - 토픽 센서의 파티션 `0,1` 및 `3`을 구독
- 이 방법을 사용하려면 **토픽 파티셔닝**에 대한 정보 필요
- 카프카 API를 사용하거나 구성을 통해 프로그래밍적인 방식으로 **파티션 정보**를 얻을 수 있음