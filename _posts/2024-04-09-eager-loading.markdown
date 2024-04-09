---
title: "쿼리 최적화, Eager Loading vs Lazy Loading"
date: 2024-04-09
category: [DB,]
tag: [db, sql, orm, n+1]
---

사이드프로젝트에서 조회하는 부분의 속도를 개선시키기 위한 노력들을 기록해보려고한다.

n+1 문제에 대해 알고는 있었지만, 막상 이 것을 해결해보려고 한 적은 별로 없는 것 같다. 
이 전에 검색 속도 향상을 위해 like 연산에서 full-text search로 리팩토링을 했고 이번엔 실제 SQL 쿼리를 보며 최적화를 해보려고 한다.

# Lazy Loading

Lazy Loading은 DB, API 호출, 또는 파일 시스템과 같은 자원 집약적인 연산에서 데이터를 실제로 필요로 할 때까지 로드를 지연시키는 기법. 
이는 ORM에서 주로 사용되며, 연관된 객체를 처음부터 로드하지 않고 해당 객체에 접근하는 순간에 DB로부터 데이터를 로드

실제 연관 객체를 사용하려 할 때 DB로 부터 로드해오기 때문에 초기 로딩 시간을 최소화하고 메모리 사용을 최적화 할 수 있음.
하지만, 데이터 로드 시점을 관리해야 해서 복잡하고 n+1 문제를 야기할 수 있어 성능을 예측하기 어려움

# Eager Loading

Eager Loading은 쿼리 최적화 전략 중 하나로 관련 엔티티, 객체를 최초 쿼리 실행 시 함께 로드하는 방법.

간단하게 설명하자면 User - Profile 관계가 있다면 User를 조회할 때, Profile도 함께 조회하는 것.
만약 User 조회하는 쿼리를 실행할 때 Profile을 같이 가져오지 않는다면, 조회된 User 객체의 수 만큼 다시 Profile 을 조회하는 쿼리를 DB에 요청해야함(N+1 문제)

## With Sqlalchemy

```python
class User(Base):
    id = Column(Integer, primary_key=True)
    name = Column(String)
    profile = relationship("profile", backref="profile")

class profile(Base):
    id = Column(Integer, primary_key=True)
    nick_name = Column(String)
    user_id = Column(Integer, ForeignKey('users.id'))
```

- `joinedload`
    - JOIN으로 관련 객체를 한번의 쿼리로 미리 로드
    - 주로 1:1, 1:N 관계에 주로 사용
    - 추가 쿼리를 방지하지만, 데이터 양에 따라 결과 집합이 너무 커질 수 있음
    - `users = session.query(User).options(joinedload(User.posts)).all()`
    ```sql
    SELECT user.id, user.name, profile.id, profile.nick_name, profile.user_id
     LEFT OUTER JOIN profile ON user.id = profile.user_id
    ```


- `subqueryload`
    - 관련 객체를 로드하기 위해 별도의 SELECT 쿼리 실행
    - 원본 쿼리의 결과를 사용하는 서브쿼리를 생성
    - 주로 1:N 관계에 쓰이고, `joinedload` 보다 복잡한 시나리오에 사용 가능
    - `users = session.query(User).options(subqueryload(User.profile)).all()`
    ```sql
    SELECT user.id, user.name
    FROM user

    SELECT profile.id, profile.nick_name, profile.user_id
    FROM profile
    WHERE profile.user_id IN (SELECT user.id FROM user)
    ```


- `selectinload`
    - `subqueryload`와 같이 관련 객체를 위해 별도의 SELECT 쿼리 실행
    - 서브쿼리 대신 python에서 결과의 ID 집합을 이용해 SELECT 쿼리 실행
    - DB에서 하는 작업을 python 단 에서 처리하는 것
    - `users = session.query(User).options(selectinload(User.profile)).all()`
    ```sql
    SELECT user.id, user.name
    FROM user

    SELECT profile.id, profile.nick_name, profile.user_id
    FROM profile
    WHERE profile.user_id IN (1, 2, ...)
    ```


## With Django ORM

