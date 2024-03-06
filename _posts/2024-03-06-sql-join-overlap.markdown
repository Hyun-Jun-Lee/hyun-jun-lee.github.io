---
title: "SQL JOIN 연산시 중복이 발생하는 상황"
date: 2024-03-06
category: [SQL]
tag: [sql, orm, sqlalchemy]
---

예전에도 같은 경험을 한 적이 있었고, 이번 기회에 정리를 해보려고 한다.

```python
    def get_multi(
        self,
        db: Session,
        order_by: str,
        skip: int,
        limit: int,
        creator_nationality: Optional[str],
        user_id: int = None,
        is_active: bool = True,
        tags_ids: Optional[List[int]] = None,
        topics_ids: Optional[List[int]] = None,
        time_filters: Optional[List[str]] = None,
        is_count_only: Optional[bool] = False,
        search_word: str = None,
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

        if order_by == MeetingOrderingEnum.CREATED_TIME:
            query = query.order_by(desc(Meeting.created_time))
        elif order_by == MeetingOrderingEnum.MEETING_TIME:
            query = query.order_by(asc(Meeting.meeting_time))
        elif order_by == MeetingOrderingEnum.DEADLINE_SOON:
            query = query.filter(Meeting.meeting_time > func.now())

            time_difference_seconds = extract(
                "epoch", Meeting.meeting_time - func.now()
            )
            query = query.order_by(time_difference_seconds)
        else:
            query = query.order_by(desc(Meeting.created_time))

        total_count = query.count()
        if is_count_only:
            return [], total_count

        return query.offset(skip).limit(limit).all(), total_count
```

위 함수에서 `query.offset(skip).limit(limit).all()`와 `total_count` 가 일치하지 않는 에러가 발생했다. 즉, `len(query.offset(skip).limit(limit).all()) != total_count` 인 상황.
처음엔 skip, limit 같은 페이지네이징 때문에 발생한 에러인지 알았으나, 계속 확인해본 결과 페이지네이션과는 관련이 없었다.

이것저것 확인하다보니 `filter_by_topics()` 메서드에서 topic_ids에 '기타' 가 포함되는 경우 에러가 발생함을 알게 되었다.

```python
    def filter_by_topics(self, query, db: Session, topics_ids: Optional[List[int]]):
        etc_tag = db.query(Topic).filter(Topic.kr_name == "기타").first()
        if etc_tag and etc_tag.id in topics_ids:
            return (
                query.join(MeetingTopic)
                .join(Topic)
                .filter(or_(Topic.is_custom == True, Topic.id.in_(topics_ids)))
            )
        return query.join(MeetingTopic).join(Topic).filter(Topic.id.in_(topics_ids))
```

위 메서드에서는 `Meeting`, `Topic` 그리고 다대다 테이블인 `MeetingTopic'을 join하고 있다. 
`JOIN` 연산은 여러 테이블의 행을 결합해서 수직으로 확장된 테이블을 생성하는데, `Meeting`이 여러 `Topic`에 해당하는 경우 해당 `Meeting`은 결과에서 중복되어 나타날 수 있다.
이런 상황은 join 하는 테이블이 1:N, N:M 상황에서 자주 발생한다.

- 1:N 관계: 한 테이블의 행이 다른 테이블의 여러 행과 관련되어 있을 때, 조인 연산은 첫 번째 테이블의 해당 행을 다른 테이블의 각 관련 행과 결합하여 중복된 결과를 생성합니다.
    - ChatGPT에게 1:N관계에서 중복이 발생하는 상황의 예시를 부탁했다.

    ![image](https://github.com/HackSoftware/Django-Styleguide/assets/76996686/9801df03-105f-4b3f-b50c-6dfdee1279ae)


- N:M 관계: 두 테이블 간에 다대다 관계가 존재하는 경우, 중간 테이블(연결 테이블)이 필요하며, 이 중간 테이블을 통한 조인은 두 테이블 간의 복잡한 관계를 반영하여 더 많은 중복된 결과를 생성할 수 있습니다.

    ![image](https://github.com/HackSoftware/Django-Styleguide/assets/76996686/f6ce9d92-7c7e-48f9-ae4f-c7904cde9bda)

그리고 `query.count()` 메서드는 내부적으로 `SELECT COUNT(*)` 쿼리를 실행하는데 SQL의 COUNT 함수는 중복을 고려하지 않고 순수하게 결과에서 행의 인덱스만 카운트 하기 때문에 `JOIN` 연산으로 인해 동일한 `Meeting`이 여러번 카운트 되서 query.all()과 다르게 결과가 나올수 있다.

나는 문제를 해결하기 위해 `distinct()` 메서드를 사용했다.

```python
total_count = query.distinct().count()
```

