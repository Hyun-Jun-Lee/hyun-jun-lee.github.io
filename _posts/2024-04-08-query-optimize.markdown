---
title: "Full-Text 검색"
date: 2024-04-08
category: [DB,]
tag: [db, sql, orm, full-text]
---

사이드프로젝트에서 조회하는 부분의 속도를 개선시키기 위한 노력들을 기록해보려고한다.

대규모 검색 서비스는 ElasticSearch 서버를 따로 두고 사용하는데, 그렇게 하려면 서비스 DB와 ElasticSearch 간의 데이터 동기화에 너무 많은 공수가 들어가고 오버 스펙인 듯하다.
그리고 내 사이드프로젝트는 그정도의 규모의 데이터가 쌓이지 않을 가능성이 매우 크다.

## Full-Text Search

text에서 키워드를 검색하는 기능으로 like, 비교연산자와 달리 각 단어의 token화 및 정규화를 통해 검색.
PostgreSQL의 경우 `to_tsvector` 함수를 통해 tsvector 타입으로 변환할 수 있고 변환 후 `tsquery`를 이용해 매칭 여부를 확인할 수 있다.

주어진 Document를 형태소 단위로 분리해서 각 토큰을 숫자, 단어 등으로 분류하고, 분류된 토큰들은 정규화 과정을 통해 `lexem` 이라는 표준화 된 형태로 변환한다.
이 과정에서 전처리된 Documnet를 lexem 배열 형태로 저장 후 검색에 사용한다.

ex) 'run', 'runs', 'running'와 같은 단어는 모두 lexem 'run'으로 변환되어, 사용자가 'run'으로 검색했을 때 'running' 이 포함된 Document도 검색 가능


## like 연산보다 Full-Text Search가 빠른 이유?

- Like
    - 주어진 패턴과 일치하는 문자열을 찾기 위해 DB의 각 행을 순차적으로 검사
    - `%word%`과 같은 경우 인덱스 적용안되서 테이블을 Full Scan

- Full-Text Search
    - 텍스트를 사전 처리해서 키워드를 추출하고, 이를 통해 검색 인덱스를 생성함. 
    - 단어의 빈도, 위치, 중요도 등을 고려한 검색 가능
    - Full-Text Search 엔진은 검색 결과에 가중치를 부여하고 순위를 매길수 있음


LIKE연산은 전체 데이터를 순자척으로 스캔해야 하지만, Full-Text Search는 사전에 처리 된 인덱스를 사용하기 때문에 더 빠름

## 검색 연산자

- `||` : Concatenation, tsvector 또는 tsquery 객체를 결합할 때 사용
    - `SELECT 'a'::tsvector || 'b'::tsvector;  -- 결과는 'a'와 'b'를 포함하는 tsvector`

- `&` : AND, 두개의 tsquery가 모두 True 일때 True 반환. 즉, 두 검색 조건을 모두 만족하는 문서 찾을 때 사용.
    - `SELECT to_tsvector('english', 'The quick brown fox') @@ to_tsquery('quick & fox');  -- true`

- `|` : OR, tsquery 에서 사용할 때 하나라도 True인 경우 True 반환. 즉, 두 검색 조건 중 하나라도 만족하는 것을 찾을 때 사용
    - `SELECT to_tsvector('english', 'The quick brown fox') @@ to_tsquery('quick | dog');  -- true`

- `!` : NOT, 특정 단어를 포함하지 않는 문서 찾을 때 유용
    - `SELECT to_tsvector('english', 'The quick brown fox') @@ to_tsquery('!dog');  -- true`

- `<->` : FOLLOWED BY, 두 단어가 텍스트 내에서 바로 이어지는 순서로 나타나는 경우 사용, 특정 단어나 구문이 정확한 순서로 나타내야할 때 사용
    - `SELECT to_tsvector('english', 'The quick brown fox') @@ to_tsquery('quick <-> fox');  -- true`

- `N` : DISTANCE, 두 단어 사이의 최대 거리(단어 수)를 지정할 때 사용. 두 단어가 문서 내에서 N개 이하의 단어로 떨여져 있을 때 일치한다고 간주
    - `SELECT to_tsvector('english', 'The quick brown fox jumps') @@ to_tsquery('quick <2> jumps');  -- true`

## 프로젝트 적용

### Before

