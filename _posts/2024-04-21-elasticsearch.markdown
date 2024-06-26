---
title: "ElasticSearch 정리"
date: 2024-04-21
category: [DB,]
tag: [db, nosql, elasticsearch]
---

전 회사에서 Data Lake + Data WareHouse의 느낌으로 ElasticSearch를 사용했다. 그 때의 기억을 되살려 관련 정보를 정리해보고자 한다.

## 주로 쓰이는 분야

- App 검색 기능
- log 분석
- 모니터링
- 위치 기반 정보 데이터 분석

## ES의 구성요소 

### Cluster & Node

- Cluster
    - ES의 가장 상위 구성 단위로 하나 이상의 노드로 구성되고, 해당 노드들이 데이터를 저장하고 클러스터 전체에서 인덱싱 및 검색 작업을 수행
    - 여러 개의 ES Process(노드)를 논리적 결합을 통해 하나의 ES Process로 작동하게 하는 것
    - 여러 노드 중 일부 노드에 에러가 발생해도 나머지 노드들은 계속해서 서비스가 운영되기 때문에 고가용성 보장
    - 클러스터 상태
        - Green : 안정적이고 문제가 없음
        - Yellow : 운영은 되나 데이터 유실의 가능성 존재(Primary 샤드는 문제 없으나, Replica 샤드 사용 불가)
        - Red : 운영 불가 및 데이터 유실(Primary 샤드 유실 상태)

> 고가용성 : 시스템이나 네트워크가 높은 수준의 운영 시간(uptime)과 최소한의 다운타임을 보장하는 능력. 즉, 시스템이나 서비스가 계획되지 않은 중단 없이 지속적으로 작동할 수 있는 능력을 의미


- Node
    - 노드는 클러스터의 구성요소로서 클러스터 내의 작업을 분산처리함, 즉 클러스터를 구성하는 개별 서버
    - 각 노드는 고유한 ID를 가짐
    - 노드의 유형
        - Master Node : 클러스터의 관리를 담당하는 노드로 클러스터의 구성 변경(노드 추가/삭제, 인덱스 생성/삭제) 관리. `discovery.zen.minimum_master_nodes` 설정으로 네트워크 분할 발생 시 하나의 서브 클러스터만 마스터 노드를 가질 수 있게 하여 `Split Brain` 문제 해결 가능
        - Data Node : 실제 데이터를 저장하고 데이터와 관련된 CRUD 작업 및 검색 등의 쿼리를 처리하는 노드
        - Ingest Node : 데이터 전처리 담당, 인제스 노드는 인덱스 생성 전 document의 형식을 다양하게 변경 가능
        - Coordinating Node : 클라이언트로 요청 받아 적절한 노드로 요청을 전달하고 최종 결과를 클라이언트에 반환하는 역활, 모든 노드는 기본적으로 코디네이팅 노드의 역활을 수행할 수 있음


> Split Brain 문제 : 클러스터에 특정 에러가 발생해 네트워크 분할이 발생했을 때 나타나는 문제, 분리된 각 서브 클러스터는 독립적으로 마스터 노드를 선출하게 되고 각 서브클러스터마다 데이터를 변경하게되면 이후 에러를 해결하고 다시 합치게 될 때 데이터 일관성에 문제가 생김.

### etc vs RDBMS

|ES|RDBMS|
|------|---|
|Indax|Database|
|Shard|Partitioning|
|Type|Table|
|Document|Record|
|Field|Column|
|Mapping|Schema|
|DSL|SQL|
|||

- Index
    - Document의 집합으로 RDBMS에서 DataBase와 유사한 역활. 
    - 샤드로 분리되고 각 데이터 노드에 분산 저장됨