- `select_related`
    - JOIN을 사용해 관련 객체 한번에 로드
    - `users = User.objects.select_related('profile').all()`
    ```sql
    SELECT user.id, user.name, profile.id, profile.nick_name, profile.user_id 
    FROM user INNER JOIN profile ON (user.id = profile.user_id);
    ```


- `prefetch_related`
    - 관련 객체를 로드하기 위해 별도의 쿼리 실행
    - ManyToManyField 또는 역참조 Forreignkey 관계에 사용
    - SQLAlchemy의 `selectinload`과 같이 python 단에서 첫번째 쿼리의 결과를 이용
    - `users = User.objects.prefetch_related('profile').all()`
    ```sql
    SELECT user.id, user.name FROM user;

    SELECT profile.id, profile.nick_name, profile.user_id FROM profile WHERE profile.user_id IN (<user_id 리스트>);
    ```


## 프로젝트 적용

### Before

```python
    def get_multi(
        self,
        ...
    ) -> List[Meeting]:
        query = db.query(Meeting).filter(Meeting.is_active == is_active)

        if user_id:
            query = self.filter_by_ban(db, query, user_id)

        if creator_nationality:
            query = self.filter_by_nationality(query, creator_nationality)

        if time_filters:
            query = self.filter_by_time(query, time_filters)

        if tags_ids:
            query = self.filter_by_tags(query, tags_ids)

        if topics_ids:
            query = self.filter_by_topics(query=query, topics_ids=topics_ids, db=db)

        if search_word:
            query = self.filter_by_search_word(query=query, search_word=search_word)

        ...


        return query.offset(skip).limit(limit).all(), total_count
```

기존 메서드에서는 필터링 작업 후 그대로 return 해줬는데, 실제 발생한 쿼리를 확인해보니 Meeting 객체의 수 마다 MeetingTag, MeetingTopic, Meeting.Creator 등을 조회해서 가져오고 있었다.

### After

```py
    def get_multi(
        self,
        ...
    ) -> List[Meeting]:
        query = db.query(Meeting).filter(Meeting.is_active == is_active)

        if user_id:
            query = self.filter_by_ban(db, query, user_id)

        if creator_nationality:
            query = self.filter_by_nationality(query, creator_nationality)

        if time_filters:
            query = self.filter_by_time(query, time_filters)

        if tags_ids:
            query = self.filter_by_tags(query, tags_ids)

        if topics_ids:
            query = self.filter_by_topics(query=query, topics_ids=topics_ids, db=db)

        if search_word:
            query = self.filter_by_search_word(query=query, search_word=search_word)

        ...

        meeting_list = (
            query.options(
                joinedload(Meeting.meeting_tags).joinedload(MeetingTag.tag),
                joinedload(Meeting.meeting_topics).joinedload(MeetingTopic.topic),
                joinedload(Meeting.creator).joinedload(User.profile),
            )
            .offset(skip)
            .limit(limit)
            .all()
        )

        return meeting_list, total_count
```

`joinedload` 메서드로 첫 쿼리에 필요한 객체들을 모두 불러오게 하였다. 
실행시간을 체크해봤을 땐 기존 방식은 400ms~700ms, Eager Loading을 적용한 방식은 200~500ms 정도 나오는 듯 하다.

나는 Eager Loading 방식이 무조건 효율적일 것이라 생각했지만, 막상 직접 구현해보면 상황에 따라 다 다른 듯하다.
예를들어, 위 예시에서 상황에 따라 filter 해야할 조건들이 다 다른데 굳이 처음에 모든 필터 상황을 고려해서 Eager Loading 해오는 것은 굉장히 비효율적이다.

즉, 간단하게 이것만 하면 된다! 같은 정답은 없고 상황마다 적절히 사용하는 것이 최적화의 길!



### ETC

- 실제 쿼리를 확인하는 방법

```py
import logging

sql_logger = logging.getLogger("sqlalchemy.engine")
```


- 실행시간 체크를 위한 미들웨어 추가

```py
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.middleware.cors import CORSMiddleware
from starlette.requests import Request


class RequestTimeMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start_time = time.time()
        response = await call_next(request)
        process_time = time.time() - start_time
        print(
            f"Request: {request.method} {request.url.path} - Completed in {process_time:.4f} secs"
        )
        return response

app.add_middleware(RequestTimeMiddleware)
```