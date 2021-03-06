# [CHAP.03] 스트리밍 아키텍처
- 클러스터가 몇 개 팀간의 공유 리소스라면,
  - 클러스터의 가치가 지속적으로 하락하므로,
  - **multitenancy 문제**를 잘 다루어야 함
- 두 팀의 요구가 다를 때, 
  - 각 팀에 공정하고, 안전한 클러스터 **리소스 접근**을 제공해야 함
  - 동시에 컴퓨팅 리소스가 시간에 따라 **잘 활용되도록 하는 것**이 중요
- **모듈성**을 통해 이러한 이질성을 해결하도록 함
- 여러 **기능 블록**이 데이터 플랫폼의 상호 교환 가능한 부분으로 등장
  - 데이터베이스 스토리지를 **기능 블록**이라고 할 때,
  - 일반적으로 `PostgreSQL`이나 `MySQL`과 같은 관계형 db지만
  - **스트리밍 어플리케이션**이 매우 높은 처리량으로 데이터 쓰기를 해야할 경우
    - **apache cassandra**와 같은 확장 가능한 컬럼 지향 데이터베이스가 더 나은 선택일 것
- 스트리밍 데이터 플랫폼의 아키텍처를 구성하는 여러 부분
  - 완전한 솔루션에 필요한 다읗 요소들과, 관련된 처리 엔진의 위치 파악
- `Lambda`와 `Kappa Architecture`

## 3.1. 데이터 플랫폼의 구성 요소
- 스키마 하단의 `bare-metal` 수준에서 비즈니스 요구 사항이 요구하는 실제 데이터 처리로 넘어가면, 다음과 같은 내용 존재

### 하드웨어 수준
- `On-promise 하드웨어 설치`, 데이터 센서, OS가 설치된 클라우드 솔루션
  - `On-promise` : 서버를 클라우드 환경이 아닌 **자체 설비**로 보유하는 것

### 지속성 수준
- 인프라 위에 기계가 **계산 결과**뿐만 아니라, **입력**도 저장하기 위해
  - **지속성 솔루션**에 **공유 인터페이스**를 제공하는 것이 예상 됨
- 이 단계에서 많은 **분산 스토리지 솔루션** 중 `HDFS`와 같은 **분산 스토리지 솔루션**을 찾을 수 있음
- 클라우드에서는 이 지속성 계층이 `Amazon S3`나 `Google Cloud Storage`와 같은 전용 서비스로 제공

### 리소스 매니저
- 클러스터에서 **실행할 작업을 제출**하기 위한 **단일 협상 지점**을 제공
- `yarn`, `mesos`와 같은 `Resource Manager`의 과제고,
- `c8s` 같은 **클라우드 네이티브**시대의 보다 **진화된 스케줄러**의 과제

### 실행 엔진
- 실제 계산을 수행하는 **실행 엔진**
- 프로그래머의 **입력**와 **인터페이스**를 유지하고
  - 데이터 **조작을 설명**하는 것
- `apache spark, flink` 및 `Map-Reduce`등이 존재

### 데이터 수집 구성 요소
- 실행 엔진에 직접 연결될 수 있는 **데이터 수집 서버**
- 분산 파일 시스템에서 데이터를 읽는 오래된 관습을 보완하거나,
  - **실시간**으로 조회할 수 있는 다른 **데이터 소스**로 대체
- `apache kafka`와 같은 **메세징 시스템**이나 **로그 처리 엔진의 영역**이 이 수준에 해당

### 처리된 데이터 싱크
- 실행 엔진의 **출력** 쪽에서 다른 분석 시스템(**ETL**(Extract, Transform, Load) 작업을 수행하는 실행엔진의 경우)
- `NoSQL` 데이터베이스 또는 기타 다른 서비스일 수 있는 **고급 데이터 싱크**를 찾을 수 있음

### 시각화 계층
- 기존의 **정적 보고서**에서 벗어나
  - 종종 일부 **웹 기반 기술**을 사용하여
  - 보다 **실시간 시각 인터페이스**로 변화함

### 정리
- 이 아키텍쳐에서 **스파크**는 **컴퓨팅 엔진**으로서
  - **데이터 처리 기능**을 제공하는데 초점을 맞추고
  - 그 그림의 **다른 블록**과의 기능 인터페이스를 가질 필요성이 있음
