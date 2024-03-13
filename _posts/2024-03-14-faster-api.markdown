---
title: "API 응답 속도 향상 시키기"
date: 2024-03-14
category: [ETC]
tag: [etc, api]
---

API 개발은 항상 완료하고 나면 아쉬움이 남는 듯 하다. 지금까지 차피 API 응답 속도를 개선하는 방법은 DB에 날리는 쿼리의 최적화 뿐이라고만 생각하고 크게 신경쓰지 않았는데, 최근 링크드인에서 관련 글을 읽게 되어 되었고 관심이 생겨 찾아보게 되었다.

코드 최적화나, 쿼리 최적화 같이 기존에 알고 있던 방법말고 새롭게 알게된 것들을 정리해보려고 한다.


# API 응답 속도 향상 방법

## HEAD Method

HEAD 리퀘스트는 GET 요청처럼 서버에 리소스를 조회하지만, 데이터를 반환하지 않고 헤더만 반환하는 메서드라고한다. 

처음 이 방법을 봤을 땐 이걸 도대체 왜 하는 거지 라고 생각해서 찾아보니 리소스의 크기를 확인하거나 존재, 변경 여부, 유형을 확인하거나 서버의 기능을 확인할 때 사용한다고 한다.

주로 캐싱을 사용할 때, 캐싱 데이터와 원본 데이터의 변경 여부를 확인할 때 많이 사용한다고 한다.

이전에 검색 관련 기능을 개발할 때, 사용자가 선택한 검색 조건에 대해 갯수만 리턴해줘야 하는 경우가 있었는데 이럴 때 사용하면 딱 인 것 같아서 적용해보앗다.

- 기존코드
```python
@router.get("/meetings", response_model=MeetingListResponse)
def get_meeting(...):

    meetings, total_count = crud.meeting.get_multi(...)
    return {"meetings": meetings, "total_count": total_count}

    def get_multi(
        self,
        db: Session,
        order_by: str,
        skip: int,
        limit: int,
        # 검색 필터 파라미터들
    ) -> List[Meeting]:
        query = db.query(Meeting).filter(Meeting.is_active == is_active)

        ...

        total_count_query = query.distinct()
        total_count = total_count_query.count()

        ...

        if is_count_only:
            return [], total_count

        return query.offset(skip).limit(limit).all(), total_count
```

기존 코드에서는 is_count_only 라는 파라미터가 요청에 포함되면 count()만 해서 응답해주고 있는데, 이 로직에 HEAD method를 적용해보았다.

```python
@router.api_route("/meetings", methods=["GET", "HEAD"], response_model=MeetingListResponse)
async def get_meeting(request: Request, ...):  
    
    meetings, total_count = crud.meeting.get_multi(...)
    
    if request.method == "HEAD":
        headers = {"X-Total-Count": str(total_count)}
        return JSONResponse(content=None, headers=headers)
    else:
        return {"meetings": meetings, "total_count": total_count}
```

fastapi에서는 `api_route`를 이용해서 하나의 함수에서 2개의 요청을 처리할 수 있었다! 사실 기존 방식이랑 응답속도의 차이는 비슷했지만, 클라이언트 쪽에서는 body부분을 분석해보지 않아도 되니 좀 더 효율적이지 않을까

## 캐싱

캐싱은 간단하게 자주 사용되는 데이터 집합을 원본 스토리지가 아닌, 고속 스토리지에 저장해둬서 더 빠르게 접근해 가져올 수 있게 하는 방법이다.


### Local Caching

로컬 캐싱은 서버의 메모리 내에 데이터를 저장하는 방식으로, 캐시된 데이터는 해당 서버 내부에서만 사용할 수 있기 때문에, 개별 서버가 독립적으로 자주 접근하는 데이터를 캐싱하는 경우에 사용하기 적합

