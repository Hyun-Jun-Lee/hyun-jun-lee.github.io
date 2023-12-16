---
title: "dataclass" #Article title.
date: 2023-05-18
category: [PYTHON, PYTHON] #One, more categories or no at all.
tag: [python]
---

python 에서 데이터를 담아두기 위해 list, tuple, dict, set 등과 같은 내장 자료구조를 활용하는 방법도 있지만, class를 이용해서 데이터를 담아두면 type-safe 해져서 오류가 발생할 확률이 적어진다

# 기존 방식

```python
from datetime import date

class User:
    def __init__(
        self, id: int, name: str, 
				birthdate: date,
				admin: bool = False
		    ) -> None:
        self.id = id
        self.name = name
        self.birthdate = birthdate
        self.admin = admin
	
```

위 코드를 보면 id, name, birthdate, admin 변수가 반복되어 사용되고 이런 것을 `boiler-plate` 라고 한다. 

위에서는 필드가 3개 뿐이지만, 만약 필드 개수가 많은 클래스라면 매우 귀찮고 타이핑 에러가 날 확률이 높다

그리고 위 클래스의 인스터스를 출력해 보려면 `__repr__` 메서드를 추가해줘야하고, 필드 값이 똑같은 두 개의 인스턴스를 동등한 객체라고 설정하고 싶을 땐 `__eq__` 메서드도 추가해줘야한다

```python
>>> user = User(id=1, name="Steve Jobs", birthdate=date(1955, 2, 24))
>>> user
<__main__.User object at 0x105558100>

>>> user1 = User(id=1, name="Steve Jobs", birthdate=date(1955, 2, 24))
>>> user2 = User(id=1, name="Steve Jobs", birthdate=date(1955, 2, 24))
>>> user1 == user2
False
```

- `__repr__` ,`__eq__` 추가

```python
from datetime import date

class User:
    def __init__(
        self, id: int, name: str, birthdate: date, admin: bool = False
    ) -> None:
        self.id = id
        self.name = name
        self.birthdate = birthdate
        self.admin = admin

    def __repr__(self):
        return (
            self.__class__.__qualname__ + f"(id={self.id!r}, name={self.name!r}, "
            f"birthdate={self.birthdate!r}, admin={self.admin!r})"
        )

		def __eq__(self, other):
        if other.__class__ is self.__class__:
            return (self.id, self.name, self.birthdate, self.admin) == (
                other.id,
                other.name,
                other.birthdate,
                other.admin,
            )
	        return NotImplemented
```

```python
>>> user1 = User(id=1, name="Steve Jobs", birthdate=date(1955, 2, 24))
>>> user1
User(id=1, name='Steve Jobs', birthdate=datetime.date(1955, 2, 24), admin=False)

>>> user1 = User(id=1, name="Steve Jobs", birthdate=date(1955, 2, 24))
>>> user2 = User(id=1, name="Steve Jobs", birthdate=date(1955, 2, 24))
>>> user1 == user2
True
```

# 데이터 클래스 활용

```python
from dataclasses import dataclass
from datetime import date

@dataclass
class User:
    id: int
    name: str
    birthdate: date
    admin: bool = False
```

`@dataclass` 데코레이터를 일반 클래스에 선언해주면 해당 클래스는 데이터 클래스가 되고 `__init__` , `__repr__` , `__eq__` 와 같은 메서드를 자동으로 생성해준다.

```python
>>> user1 = User(id=1, name="Steve Jobs", birthdate=date(1955, 2, 24))
>>> user1
User(id=1, name='Steve Jobs', birthdate=datetime.date(1955, 2, 24), admin=False)
>>> user2 = User(id=1, name="Steve Jobs", birthdate=date(1955, 2, 24))
>>> user1 == user2
True
```

## 불변 데이터 만들기

데이터 클래스는 담고 있는 데이터를 자유롭게 변경할 수 있는데, 만약 변경할수 없도록 만드려면 `frozen` 옵션을 사용해야 한다.

```python
from dataclasses import dataclass
from datetime import date

@dataclass(frozen=True)
class User:
    id: int
    name: str
    birthdate: date
    admin: bool = False
```

## 데이터 대소비교 및 정렬

필드값에 따라 데이터 대소비교가 필요하다면 `order` 옵션을 사용할 수 있다.

비교되는 두 인스턴스는 같은 형태여야한다

```python
from dataclasses import dataclass
from datetime import date

@dataclass(order=True)
class User:
    id: int
    name: str
    birthdate: date
    admin: bool = False
```

```python
>>> user1 = User(id=1, name="Steve Jobs", birthdate=date(1955, 2, 24))
>>> user2 = User(id=2, name="Bill Gates", birthdate=date(1955, 10, 28))
>>> user1 < user2
True
>>> user1 > user2
False
>>> sorted([user2, user1])
[User(id=1, name='Steve Jobs', birthdate=datetime.date(1955, 2, 24), admin=False), User(id=2, name='Bill Gates', birthdate=datetime.date(1955, 10, 28), admin=False)]
```

## 세트나 사전에서 사용하기

데이터 클래스의 인스턴스는 unhasable 하기 때문에 set,dict의 key로 사용할 수 없고 `unsafe_hash` 옵션을 사용해야한다

```python
from dataclasses import dataclass
from datetime import date

@dataclass(unsafe_hash=True)
class User:
    id: int
    name: str
    birthdate: date
    admin: bool = False
```

```python
>>> user1 = User(id=1, name="Steve Jobs", birthdate=date(1955, 2, 24))
>>> user2 = User(id=2, name="Bill Gates", birthdate=date(1955, 10, 28))
>>> user3 = User(id=1, name="Steve Jobs", birthdate=date(1955, 2, 24))
>>> user4 = User(id=2, name="Bill Gates", birthdate=date(1955, 10, 28))
>>> set([user1, user2, user3, user4])
{User(id=2, name='Bill Gates', birthdate=datetime.date(1955, 10, 28), admin=False), User(id=1, name='Steve Jobs', birthdate=datetime.date(1955, 2, 24), admin=False)}
```

## 가변 데이터를 필드의 디폴트 값에 부여

필드의 기본 값은 인스턴스 간에 공유가 되기 때문에 기본 값 할당이 허용되지 않는다. 

그래서 데이터 클래스에서 제공하는 `field(default_factory=list)` 를 사용해서 매번 새로운 값이 생성되도록 해줘야 한다.

```python
from dataclasses import dataclass, field
from datetime import date
from typing import List

@dataclass(unsafe_hash=True)
class User:
    id: int
    name: str
    birthdate: date
    admin: bool = False
    friends: List[int] = field(default_factory=list)
```

```python
>>> user1 = User(id=1, name="Steve Jobs", birthdate=date(1955, 2, 24))
>>> user1.friends
[]
>>> user1.friends.append(2)
>>> user1.friends
[2]
```