- 특히 `RM`으로서 `yarn, mesos, k8s`와 인터페이스 하고, 많은 데이터 소스에 **커넥터**를 제공하는 동시에
- 새로운 데이터 소스는 쉽게 확장 가능한 `API`를 통해 추가하고,
  - **출력 데이터 싱크**와 통합하여 **업스트림 시스템에 결과를 제시**할 수 있는 **클러스터 추상화 레이어**를 구현한다

## 3.2. 아키텍처 모델
- 람다 아키텍처
  - 병렬로 실행되는 **배치 상대**와 **스트리밍 어플리케이션**을 복제
- 카파 아키텍처(Kappa Architecture)
  - 어플리케이션의 두 버전을 비교해야할 경우
  - 둘 다 **스트리밍 어플리케이션**이어야 한다
- 카파 아키텍처가 일반적으로 구현하기 쉽고 가벼움에도
  - **람다 아키텍처**가 필요

## 3.3. 스트리밍 어플리케이션에서 배치 처리 구성 요소의 사용
- 배치 어플리케이션 -> 스트리밍 어플리케이션
  - 배치 데이터셋, 주기적 분석을 대표하는 배치 프로그램이 필요
- **그린필드 어플리케이션**, **참조 배치 데이터셋**을 만드는 것이 관심일 수 있음
- 대부분의 **데이터 엔지니어**는
  - 한 번만 문제를 해결하는 것이 아닌
  - 그 솔루션을 다시 찾고, 그 가치가 성능에 연계되어있는 경우, **지속적으로 개선**
- **배치 데이터셋**은 수집된 이후
  - 더 이상 변경되지 않고 **테스트셋**으로 사용할 수 있다는
  - 벤치마크를 설정 가능
- **배치 데이터셋**을
  - **스트리밍 시스템**으로 재생하여, 이전 반복 또는, 알려진 벤치마크와 성능 비교 가능
- 배치와 스트림 처리 구성 요소 사이의 **상호작용**(3)
  - **코드 재사용**
    - 동일한 API를 공유하며, **데이터 입력/출력**만이 다름
  - **데이터 재사용**
    - 스트리밍 어플리케이션이 준비된 기능 또는 데이터 소스로부터
    - 정기적으로 배치 처리 작업으로 자체적인 공급을 수행하는 경우
  - **혼합 처리**
    - 어플리케이션이 그 자체의 **수명 주기** 동안 **배치**와 **스트리밍** 구성 요소를 모두 가지고 있는 것
    - 어플리케이션이 제공하는 **통찰력의 정확성**을 관리하고자 함
    - 어플리케이션 자체의 **버저닝**과 **진화**를 처리하는 방법
- 위 두가지 용도는 **편의성**이지만
  - 마지막 용도는 **배치 데이터셋을 벤치마크로 사용**하는 새로운 개념

## 3.4. 참조 스트리밍 아키텍처
- `시간의 경과에 따른 재생 가능성`과 `성능 분석`의 세계에는
  - 역사적이지만 **상반**되는 두가지 권고사항 돈재
- 스트리밍 어플리케이션의 성능 측정 및 테스트시, 설정에서 변화할 수 있는  부분(2)
  - **모델의 특성**(개선하려는 시도의 결과)
  - **모델이 작동하는 데이터**(유기적인 변화의 결과)

### 3.4.1. 람다 아키텍처
- 하루 전체 데이터를 기반으로 **새로운 배치 분석 버전**을 생산할 수 있을 때까지
  - 정기적으로 수행되는 **배치 분석**을 수행하고
- **데이터 발생**시 **스트리밍 개선**으로 **생성된 모델**을 보완
- 중요한 점(2)
  - 데이터 분석의 **과거 재생 가능성**은 중요
  - 새로운 데이터로부터 **결과를 진행할 수 있는 가능성** 또한 매우 중요
- 단점
  - 복잡하고, 같은 목적을 위해 **같은 코드의 두가지 버전**의 유지 필요
- 이 문제에 대한 대안적인 견해
  - 동일한 데이터셋을 **스트리밍 어플리케이션의 두 버전**에 공급할 수 있는 능력을 유지하기
    - 두 버전 : 새로운 개선된 실험 및 오래되고 안정된 작업을 의미

### 3.4.2. 카파 아키텍처
- 두 개의 **스트리밍 어플리케이션**을 비교
- **배치 파일**을 읽어야 할 경우
  - 간단한 **구성 요소**가 이 파일의 내용을 **레코드별로 기록**하여
  - **스트리밍 데이터 소스**로 재생할 수 있음
- 이 어플리케이션의 두 가지 버전에 **데이터를 공급**하는 데 구성되는 코드도 **재사용 가능**
  - **단순성**이 큰 이점
