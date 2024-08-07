# 문제 이해 및 설계 범위 확정

### 요구사항

- 빠른 응답 속도: 페이스북 검색어 자동완성 시스템에 관한 문서에는 100밀리초 이내여야 한다.
- 연관성
- 정렬: 시스템의 계산 결과는 인기도 등의 순위 모델에 의해 정렬되어 있어야 한다.
- 규모 확장성: 많은 트래픽을 감당할 수 있도록 확장 가능해야 한다.
- 고가용성

### 개략적 규모 측정

- DAU: 천만명
- 평균적으로 사용자는 매일 10건의 검색 수행
- 질의할 때마다 평균적으로 20바이트 데이터를 입력한다고 가정
    - 문자 인코딩 방법으로는 ASCII를 사용한다고 가정할 것이므로, 1문자=1바이트다.
    - 질의문은 평균적으로 4개 단어로 이루어진다고 가정할 것이며, 각 단어는 평균 다섯 글자로 구성된다 가정한다.
    - 따라서 질의당 평균 20바이트다.
- 검색창에 글자를 입력할 때마다 클라이언트는 검색어 자동완성 백엔드에 요청을 보낸다. 따라서 평균적으로 1회 검색당 20건의 요청이 백엔드로 전달된다.
    - dinner라고 입력하면 다음 6개 요청이 순차적으로 백엔드에 전송된다.
        - search?q=d
        - search?q=di
        - search?q=din
        - search?q=dinn
- 대략 초당 24,000건의 질의(qps)가 발생
- 최대 qps = qps x 2 = 대략 48,000
- 질의 가운데 20% 정도는 신규 검색어라고 가정할 것이다. 따라서 대략 0.4GB 정도다. 매일 0.4GB의 신규 데이터가 시스템에 추가된다는 뜻이다.

# 개략적 설계안 제시 및 동의 구하기

개략적으로 보면 시스템은 두 부분으로 나뉜다.

### 데이터 수집 서비스(data gathering service)

