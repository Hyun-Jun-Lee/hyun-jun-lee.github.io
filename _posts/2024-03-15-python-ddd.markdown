---
title: "Python Domain Driven Design" #Article title.
date: 2024-03-16
category: [PYTHON, ]
tag: [pycorn, DDD, Architecture,python]
---

__PyCon korea 2023 신동현__

 > 해당 스피치 영상 시청 후 추가로 자료 조사해서 내용 추가

# Python Domain Driven Design

- 호텔 예약 시스템 예시로 Python을 통해 DDD 구현

## Domain Driven Design

일반적으로 많이 사용하는 데이터 중심의 접근법을 탈피해서 순수한 도메인의 모델과 로직에 집중하는 것으로 ,소프트웨어가 해결하는 문제(domain)에 집중하는 개발 방법론(ex. 온라인 쇼핑 시스템 에서는 주문, 결제 등)이다. 

모듈간의 의존성은 최소화하고 응집선을 최대화 하려는 아키텍처.


## 데이터 중심 개발과의 비교

지금까지 내가 개발해온 방식이 데이터 중심 개발이기 때문에 비교

### 데이터 중심 개발
DB 설계에 초점을 맞추는 개발 방식으로, DB 모델링이 어플리케이션 구조와 기능을 결정 짓는 핵심요소


- 장점
    - DB 모델링 설계가 핵심이기 때문에 데이터 일관성 및 무결성 보장에 유리
    - 데이터 모델이 중심이기 때문에 데이터 관리 효율적
    - DB 성능 최적화


- 단점
    - 비즈니스 로직을 DB 구조에 맞춰 구현해야하는 경우가 생겨, 비즈니스 요구사항과 불일치 발생 할수도 있음
    - 비즈니스 요구사항의 변화에 따라 DB 스키마를 변경해야하는데, 이것에 비용이 많이듬
    - 비개발직군과 같이 DB 설계를 잘 모르는 팀원과의 의사소통이 힘듬


### 도메인 중심 설계
핵심적인 비즈니스 도메인을 중심으로 설계, 비즈니스 로직과 비즈니스가 직면한 문제를 해결하는 것을 최우선으로 여기고, 이를 기반으로 개발

- 장점
    - 비즈니스 로직과 비즈니스 문제 해결을 중심으로 설계하기 때문에 요구사항을 명확하게 반영 가능
    - Bounded Context를 통해 시스템을 모듈화하면, 비즈니스 요구사항의 변화에 따라 유연한 대응 및 확장성 향상
    - 유비쿼터스 언어라는 보편적인 언어를 사용해서 비개발자와의 소통 용이


- 단점
    - DDD를 효과적으로 적용하기 위해선 어느 정도 학습의 시간이 필요
    - 복잡한 도메인에서는 세밀한 모델링과 설계가 요구되어 프로젝트 초기에 많은 시간이 소모
    - 소규모 프로젝트에서는 오버 엔지니어링이 될 수도


## DDD Layers

### Domain Layer
시스템의 중심이 되는 도메인 비즈니스 로직을 다루는 계층이고, 기술적인 제약을 받지 않아야 하기에 순수 Python으로 구현하는 것을 추천

- Entity : 고유의 식별자를 가지는 객체(상품,고객 등)
- Value Object(VO) : 도메인 내의 값이나 성질을 표현하는 객체(가격, 위치 등)
- Aggregate : 단일 객체로 취급되는 entity와 value object가 묶여진 하나의 객체(주문 = 고객+상품+금액)
- Aggregate Root : Aggregate에 접근하거나 조작하기 위한 entry point Entity
- Exception : 도메인 내에서 발생하는 예외상황,에러
- Service : 다수의 Entitiy나 VO에 의한 동작,연산 모음

### Infrastructure Layer
기술적 인프라 제공 및 에플리케이션을 도와주기 위한 계층으로, 도메인과 의존성이 생겨서는 안됨.

- Database
- Caching
- Logging
- 외부 API

### Application Layer
도메인 비즈니스 로직과 인프라 계층을 활용해서 Use Case를 구현하는 계층

- Use Case : 시스템이 제공하는 특정 기능(회원가입, 조회, 구매 등)들을 유저의 관점에서 기능적인 요구사항을 구현한 메서드
- Query : 데이터를 읽어오는 동작
- Command : Create, Update, Delete와 같이 상태를 변경하는 동작
- CQRS : Command Query Responsibility Segregation, Read(Query)와 Write(Command)의 권한과 책임을 분리하는 패턴

