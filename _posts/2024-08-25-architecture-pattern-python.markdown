---
title: "Architecture Pattern With Python 1"
date: 2024-08-25
category: [PYTHON]
tag: [repository_pattern, declarative]
---

__Architecture Pattern With Python__


# Repository Pattern

Repository Pattern은 데이터 저장소를 더 간단히 추상화 한 것으로 이 패턴을 사용하면 모델 계층과 데이터 계층을 분리할 수 있다.

저장소 패턴은 영속적인 저장소를 추상화한 것으로 모든 데이터가 메모리상에 존재하는 것 처럼 가정해서 데이터 접근과 관련된 세부사항을 감출 수 있다.

## 데이터 접근에 DIP 적용

일반적으로 사용되는 Layer 아키텍처는 UI, 로직, DB로 이루어진 시스템을 구조화 할 때 일반적으로 쓰이는 방법으로 '표현 계층 -> 비즈니스 로직 -> 데이터베이스 계층' 과 같은 형태로 주로 구성 된다.

어떤 경우에든 계층을 분리해서 유지하고 각 계층이 자신의 바로 아래 계층에만 의존하게 만드는 것이 목표이지만, 도메인 모델에는 의존성이 전혀 존재하지 않는 것을 목표한다. 즉, 하부 구조와 관련된 문제가 도메인 모델에 영향을 끼쳐서 유연하게 변경할 수 없는 상황이 오는 것을 최대한 피하려는 것이다. 그러기 위해서 '표현 계층 -> 비즈니스 로직 <- 데이터베이스 계층' 과 같은 양파 아키텍처 구조가 필요하다.

### 일반적인 선언적 ORM 방식

```python
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchmey.ext.declarative import declarative_base
from sqlalchemy.orm import relationship

Base = declarative_base()

class Order(Base):
    id = Column(Integer, primary_key=True)

class OrderLine(Base):
    id = Column(Integer, primary_key=True)
    sku = Column(String(250))
    qty = Column(String(250))
    order_id = Column(Integer, ForeignKey('order.id'))
    order = relationship(Order)

class Allocation(Base):
    ...
```

- 장점 
    - `declarative_base` 와 같은 베이스 클래스를 사용해 모델과 데이터베이스 테이블을 한 번에 정의할 수 있어서 간결함
    - 개발자가 데이터베이스와 상호작용 하는 코드를 직접 작성할 필요 없이 ORM이 대신 처리해주므로 생산성 향상
    - 마이그레이션 도구(`alembic`, etc..) 나 스키마 자동 생성 등 다양한 자동화 작업 지원


- 단점
    - ORM에 강하게 결합된 모델은 ORM을 변경하거나, 특정 SQLAlchemy와 같은 라이브러리에서 다른 ORM이나 데이터베이스 접근 방법으로 전환할 때 많은 수정이 필요하기 때문에 유연성 부족
    - 복잡한 비즈니스 로직이 있는 도메인에서는 ORM이 객체 지향 패러다임과 관계형 데이터베이스 패러다임 사이의 임피던스 미스매치를 해결하지 못해, 복잡성이 증가


### 비선언적 ORM 방식

```python
# models.py
from __future__ import annotations
from dataclasses import dataclass
from datetime import date
from typing import Optional, List, Set


class OutOfStock(Exception):
    pass


def allocate(line: OrderLine, batches: List[Batch]) -> str:
    try:
        batch = next(b for b in sorted(batches) if b.can_allocate(line))
        batch.allocate(line)
        return batch.reference
    except StopIteration:
        raise OutOfStock(f"Out of stock for sku {line.sku}")


@dataclass(unsafe_hash=True)
class OrderLine:
    orderid: str
    sku: str
    qty: int


class Batch:
    def __init__(self, ref: str, sku: str, qty: int, eta: Optional[date]):
        self.reference = ref
        self.sku = sku
        self.eta = eta
        self._purchased_quantity = qty
        self._allocations = set()  # type: Set[OrderLine]

    def __repr__(self):
        return f"<Batch {self.reference}>"

    def __eq__(self, other):
        if not isinstance(other, Batch):
            return False
        return other.reference == self.reference

    def __hash__(self):
        return hash(self.reference)

    def __gt__(self, other):
        if self.eta is None:
            return False
        if other.eta is None:
            return True
        return self.eta > other.eta

    def allocate(self, line: OrderLine):
        if self.can_allocate(line):
            self._allocations.add(line)

    def deallocate(self, line: OrderLine):
        if line in self._allocations:
            self._allocations.remove(line)

    @property
    def allocated_quantity(self) -> int:
        return sum(line.qty for line in self._allocations)

    @property
    def available_quantity(self) -> int:
        return self._purchased_quantity - self.allocated_quantity

    def can_allocate(self, line: OrderLine) -> bool:
        return self.sku == line.sku and self.available_quantity >= line.qty
```