- 이 패러다임에서는 **중복 제거**가 없고, mental model이 더 단순
- 결국 배치 연산이 여전히 관련이 있는가에 대한 질문을 하게 됨
  - 람다 아키텍처의 몇몇 개념이 여전히 관련이 있음
  - 어떤 경우에는 매우 유용함
- 분석의 **배치 버전**을 구현한 다음, 그것을 **스트리밍 솔루션**과 비교하는 노력을 하는 것이
  - 여전히 유용한 몇몇 **사용 사례**가 있음

## 3.5. 스트리밍과 배치 알고리즘
- 스트리밍 어플리케이션의 **일반 아키텍처 모델**을 선택할때 고려할점(2)
  - 스트리밍 알고리즘이 완전히 다른 경우
  - 스트리밍 알고리즘이 **배치 알고리즘**에 비해 잘 측정될 수 있다고 보장 불가

### 3.5.1. 스트리밍 알고리즘은 때때로 완전히 다르다
- 배치/스트리밍 간 코드를 **재사용**할 수 없는 경우 존재
- **성능 특성**과 관련하여 높은 주의가 필요

#### 성능비/경쟁률
- **성능비** 또는 **경쟁률**은 **최적성**의 척도로 볼 때
- 알고리즘에 의해 반환되는 값이 **최적의 값**에서 얼마나 멀리 떨어져 있는지 나타내는 척도
- 알고리즘의 객관적인 값이
  - **모든 인스턴스**의 최적 오프라인 값의 `p배 이하`일 경우
  - 이 알고리즘은 공식적으로 `p 경쟁적`
- 수치가 **작을 수록** 좋으며
  - `1`이상의 경쟁률은
  - **스트리밍 알고리즘**이 **일부 입력**에서 훨씬 **나쁜 성능**을 보여준다는 의미

### 3.5.2. 스트리밍 알고리즘이 배치 알고리즘에 비해 잘 측정한다고 보장할 수는 없다
- 상자 채우기 문제(bin-packaging problem)
  - 서로 다른 크기/무게의 물체 집합의 입력
  - 여러개의 상자나 용기에 장착 되어야 함
  - 각각의 상자는 `크기/무게`면에서 정해진 **부피/용량**을 가져야 함
  - 용기의 사용수롤 최소화 하기
- **NP-난해(hard)**의 문제
- 문제를 간단히 변형하여 **결정**으로 변경 시, `NP-complete`로 변경됨
  - 그 개체 집합이 지정된 수의 상자에 들어갈지
- **온라인 알고리즘**의 경우, `greedy approximation algorithm`으로 처리
  - 더 나은 방법으로 `first fit decreasing strategy`(최적 적합 감소 전략)을 사용
    - 먼저 삽입할 항목을 크기에 대한 **내림차순**으로 분류한 다음
      - 각 항목을 **충분한 여백**으로
      - 목록의 첫번째 상자에 삽입하는 방법
    - 단, 이방법은 크기별로 **내림차순 분류**가 가능하다는 근거가 있어야 함
- **스트리밍 알고리즘**이 **배치 알고리즘**보다 성능이 우수하다는 보장이 없음
  - `예측 없이 작동해야 하기 때문`
- 중요한 점
  - 스트리밍 시스템은 실제로 **가벼움**
    - symentic은 **지연 시간**이 짧은 분석을 표현하는 용어로 사용할 수 있음
  - 스트리밍 API는 **휴리스틱에 한계**가 있는 스트리밍 또는 **온라인 알고리즘**을 사용하여 분석을 구현할 수 있게 함

## 3.6. 요약
- 결론적으로 **배치 처리의 소멸**은 과대평가 됨
- **배치 처리**는 적어도
  - **스트리밍 문제**에 대한 **성능의 기준선**을 제공하기 위해 남아 있음
- 담당 엔지니어는
  - **스트리밍 어플리케이션**과 **동일한 입력**에 대해
  - **나중에** 작동하는 **배치 알고리즘**의 성능을 잘 파악해야 함
- 주요 포인트(2)
  - 스트리밍 알고리즘에 대해 **알려진 경쟁률**이 주어져 있고
    - 그 **결과 성능**이 허용될 경우, **스트림 처리**만 실행하면 충분할 수 있음
  - 구현한 **스트림 프로세스**와 **배치 버전**사이에 **알려진 경쟁률**이 없는 경우
    - 정기적으로 **배치 계산을 실행**하는 것은
    - 어플리케이션을 보유할 수 있는 **중요한 벤치마크**