- 사용자가 입력한 질의를 실시간으로 수집하는 서비스이다.
- 질의문과 사용빈도를 저장하는 빈도 테이블(frequency table)이 있다고 가정
    
    ![image](https://github.com/user-attachments/assets/c4389f08-005d-4219-b53d-c03dc46f01e3)
    

### 질의 서비스

![image](https://github.com/user-attachments/assets/8312f6aa-1b10-44ed-9cef-99449065a0b1)

- 이 상태에서 사용자가 “tw”를 검색창에 입력하면 아래의 “top5” 자동완성 검색어가 표시되어야 한다.
- “top5”는 위의 빈도 테이블에 기록된 수치를 사용해 계산한다고 가정한다.
- 데이터 양이 적으면 db sql로 계산할 수 있지만, 데이터가 많아지면 병목이 될 수 있다.

# 상세 설계

### 트라이 자료구조

- 루트 노드는 빈 문자열
- 각 노드는 글자 하나를 저장, 26개의 자식 노드를 가질 수 있음
- 각 트리 노드는 하나의 단어, 또는 접두어 문자열을 나타냄

![image](https://github.com/user-attachments/assets/c2b626fa-0e51-47f5-922c-3f3e3340f31a)

- p: 접두어(prefix)의 길이
- n: 트라이 안에 있는 노드 개수
- c: 주어진 노드의 자식 노드 개수
    - 모든 자식을 말하는건지..? 아니면 직속만?

가장 많이 사용된 질의어 k개는 다음과 같이 찾을 수 있다.

- 해당 접두어를 표현하는 노드를 찾는다. O(p)
- 해당 노드부터 시작하는 하위 트리를 탐색하여 모든 유효 노드를 찾는다. O(c)
- 유효 노드들을 정렬하여 가장 인기 있는 검색어 k개를 찾는다. O(clogc)

⇒ O(p) + O(c) + O(clogc)

하지만 최악의 경우, k개 결과를 얻으려고 전체 트라이를 다 검색해야 하는 일이 생길 수 있다. 이 문제를 해결할 방법으로는 다음 두 가지 방법이 있다.

1. 접두어의 최대 길이를 제한
    - 사용자가 검색창에 긴 검색어를 입력하는 일은 거의 없다.
    - 검색어 최대 길이를 제한(가령 50)하면 “접두어 노드를 찾는”단계는 O(p)에서 O(1)로 바뀔 것이다.
2. 각 노드에 인기 검색어를 캐시
    - 각 노드에 k개의 인기 검색어를 저장해두면 전체 트라이를 검색하는 일을 방지할 수 있다.
    - 각 노드에 5~10개 정도의 인기 질의어를 캐시하면 시간복잡도를 엄청나게 낮출 수 있다.
    - 하지만 저장 공간이 많이 필요하게 된다는 단점도 있다.
        
        ![image](https://github.com/user-attachments/assets/0f45690a-19ca-494c-92ad-550e76deedd3)
        

### 데이터 수집 서비스

사용자가 검색창에 타이핑 할 때마다 실시간으로 데이터를 수정하게 되면, 다음과 같은 문제가 있다.

1. 매일 수천만 건의 질의가 입력될 텐데 그때마다 트라이를 갱신하면 질의 서비스는 심각하게 느려질 것이다.
2. 일단 트라이가 만들어지고 나면 인기 검색어는 그다지 자주 바뀌지 않을 것이다.

![image](https://github.com/user-attachments/assets/1b814e2a-b33c-490b-823b-800fd8bda17a)

- 데이터 분석 서비스 로그
    - 검색창에 입력된 질의에 관한 원본 데이터가 보관된다.
    - 새 데이터가 추가될 뿐 수정은 이루어지지 않는다.
    
    ![image](https://github.com/user-attachments/assets/60d10ddc-d9aa-4955-9ca6-c0ba35fd3b5a)
    
- 로그 취합 서버
    - 데이터 취합 방식은 서비스 용례에 따라 달라질 수도 있다.
    - 트위터같은 실시간 애플리케이션 → 데이터 취합 주기를 짧게 가져간다.
    - **대부분 일주일에 한 번 정도로 로그를 취합**해도 충분할 것이다.
- 취합된 데이터
    - time 필드는 해당 주가 시작한 날짜를 나타낸다.
    - frequency는 해당 주에 사용된 횟수의 합이다.
        
        ![image](https://github.com/user-attachments/assets/4556b562-9f85-474c-9d04-cb73a6f89b08)
        
- 작업 서버
    - 주기적으로 비동기적 작업을 실행하는 서버 집합이다.
    - **트라이 자료구조를 만들고 트라이 DB에 저장하는 역할**을 담당한다.
- 트라이 캐시
    - 분산 캐시 시스템으로 트라이 데이터를 메모리에 유지하여 읽기 성능을 높이는 역할을 한다.
    - 매주 트라이 DB의 스냅샷을 떠서 갱신한다.
- 트라이 DB
    - 트라이 DB로 사용할 수 있는 선택지는 다음 두 가지가 있다.
        - 문서 저장소(document store)
            - MongoDB
        - key-value 저장소
            - 모든 접두어를 해시 테이블 키로 변환
            - 각 트라이 노드에 보관된 모든 데이터를 해시 테이블 값으로 변환

### 질의 서비스

![image](https://github.com/user-attachments/assets/cf8c7ad3-de18-4771-8ee4-d94ea2be2e4a)

1. 검색 질의 로드밸런서로 전송
2. 로드밸런서는 해당 질의를 API 서버로 전송
3. API 서버는 트라이 캐시에서 데이터를 가져와 요청에 대한 자동완성 검색어 제안 응답을 구성
4. 데이터가 트라이 캐시에 없는 경우, DB에서 가져와 캐시를 채운다.

질의 서비스 최적화 방안

- AJAX 요청(request): 브라우저는 AJAX 요청을 보내 자동완성된 검색어 목록을 가져온다. 장점은 요청을 보내고 받기 위해 페이지 새로고침이 필요 없다는 것이다.
- 브라우저 캐싱(browser chching): 제안된 검색어들을 브라우저 캐시에 넣어두고, 활용한다. 구글 검색 엔진이 이런 캐시 메커니즘을 사용한다. 구글은 제안된 검색어를 한 시간 동안 캐시해 둔다.
- 데이터 샘플링(data sampling): 대규모 시스템의 경우, 모든 질의 결과를 로깅해도록 해놓으면 CPU 자원과 저장공간을 엄청나게 소진하게 된다. N개 요청 가운데 1개만 로깅하도록 한다.

### 트라이 연산

- 트라이 생성
    - 트라이 생성은 작업 서버가 담당하며, 데이터 분석 서비스의 로그나 DB로부터 취합된 데이터를 이용한다.

- 트라이 갱신
    - 트라이 갱신 두 가지 방법
        1. 매주 한 번 갱신: 새로운 트라이를 기존 트라이를 대체
        2. 각 노드를 개별적으로 갱신하는 방법: 성능이 좋지 않음. 

### 검색어 삭제

![image](https://github.com/user-attachments/assets/2d4449b0-9c3f-479d-9264-9650afda21bf)

- 부적절한 질의어는 자동완성 결과에서 제거해야 한다.
- 트라이 캐시 앞에 필터 계층(filter layer)을 두고 부적절한 질의어가 반환되지 않도록 한다.
- 필터 계층을 두면 필터 규칙에 따라 검색 결과를 자유롭게 변경할 수 있다는 장점이 있다.
- DB에서 해당 검색어를 물리적으로 삭제하는 것은 다음번 업데이트 사이클에 비동기 적으로 진행하면 된다.

### 저장소 규모 확장

첫 글자를 기준으로 샤딩하는 방법

- 검색어를 보관하기 위해 두 대 서버가 필요하다면 ‘a’부터 ‘m’까지 글자로 시작하는 검색어는 첫 번째 서버, 나머지는 두 번째 서버에 저장한다.

과거 질의 데이터의 패턴을 분석하여 샤딩하는 방법

- 샤드 관리자(shard map manager)는 어떤 검색어가 어느 저장소 서버에 저장되는지에 대한 정보를 관리한다.
- 예를 들어 ‘s’로 시작하는 검색어 양이 ‘u’~’z’로 시작하는 검색어를 전부 합친 것과 비슷하다면 ‘s’에 대한 샤드 하나와 ‘u’~’z’까지의 검색어를 위한 샤드 하나를 두어도 충분할 것이다.

# 마무리

- 다국어 지원 가능 시스템으로 확장하려면?
    - 트라이에 유니코드 데이터를 저장해야 한다.
- 국가별 인기 검색어 순위가 다르다면?
    - 국가별로 다른 트라이를 사용하도록 하면 된다. 트라이를 CDN에 저장하여 응답속도를 높이는 방법도 생각해볼 수 있다.
- 실시간 변하는 검색어의 추이를 반영하려면?
    - 본 설계안은 적합하지 않다.
        - 작업 서버가 매주 한 번씩만 돌도록 되어 있어서 시의 적절하게 트라이를 갱신할 수 없다.
        - 때맞춰 서버가 실행된다 해도, 트라이를 구성하는 데 너무 많은 시간이 걸린다.
- 몇 가지 아이디어
    - 샤딩을 통해 작업 대상 데이터 양을 줄인다.
    - 순위 모델을 최근 검색어에 높은 가중치를 두도록 한다.
    - 데이터가 스트림 형태로 올 수 있다는 점
        - 데이터가 스트리밍 된다는 것은, 데이터가 지속적으로 생성된다는 뜻이다.