### Presentation Layer
유저 인터페이스 제공(Rest API, GraphQL 등) 및 input/output validation/serialization 등을 담당하는 계층


## DDD : Bounded Context

특정한 도메인 모델이 적용될 수 있는 영역.

![image](https://github.com/qu3vipon/python-ddd/assets/76996686/69468fa4-8181-4b41-852b-205defe7fc78)

위 예시 이미지에서 보이듯이, `Display context`와 `Reception Context`에서 같은 Room 모델이지만 Bounded Context가 달라지면 서로 다른 의미를 가지게 된다.


## DDD Components example

### Entity: Definition

```python
from dataclasses import dataclass, field


class Entity:
    # entity의 고유 식별자
    # init=False 옵션을 설정해 객체가 생성될 때 id 값이 자동할당 되지 않고
    # 클래스의 인스턴스가 다른 방식(예: 데이터베이스에서 엔티티를 로드하는 과정 중)으로 초기화되거나 명시적으로 값을 설정한 후에 값을 가지게 된다.
    id: int = field(init=False)
  
    # 두 객체가 동일한 객체인지 판단할 때 id 필드를 사용하도록 정의
    def __eq__(self, other: Any) -> bool:
        if isinstance(other, type(self)):
            return self.id == other.id
        return False
    
    # __eq__ 매직 메서드를 구현한 경우, 일간된 해시 값을 제공해야 하기 때문에 반드시 구현해야함.
    def __hash__(self):
        return hash(self.id)


class AggregateRoot(Entity):
    pass

# eq 메서드를 집적 오버라이드하여 사용할 것이기 때문에 eq=False
# 불필요한 메모리 낭비 줄이기 위해 slots=True
@dataclass(eq=False, slots=True)
class Reservation(AggregateRoot):
    room: Room
    reservation_number: ReservationNumber
    reservation_status: ReservationStatus
    date_in: datetime
    date_out: datetime
    guest: Guest
```

```python
# Display Context에 정의된 Room
@dataclass(eq=False, slots=True)
class Room(AggregateRoot):
    number : str
    room_status : RoomStatus
    image_url : str
    description : str | None = None

# Reception Context에 정의된 Room
@dataclass(eq=False, slots=True)
class Room(Entity):
    number : str
    room_status : RoomStatus

    def reserve(self):
        if not self.room_status.is_avaliable:
            raise RoomStatusException

        self.room_status = RoomStatus.RESERVED
```

Display Context의 Room 모델은 테이블의 모든 데이터가 매핑되어 있지만, Reception Context의 Room 모델은 예약에 필요한 데이터만 매핑되어 있다.

### Entity: Life Cycle

```python
@dataclass(eq=False, slots=True)
class Reservation(AggregateRoot):
    # ...

    @classmethod
    def make(
        cls, room: Room, date_in: datetime, date_out: datetime, guest: Guest
    ) -> Reservation
        # reserve()가 성공하면, Reservation 객체 생성
        room.reserve()
        return cls(
            room=room,
            date_in=date_in,
            date_out=date_out,
            guest=guest,
            reservation_number=ReservationNumber.generate(),
            reservation_status=ReservationStatus.IN_PROGRESS,
        )

    def cancel(self):
        if not self.reservation_status.in_progress:
            raise ReservationStatusException
  
        self.reservation_status = ReservationStatus.CANCELLED
  
    def check_in(self):
        # ...
  
    def check_out(self):
        # ...
  
    def change_guest(self, guest: Guest):
```

위 코드처럼 Reservaion를 생성하거나 상태를 변경하는 기능을 메소드화 하여 Entity에 정의해두면 코드 응집력을 향상시킬 수 있고, 각 Entity가 어떤 life cycle을 겪으면서 시스템에 사용되는지 파악하기 쉽다.

### Entity: Table Mapping

```python
from sqlalchemy import MetaData, Table, Column, Integer, String, Text, ForeignKey, DateTime
from sqlalchemy.orm import registry

metadata = MetaData()
mapper_registry = registry()

# Table 정의
room_table = Table(
  "hotel_room",
  metadata,
  Column("id", Integer, primary_key=True, autoincrement=True),
  Column("number", String(20), nullable=False),
  Column("status", String(20), nullable=False),
  Column("image_url", String(200), nullable=False),
  Column("description", Text, nullable=True),
  UniqueConstraint("number", name="uix_hotel_room_number"),
)

reservation_table = Table(
  "room_reservation",
  metadata,
  Column("id", Integer, primary_key=True, autoincrement=True),
  Column("room_id", Integer, ForeignKey("hotel_room.id"), nullable=False),
  Column("number", String(20), nullable=False),
  Column("status", String(20), nullable=False),
  Column("date_in", DateTime(timezone=True)),
  Column("date_out", DateTime(timezone=True)),
  Column("guest_mobile", String(20), nullable=False),
  Column("guest_name", String(50), nullable=True),
)

# Table - Entity 매핑
def init_orm_mappers():
    from reception.domain.entity.room import Room as ReceptionRoomEntity
    from reception.domain.entity.reservation import Reservation as ReceptionReservationEntity

    # ReceptionRoomEntity 클래스는 room_table 데이터베이스 테이블과 매핑
    # properties 매개변수를 통해 room_status 속성이 설정
    mapper_registry.map_imperatively(
    ReceptionRoomEntity,
    room_table,
    properties={
        "room_status": composite(RoomStatus.from_value, room_table.c.status),
    }
    )

    # ReceptionReservationEntity 클래스는 reservation_table 데이터베이스 테이블과 매핑
    mapper_registry.map_imperatively(
    ReceptionReservationEntity,
    reservation_table,
    properties={
        # Room 엔티티와의 관계를 정의하며, reservations로의 역참조(backref), 정렬(order_by), 그리고 lazy="joined" 옵션을 통한 조인 로딩 전략이 설정
        "room": relationship(
        Room, backref="reservations", order_by=reservation_table.c.id.desc, lazy="joined"
        ),
        "reservation_number": composite(ReservationNumber.from_value, reservation_table.c.number),
        "reservation_status": composite(ReservationStatus.from_value, reservation_table.c.status),
        # guest_mobile, guest_name이라는 2개의 컬럼을 'Guest'라는 하나의 VO로 병합해서 가져옴
        "guest": composite(Guest, reservation_table.c.guest_mobile, reservation_table.c.guest_name),
    }
    )

    from display.domain.entity.room import Room as DisplayRoomEntity

    mapper_registry.map_imperatively(
    DisplayRoomEntity,
    room_table,
    properties={
        "room_status": composite(RoomStatus.from_value, room_table.c.status),
    }
    )
```
위 매핑은 sqlalchemy가 자동으로 해주는 것이 아니라, `from_value` 같은 메소드를 만들어서 매핑함. 이것은 VO 단에서 자세히 설명.

### Value Object(VO)

```python
from pydantic import constr

mobile_type = constr(regex=r"\+[0-9]{2,3}-[0-9]{2}-[0-9]{4}-[0-9]{4}")

# instance의 동등성 비교하지 않고, 주로 값을 비교하기 때문에 dataclass의 default eq 메서드 사용
@dataclass(slots=True)
class Guest(ValueObject):
    mobile: mobile_type
    name: str | None = None

class ValueObject:
    # sqlalchmey의 composite() 메서드를 사용하기 위한 함수
    def __composite_values__(self):
        return self.value,
  
    @classmethod
    def from_value(cls, value: Any) -> ValueObjectType | None:
        # Enum객체도 처리할 수 있도록 구현해둠
        if isinstance(cls, EnumMeta):
            for item in cls:
                if item.value == value:
                    return item
            raise ValueObjectEnumError
        
        instance = cls(value=value)
        return instance


@dataclass(slots=True)
class Guest(ValueObject):
    mobile: mobile_type
    name: str | None = None

    def __composite_values__(self):
        return self.mobile, self.name
```

Sqlalchemly에서 `composite()` 메서드는 여러 개의 데이터베이스 컬럼을 하나의 Python 객체로 매핑할 때 사용되고, 이를 위해 SQLAlchemy는 해당 객체가 어떤 컬럼들로 구성되어 있는지 알아야 하며, `__composite_values__` 메서드는 이 정보를 SQLAlchemy에 제공한다.

`__composite_values__` 메서드는 객체가 데이터베이스의 여러 컬럼에 매핑될 때, 그 컬럼들의 값을 순서대로 나열한 튜플을 반환해야 하고, 이 메서드는 SQLAlchemy에 의해 호출되며, 반환된 값은 데이터베이스에 저장되거나 쿼리에 사용된다.

### Repository

Infra structure 계층에서 기술적인 구현을 추상화 하는 방법 중 가장 쉬운 방법으로 Repository 패턴 사용

- Repository 패턴 : repository라고 불리는 별도의 class를 생성해서 데이터 처리에 대한 권한을 위임하고 구체적인 기술적 구현은 repository 내에서 처리하게 만드는 것

```python
class RDBRepository:
    @staticmethod
    def add(session, instance: EntityType):
        return session.add(instance)

    @staticmethod
    def commit(session):
        return session.commit()

class ReservationRDBRepository(RDBRepository):
    @staticmethod
    def get_reservation_by_reservation_number(
        session: Session, reservation_number: ReservationNumber
    ) -> Reservation | None:
        return session.query(Reservation).filter_by(reservation_number=reservation_number).first()
```

### Use Case(CQRS)

app 계층에서 사용되는 Use Case는 domain, infra 계층을 적절히 조합해서 요구사항에 맞는 기능을 구현한다.

```python
# 예약 번호로 예약을 조회하는 기능
class ReservationQueryUseCase:
    def __init__(
        self,
        reservation_repo: ReservationRDBRepository,
        db_session: Callable[[], ContextManager[Session]],
    ):
        self.reservation_repo = reservation_repo
        self.db_session = db_session

    def get_reservation(self, reservation_number: str) -> Reservation:
        # str 로 입력받은 예약번호를 VO로 변환해줌
        reservation_number = ReservationNumber.from_value(value=reservation_number)

        with self.db_session() as session:
            # repository 활용해서 예약 번호로 조회하고, entity나 None을 반환받음
            reservation: Reservation | None = (
                self.reservation_repo.get_reservation_by_reservation_number(
                    session=session, reservation_number=reservation_number
                )
            )

        if not reservation:
            raise ReservationNotFoundException

        return reservation
```

### Presentation 계층

- Request
```python
class CreateReservationRequest(BaseModel):
    room_number: str
    date_in: datetime
    date_out: datetime
    guest_mobile: mobile_type
    guest_name: str | None = None
```


- Response
```python
class ReservationSchema(BaseModel):
    room: RoomSchema
    reservation_number: str
    status: ReservationStatus
    date_in: datetime
    date_out: datetime
    guest: GuestSchema

    @classmethod
    def build(cls, reservation: Reservation) -> ReservationSchema:
        return cls(
            room=RoomSchema.from_entity(reservation.room),
            reservation_number=reservation.reservation_number.value,
            status=reservation.reservation_status,
            date_in=reservation.date_in,
            date_out=reservation.date_out,
            guest=GuestSchema.from_entity(reservation.guest),
        )

class ReservationResponse(BaseResponse):
    result: ReservationSchema
```

entity를 바로 반환하지 않고 build() 라는 메서드로 다수의 entity를 조합해 반환하고 아래와 같은 과정으로 동작함

1. Reservation 엔터티의 각 속성을 접근하여, ReservationSchema가 요구하는 형태와 타입으로 변환한다. 예를 들어, reservation.room은 RoomSchema.from_entity() 메서드를 통해 RoomSchema 객체로 변환되고 reservation.guest도 GuestSchema.from_entity() 메서드를 통해 GuestSchema 객체로 변환된다.
2. 변환된 데이터를 사용하여 ReservationSchema의 인스턴스를 생성한다.
3. 생성된 ReservationSchema 인스턴스를 반환한다.


- API
```python
@router.get("/reservations/{reservation_number}")
@inject
def get_reservation(
    reservation_number: str,
    reservation_query: ReservationQueryUseCase = Depends(
      Provide[AppContainer.reception.reservation_query]
    ),
):
    try:
      reservation: Reservation = reservation_query.get_reservation(
        reservation_number=reservation_number
      )
    except ReservationNotFoundException as e:
      raise HTTPException(
        status_code=status.HTTP_404_NOT_FOUND,
        detail=e.message,
      )
    return ReservationResponse(
      detail="ok",
      result=ReservationSchema.build(reservation=reservation),
  )
```

실제 로직은 Use Case에 모두 구현되어 있음.



## Reference 
    - https://github.com/qu3vipon/python-ddd/

    - https://blog.doctor-cha.com/introduction-to-domain-driven-design-for-everyone