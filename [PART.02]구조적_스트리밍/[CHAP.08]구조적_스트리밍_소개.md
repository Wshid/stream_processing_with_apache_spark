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