- Shard 
    - 인덱스 데이터를 분할하여 저장하는 논리적 단위, 샤드의 데이터는 `Segment` 라는 물리적 파일에 저장됨.
    - RDBMS에는 직접적인 매핑 개념이 없지만 데이터 분산을 위해 사용되는 '파티셔닝(Partitioning)'과 유사한 개념
    - 원본 데이터를 담는 Primary 샤드와 백업용 복제데이터를 저장하는 Replica로 나뉨(https://jeongxoo.tistory.com/17, 해당 블로그에 그림예시와 함께 상세한 설명)

- Type는 ES 7.0 이전에 Index 내에서 문서를 구분하는데 사용되었지만, 7.0 이후 부터는 없어짐

- DSL은 ES에서 검색, 집계 등을 위해 사용되는 JSON 기반의 쿼리 언어


## 역인덱스(Inverted Index)

역인덱스는 텍스트 기반 검색을 빠르게 수행할 수 있게 해주는 데이터 구조.

- RDBMS의 Index
    - 목적 : 테이블의 특정 컬럼에 대한 빠른 검색과 정렬이 주 목적
    - 구조 : B-Tree 또는 B+Tree 같은 Tree 기반 구조를 사용해서, key-value 쌍으로 데이터를 정렬하고 Key를 기준으로 빠른 검색 구현
        - B-Tree : 이진 트리를 확장한 형태로 노드 하나가 여러 개의 자식을 가질 수 있는 다진 트리. 각 노드는 여러 개의 Key를 저장할 수 있고 정렬된 상태로 유지된다. 
        - B+Tree : B-Tree의 변형으로, 모든 실제 데이터와 Key 값들이 리트 노드에만 저장되고, 내부 노드들은  리프 노드들을 가리키는 포인터만 저장
    - 동작 : 사용자가 특정 컬럼 기준으로 데이터 조회할 때, 인덱스는 해당 컬럼의 값을 빠르게 접근해 실제 데이터가 저장된 위치를 찾아줌


    > B-Tree 검색 과정 : 검색을 시작하면 루트 노드에서 시작해 Key 값을 비교하면서 검색하려는 키가 위치할 수 있는 자식 노드로 내려간다. 이 과정일 재귀적 반복하여 검색하려는 Key를 포함하고 있는 노드를 찾는다. 이 것은 트리가 정렬 된 상태로 유지되기 때문에 가능함.



- ES의 Inverted Index
    - 목적 : 텍스티 데이터의 Full-Text Search를 효율적으로 사용하기 위해 사용
    - 구조 : 단어를 Key, 해당 단어가 포함된 Document의 목록을 Value로 설정하는 데이터 구조, 각 단어마다 문서의 위치 정보를 저장해서, 검색어가 포함된 모든 문서를 빠르게 검색 가능
        - ex) 'beer'를 검색했다면 key 값은 'beer'가 되고, Value 값에는 'beer'라는 단어가 포함된 특정 Index 내부의 모든 Document들이 저장된다. 그리고 각 Document에서 'beer' 라는 단어의 위치, 빈도수 등이 저장되어 `Key : List[Value]` 형태가됨.
    - 동작 : 사용자가 검색어를 입력하면, ES는 역인덱스를 통해 해당 검색어가 포함된 모든 문서를 식별

### Analyzer

Inverted Index 생성과정에서 텍스트 데이터를 토큰화 하는 과정도 포함되는데, 이 토큰화를 담당하는 도구를 `Analyzer` 라고 부른다. 

analyzer는 클러스터나 특정 인덱스에 의해 정의되고 관련된 모든 노드에서는 사용할 수 있다.

- Character Filters : 텍스트를 분석하기 전에 사전 처리 단계에서 사용. HTML 태그 제거, 불필요한 문자 제거, 특정 문자를 다른 문자로 대체하는 작업을 수행. ex) "<"와 ">" 같은 HTML 태그를 제거하거나, "$" 문자를 "dollar"로 변경

- Tokenizer : 정규화된 텍스트는 토크나이저에 의해 개별 토큰으로 분리되는데, ES에서는 다양한 토크나이저를 제공함. 가장 기본적인 토크나이저는 공백을 기준으로 분리하는 `standard`, 특수한 문자를 구분자로 사용하는 `pattern`

- Token Filtering : 토큰화 과정 이후 'a', 'the' 와 같은 불용어(Stop Word), 동의어(Synonym) 등을 제거하는 역활


## ElasticSearch 에서의 Update

전 회사 프로젝트에서 ElasticSearch를 운영할 때, Heap Memory가 부족하다거나 log가 너무 많이 쌓여서 문제가 된 적이 있다. 

원인이 무엇인지 파악하다가 ElasticSearch에서는 document를 update 할 때, 해당 데이터가 실제로 저장된 Segement에서 update 되는 것이 아니라 업데이트된 document는 새로운 segement에 저장되고 기존 segement는 삭제되는 방식으로 처리된 다는 것을 알게됬다. 

```
{
  "name": "맛있는 식당",
  "location": "서울",
  "cuisine": "한식",
  "menus": [
    {
      "name": "김치찌개",
      "price": 8000,
      "ingredients": ["김치", "돼지고기", "두부"]
    },
    {
      "name": "된장찌개",
      "price": 7500,
      "ingredients": ["된장", "콩나물", "호박"]
    }
  ]
}

```

예를들어, 위의 예시에서 "김치찌개" 를 "부대찌개"로 변경한다고 가정해보면 Index에서 해당 Document는 삭제되고 새로 생성되는 것이다.
위와 같이 중첩 구조를 사용할 때, `nested` 타입을 지정할 수 있는데 해당 타입을 지정하면 중첩된 객체는 개별 문서로 취급되지만 상위 Document의 Segement에 데이터가 저장되기 때문에
상위, 하위 document 중 하나만 변경하더라도 모두 새로운 segement에 저장된다.

회사에서 운영 중인 ETL 파이프라인은 DB 로그를 읽어와 ElasticSearch에 저장하고, 이후 해당 데이터를 서비스 DB로 이전하는 과정에서 ElasticSearch에 대한 업데이트 작업을 수행합니다. 이러한 절차를 거친다면, 로그를 매번 읽어올 때마다 상당한 양의 쓰기(Write) 작업이 발생했을 것으로 추정됩니다.

어떻게 이 문제를 해결 할 수 있을까 검색하다보니 비슷한 상황을 해결한 글을 발견했다! https://techblog.woowahan.com/7425/