> 이와 반대대는 개념으로, 별도의 캐시서버를 두고 사용하는 Global Cache(Redis, Memcached) 가 있음. 보통 이런 외부 캐싱 툴 쓰겠지만, 혹시 모르니 알아는둘 필요가 있을 듯

python 에서는 로컬 캐싱을 지원하는 `cachetools` 라이브러리가 있고 python3.9 이상 부터는 `functools` 모듈에 기본으로 추가되어 있다.

- cachetools

```python
from cachetools import cached, TTLCache
from time import sleep

# 최대 100개 항목을 저장하며 각 항목은 10초 동안 유효한 캐시를 생성
cache = TTLCache(maxsize=100, ttl=10)

@cached(cache)
def get_result(number):
    return number * number
```

- functools

```python
from functools import cache

@cache
def factorial(n):
    return n * factorial(n-1) if n else 1
```

위의 두 방식 모두 처음에는 계산을 수행해야 하지만, 동일한 값을 다시 호출할 때는 캐싱하기 때문에 더 빠르게 값을 받을 수 있다. 

차이점이라면, `functools.cache`는 python 기본 모듈에 포함되어 있고 단순한 무한 캐시로 추가적인 설정 없이 바로 사용할 수 있지만 캐시 크기를 제한할 수 없기 때문에 메모리 샤용량 제한을 둘 수 없다.
`cachetools.cached`는 그와 반대로, 외부 라이브러리이고 캐시 전략에 대한 디테일한 설정을 할 수 있다.

### HTTP 캐싱

서버에서 응답할 때, HTTP 헤더를 추가해서 클라이언트나 프록시 서버에 캐싱을 지시하는 방법이다. 

```python
from fastapi import FastAPI, Response
from datetime import datetime, timedelta

app = FastAPI()

@app.get("/items/{item_id}")
def get_item(db,item_id: int, response: Response):
    item = db.get(item_id)

    response.headers["Cache-Control"] = "max-age=60"    
    return item
```

'max-age'는 서버로 부터 응답 받은 시간으로부터 캐싱될 수 있는 시간을 지정하는 지시어이고, 이외에도 캐시관련 지시어 많이 존재한다.


## 압축

말 그대롣 응답 데이터를 압축하는 방법으로, 서버 쪽에서도 압축하는 시간이 필요하고 클라이언트에서도 해제하는 시간이 필요하지만 대형 서비스에서는 필수적으로 사용되는 방법이라고 한다.

## 서버 상태 유지

많은 데이터를 항상 최신 데이터로 유지해야하는 은행, 결제 관련 프로그램에서 많이 사용되는 방법으로 엄청나게 많은 양의 데이터를 사용자가 조회 할 때 마다 조회해오는 것이 무리가 될 때 사용하는 방법이라고 한다.

예를 들어, 사용자가 로그인한 후 30분 동안은 서버와 클라이언트가 유저의 상태를 유지시키고 여러 요청에 대해 재요청은 상태를 유지하고 새로운 요청만 응답을 받아오는 방식이다. 이런 방식은 대부분 불변하는 정보에서 많이 쓴다고 한다.

## OverFetching 줄이기

클라이언트가 요청하는 데이터 보다 더 많은 데이터를 응답해주는 것을 OverFetching이라고 부른다고 한다. 나는 api 개발할 때 재사용성을 고려해서 항상 오버패칭을 해왔던 것 같은데, 이런 사소한 문제들을 지금이라도 발견해서 다행이다.

페이스북에서는 이런 문제를 해결하기 위해 `GraphQL`을 개발해서 클라이언트가 원하는 데이터를 정의해서 보낼 수 있도록 만들었다고 하는데 나중에 GraphQL에 대해서도 공부 해봐야겠다.




> Reference
    - https://medium.com/uplusdevu/%EB%A1%9C%EC%BB%AC-%EC%BA%90%EC%8B%9C-%EC%84%A0%ED%83%9D%ED%95%98%EA%B8%B0
    - https://brunch.co.kr/@skykamja24/671