```python
# orm.py
from sqlalchemy import Table, MetaData, Column, Integer, String, Date, ForeignKey
from sqlalchemy.orm import mapper, relationship

import model

metadata = MetaData()

# sqlalchemy가 제공하는 추상화를 사용해 데이터베이스 테이블 정의
order_lines = Table(
    "order_lines",
    metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("sku", String(255)),
    Column("qty", Integer, nullable=False),
    Column("orderid", String(255)),
)

batches = Table(
    "batches",
    metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("reference", String(255)),
    Column("sku", String(255)),
    Column("_purchased_quantity", Integer, nullable=False),
    Column("eta", Date, nullable=True),
)

allocations = Table(
    "allocations",
    metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("orderline_id", ForeignKey("order_lines.id")),
    Column("batch_id", ForeignKey("batches.id")),
)

def start_mappers():
    """
    model.OrderLine이 order_lines 테이블과 매핑되고, Batch 클래스가 batches 테이블과 매핑되며, _allocations 속성이 allocations 테이블을 통해 OrderLine과 관계를 맺는 것을 설정
    """
    lines_mapper = mapper(model.OrderLine, order_lines)
    mapper(
        Batch,
        batches,
        properties={
            "_allocations": relationship(
                lines_mapper, secondary=allocations, collection_class=set,
            )
        },
    )
```

- 장점 
    - 비선언적 방식은 데이터베이스와 객체 간의 매핑을 세밀하게 제어할 수 있고, 이 방식은 복잡한 매핑이나 맞춤형 매핑이 필요한 경우 매우 유용(기존 데이터베이스 스키마에 새로운 ORM 적용할 때 등)
    - 객체 모델과 데이터베이스 모델이 분리되어 있어, 도메인 모델을 ORM이나 특정 데이터베이스와 독립적으로 유지할 수 있어 도메인 모델의 순수성을 유지
    - 테이블 구조와 매핑 구조가 독립적으로 관리되므로, 이 두 가지를 별도로 관리 가능. ex) 데이터베이스 구조가 변경되더라도 ORM 매핑 코드에서 최소한의 수정으로 대응 가능


- 단점
    - 선언적 방식에 비해 코드가 복잡함, ORM 방식에 비해 직관적이지 않기 떄문에 이해하는 시간이 필요
    - 매핑을 명시적으로 작성해야하기 때문에, boilerplate 코드 증가
    - 객체와 데이터베이스 테이블 사이의 매핑이 분리되어 있기 때문에, 구조가 변경될 때 이를 동기화 하는 과정이 필요


> 도메인 순수성 : 도메인 모델이 비즈니스 로직에만 집중하고 데이터베이스,UI, 인프라 등의 기술 세부사항으로부터 독립적이어야 한다는 개념으로 DDD에서 강조됨


`start_mappers` 함수를 호출하면 쉽게 도메인 모델을 데이터베이스에 저장하거나 읽어올 수 있다. 이런 방식을 채택하면 아래 예시와 같이 orm 설정에 대한 테스트를 작성하는 것도 가능하다. 

```python
def test_orderline_mapper_can_load_lines(session):
    session.execute(
        "INSERT INTO order_lines (orderid, sku, qty) VALUES "
        '("order1", "RED-CHAIR", 12),'
        '("order1", "RED-TABLE", 13),'
        '("order2", "BLUE-LIPSTICK", 14)'
    )
    expected = [
        model.OrderLine("order1", "RED-CHAIR", 12),
        model.OrderLine("order1", "RED-TABLE", 13),
        model.OrderLine("order2", "BLUE-LIPSTICK", 14),
    ]
    assert session.query(model.OrderLine).all() == expected


def test_orderline_mapper_can_save_lines(session):
    new_line = model.OrderLine("order1", "DECORATIVE-WIDGET", 12)
    session.add(new_line)
    session.commit()

    rows = list(session.execute('SELECT orderid, sku, qty FROM "order_lines"'))
    assert rows == [("order1", "DECORATIVE-WIDGET", 12)]
```

`start_mapper` 함수는 Sqlalchmey의 ORM 매핑을 설정하는 함수로 데이터베이스에 실제 테이블을 생성하지는 않고 단지 python 객체와 데이터베이스 테이블 간의 매핑을 설정해준다.

또한 의존성을 역전하고자 하는 목적은 달성되었고 도메인 모델은 더 이상 인프라에 신경쓰지 않아도 되며 Sqlalchemy를 제거하고 다른 ORM 라이브러리를 사용해도 도메인 모델을 변경할 필요가 없다.


## 테스트에 자주 사용되는 가짜 저장소 생성법

```python
import abc
import model

class AbstractRepository(abc.ABC):
    @abc.abstractmethod
    def add(self, batch: model.Batch):
        raise NotImplementedError

    @abc.abstractmethod
    def get(self, reference) -> model.Batch:
        raise NotImplementedError


class SqlAlchemyRepository(AbstractRepository):
    def __init__(self, session):
        self.session = session

    def add(self, batch):
        self.session.add(batch)

    def get(self, reference):
        return self.session.query(model.Batch).filter_by(reference=reference).one()

    def list(self):
        return self.session.query(model.Batch).all()


class FakeRepository(AbstractRepository):

    def __init__(self, batches):
        self._batches = set(batches)

    def add(self, batch):
        self._batches.add(batch)

    def get(self, reference):
        return next(b for b in self._batches if b.reference == reference)

    def list(self):
        return list(self._batches)
```