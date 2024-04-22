---
title: "ElasticSearch 정리"
date: 2024-04-21
category: [DB,]
tag: [db, nosql, elasticsearch]
---

전 회사에서 Data Lake + Data WareHouse의 느낌으로 ElasticSearch를 사용했다. 그 때의 기억을 되살려 관련 정보를 정리해보고자 한다.

## ES의 구성요소 

### Cluster & Node



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

- Shard는 인덱스 데이터를 분할하여 저장하는 단위, RDBMS에는 직접적인 매핑 개념이 없지만 데이터 분산을 위해 사용되는 '파티셔닝(Partitioning)'과 유사한 개념

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
