# [CHAP.21] 시간 기반 스트림 처리

## 21.1. 윈도우 집계
- `aggregation`은 스트림 데이터 처리에서 빈번한 패턴
  - producer와 consumer의 우려의 차이를 반영
- window
  - 오랜기간 걸친 집계를 만드는데 도움이 됨
- tumbling window, sliding window
- 일정 기간 동안 **특정 집계**를 실행하는데 필요한
  - **중간 메모리의 양**을 제한하기 위해
  - 윈도우를 통해 작동하는 특화된 **축소 기능**을 제공
  
### 21.2. 텀블링 윈도우
- 스파크 스트리밍에서 가장 기본적인 윈도우 정의
  - `Dstream`의 `window(<time>)`
- `DStream`은 원하는 로직을 구현하기 위해
  - 추가로 변환할 수 있는 새로운 `윈도우 DStream`을 만듦
- `hashtagg`의 `DStream`을 가정할 때, 텀블링 윈도우에서 다음 작업 수행
  ```scala
  val tumblingHashtagFrequency = hashTags.window(Seconds(60))
                                          .map(hashTag => (hashTag, 1))
                                          .reduceByKey(_ + _)
  ```
- `window`연산에는 `map`과 `remedByKey`단계(현재는 간단한 계산)
  - 전에 `DStream`을 분할하는 것을 재프로그래밍
- 원래 스트림인 `hashTags`는
  - **배치 간격**에 따라 엄격한 분할을 따름(배치당 RDD 1개)
- `60초`당 하나의 `RDD`를 포함하기 위해
  - 새로운 `DStream`인 `hashTags.window(Seconds(60))`을 구성
    - 시계가 60초를 체크할 때마다 동일한 윈도우가 있는 `DStream`의 이전 요소와는 독립적으로
    - 클러스터의 리소스에 새로운 `RDD`가 생성
- 그런 의미에서 윈도우는 **하락**(tumbling)하고 있음
  - 모든 RDD는 실시간으로 읽은 새로운 **신선한**요소들로 `100%` 구성됨

#### 21.2.1. 윈도우 길이와 배치 간격
- 윈도우 스트림의 작성은
  - 원본 스트림의 **여러 RDD 정보**를 **윈도우 스트림**에 대한 **단일 RDD**로 병합하여 얻어지므로
  - 윈도우 간격은 **배치 간격**의 **배수**이어야 함
- `초기 배치 간격`의 배수가 되는 **모든 윈도우 길이**는 **인수**르 전달될 수 있음
- 이러한 종류의 **그룹화된 스트림**을 사용하면
  - 사용자는 **윈도우 스트리밍 계산 런타임**의 `k`번째 간격에 대해
    - 보다 정확하게 `마지막 1분`, `마지막 15분` 또는 `마지막 1시간`의 데이터를 질문할 수 있음
- `윈도우 간격 = 스트리밍 어플리케이션의 시작`과 일치
  - e.g. 배치 간격이 `2분`인 `DStream`을 통해 `30분`간의 윈도우가 주어졌을 때
    - `10:11`에서 스트리밍 작업 시작시
    - 윈도우 간격은 `11:41, 12:11, 12:11, 12:41`에 계싼됨