```python
    def filter_by_search_word(self, query, search_word: str):
        search_filter = or_(
            Meeting.name.like(f"%{search_word}%"),
            Meeting.description.like(f"%{search_word}%"),
            # MeetingLanguage를 통해 연결된 Language의 kr_name 또는 en_name 필드 검색
            Meeting.meeting_languages.any(
                MeetingLanguage.language.has(
                    or_(
                        Language.kr_name.like(f"%{search_word}%"),
                        Language.en_name.like(f"%{search_word}%"),
                    )
                )
            ),
        )

        return_query = query.filter(search_filter)
        return return_query
```

위 메서드를 실행하게 되면 실제 발생되는 쿼리는 아래와 같다. (sqlalchmey logger 활용)

```sql
SELECT *
FROM (
    SELECT *
    FROM meeting
    WHERE meeting.is_active = true 
    AND meeting.university_id = %(university_id_1)s 
    AND (
        meeting.name LIKE %(name_1)s 
        OR meeting.description LIKE %(description_1)s 
        OR (
            EXISTS (
                SELECT 1
                FROM meetinglanguage
                WHERE meeting.id = meetinglanguage.meeting_id 
                AND (
                    EXISTS (
                        SELECT 1
                        FROM language
                        WHERE language.id = meetinglanguage.language_id 
                        AND (
                            language.kr_name LIKE %(kr_name_1)s 
                            OR language.en_name LIKE %(en_name_1)s
                        )
                    )
                )
            )
        )
    ) 
    ORDER BY meeting.created_time DESC
    LIMIT %(param_1)s OFFSET %(param_2)s
) AS anon_1 
ORDER BY anon_1.meeting_created_time DESC

```

어질어질한 서브쿼리들;;


여기서 개선점

- 중첩되어 있는 EXISTS 서브쿼리 대신 JOIN을 사용해 단순화
- `LIKE` 검색은 인덱스 활용이 비효율적이기 때문에, 풀 텍스트 검색

### After

```python
    def filter_by_search_word(self, query, search_word: str):
        # 검색어를 풀텍스트 쿼리 형식으로 변환
        search_query = func.to_tsquery(f"{search_word}:*")

        # 텍스트 검색 벡터 생성
        name_vector = func.to_tsvector("simple", Meeting.name)
        description_vector = func.to_tsvector("simple", Meeting.description)
        language_kr_vector = func.to_tsvector("simple", Language.kr_name)
        language_en_vector = func.to_tsvector("simple", Language.en_name)

        # 텍스트 검색 벡터와 쿼리를 비교
        search_filter = (
            name_vector.op("@@")(search_query)
            | description_vector.op("@@")(search_query)
            | Meeting.meeting_languages.any(
                MeetingLanguage.language.has(
                    language_kr_vector.op("@@")(search_query)
                    | language_en_vector.op("@@")(search_query)
                )
            )
        )

        return_query = query.filter(search_filter)
        return return_query
```

- `to_tsvector()`
    - 주어진 텍스트를 토큰 단위 분리
    - 분리된 토큰을 정의된 configurations로 정규화
    - 텍스트에서 추출하고 정규화된 토큰을 포함하는 검색 벡터를 생성

기존 `like` 연산으로 수행할 때는 400ms~500ms 대의 속도였는데, 이렇게 Full-Text 검색을 이용하니 200ms~300ms 대로 약 0.1초 정도 줄일 수 있었다.
하지만 Full-Text 검색은 `%search_word%` 와 같은 기능은 지원하지 않기 때문에 `search_word:*` 와 같이 접미사 검색을 이용했다.

접미사 검색이기 때문에 meeting_name="같이 영화 봐요!" 일 때, "영화" 로 검색하면 해당 meeting이 집계되지 않을 것이라 생각했는데 검색이 잘 되서 찾아보니,
Full-Text 검색 시 "같이 영화 봐요!" 는 "같이", "영화", "봐요" 와 같이 3개의 토큰으로 분리되고 이 토큰들 중에서 사용자 검색어와 일치하는지 판단하기 때문에 조회가 되는 것이었다.

PostgreSQL에서는 다양한 언어에 대한 텍스트 검색을 지원하기 위해 여러 Configurations를 지원하고 있고 아래와 같이 확인할 수 있다.

```sql
SELECT cfgname FROM pg_ts_config;

  cfgname   
------------
 simple
 arabic
 armenian
 basque
 catalan
 danish
 dutch
 english
 finnish
 french
 german
 greek
 hindi
 hungarian
 indonesian
 irish
 italian
 lithuanian
 nepali
 norwegian
 portuguese
 romanian
 russian
 serbian
 spanish
 swedish
 tamil
 turkish
 yiddish
(29 rows)
```

한국어는 없기 때문에 default config를 사용 했고, 플러그인 설치를 통해 한국어를 위한 config도 설치할 수 있는 듯 하다.
