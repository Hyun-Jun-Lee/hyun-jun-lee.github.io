---
title: "python generator" #Article title.
date: 2023-06-22
category: [PYTHON, PYTHON] #One, more categories or no at all.
tag: [python]
---

# python ***generator***

제너레이터는 한 번에 하나씩 구성요소를 반환해주는 이터러블을 생성해주는 객체로 주로 메모리를 절약하기 위해 활용한다.

많은 양의 데이터를 메모리에 저장하는 대신 특정 요소를 어떻게 만드는지 아는 객체를 만들어서 필요 할 때 마다 하나씩 만 가져오는 것이다. 

이런 방식은 lazy computation을 통해 객체가 많은 양의 데이터를 다룰 수 있게 해준다.

- 모든 구매 정보를 받이 필요한 지표를 만들어주는 객체

```python
class PurchasesStats:
    def __init__(self, purchases):
        self.purchases = iter(purchases)
        self.min_price: float = None
        self.max_price: float = None
        self._total_purchases_price: float = 0.0
        self._total_purchases = 0
        self._initialize()

    def _initialize(self):
        try:
            first_value = next(self.purchases)
        except StopIteration:
            raise ValueError("no values provided")

        self.min_price = self.max_price = first_value
        self._update_avg(first_value)

    def process(self):
        for purchase_value in self.purchases:
            self._update_min(purchase_value)
            self._update_max(purchase_value)
            self._update_avg(purchase_value)
        return self

    def _update_min(self, new_value: float):
        if new_value < self.min_price:
            self.min_price = new_value

    def _update_max(self, new_value: float):
        if new_value > self.max_price:
            self.max_price = new_value

    @property
    def avg_price(self):
        return self._total_purchases_price / self._total_purchases

    def _update_avg(self, new_value: float):
        self._total_purchases_price += new_value
        self._total_purchases += 1

    def __str__(self):
        return (
            f"{self.__class__.__name__}({self.min_price}, "
            f"{self.max_price}, {self.avg_price})"
        )
```

현재 PurchasesStats는 모든 구매 정보(purchases)를 받아서 필요한 계산을 하고 있다.

여기서 모든 정보를 로드해서 어딘가에 담아서 반환해주는 함수를 추가해보자

```python
def _load_purchases(filename):
    purchases = []
    with open(filename) as f:
        for line in f:
            *_, price_raw = line.partition(",")
            purchases.append(float(price_raw))

    return purchases
```

이 함수는 파일에서 모든 정보를 읽어서 리스트에 저장하는데, 파일이 크기가 클수록 시간이 오래걸리고 메모리가 터질수도 있다.

위 함수를 제너레이터로 만들어본다

결과를 저장하던 리스트도 없고 return 문도 없다.

```python
def load_purchases(filename):
    with open(filename) as f:
        for line in f:
            *_, price_raw = line.partition(",")
            yield float(price_raw)
```

python에서는 어떤 함수라도 `yield` 키워드를 사용하면 제너레이터 함수가 된다.

컴프리헨션으로 정의될 수 있는 리스트,딕셔너리 처럼 제너레이터도 표현식으로 정의할 수 있다. 

```python
>>> [x**2 for x in range(10)]
[0,1,4,9,16,25,36,49,64,81]

>>> (x**2 for x inrange(10))
<generator object <genexpr> at 0x>

>>>sum(x**2 for x in rane(10))
285
```

위와 같이 컴프리헨션 대신에 제너레이터 표현식을 사용해서 min, max, sum 같은 이터러블 연산을 하는 것이 효율적이다.

```python
numbers = [x for x in range(1, 1000001)]
total = sum(numbers)
```

위와 같은 일반적인 리스트는 1부터 1,000,000까지의 숫자를 모두 numbers라는 리스트에 저장하고 메모리 공간을 매우 많이 차지한다.

```python
numbers = iter(range(1,1000001))
total = sum(numbers)
```

이 경우에는 number는 이터레이터로 생성되어 특정 값이 필요할 때만 메모리에 올라간다. 

즉 , sum을 실행하면 이터레이터에서 필요한 값을 하나 씩 가져와서 계산하는 것이다.

## itertools

특정 기준을 넘은 값에 대해서만 연산을 하는 함수를 구현해보자

- 1000건 이상의 구매 이력중 첫 10개 까지만 처리하는 작업

```python
from itertools import islice
purchases = islice(filter(lambda p:p > 1000, purcheases),10)
stats = PurchasesStats(purchases).process()
```

위 코드는 전체 구매이력에서 필터링 한 것처럼 보이지만, 하나 씩 가져와서 PruchasesStats에 전달해주고 있는 것이다.

itertools 라이브러리를 활용하면 다음과 같은 활용도 가능하다

```python
def process_purchases(purchases):
    _min, _max, _avg = itertools.tee(purchases,3)
		return min(_min), max(_max), median(_avg)
```

`itertools.tee` 는 원래의 이터러블을 3개의 새로운 이터러블로 분할해줘서, 굳이 purchases를 3번 반복할 필요 없다.

## 중첩 루프

경우에 따라 1차원 이상을 반복해서 값을 찾아야하는 경우가 있는데, 가장 쉽게 해결하는 방법은 중첩 루프를 사용하는 것이다. 원하는 값을 찾으면 순환을 멈추고 break 를 호출하는데, 이런 경우 두 단계 이상 벗어나야 하므로 정상적으로 동작하지 않는다.

가장 좋은 방법은 중첩을 풀어서 1차원 루프로 만드는 것이다.

- Bad case

```python
def search_nested_bad(array, desired_value):
    """Example of an iteration in a nested loop."""
    coords = None
    for i, row in enumerate(array):
        for j, cell in enumerate(row):
            if cell == desired_value:
                coords = (i, j)
                break

        if coords is not None:
            break

    if coords is None:
        raise ValueError(f"{desired_value} not found")

    logger.info("value %r found at [%i, %i]", desired_value, *coords)
    return coords
```

- Good case

```python
def _iterate_array2d(array2d):
    for i, row in enumerate(array2d):
        for j, cell in enumerate(row):
            yield (i, j), cell

def search_nested(array, desired_value):
    """"Searching in multiple dimensions with a single loop."""
    try:
        coord = next(
            coord
            for (coord, cell) in _iterate_array2d(array)
            if cell == desired_value
        )
    except StopIteration:
        raise ValueError("{desired_value} not found")

    logger.debug("value %r found at [%i, %i]", desired_value, *coord)
    return coord
```

위 코드에서는 나중에 더 많은 차원의 배열을 사용하는 경우에도 클라이언트는 그것에 대해 알 필요가 ㅇ벗이 기존 코드를 사용하면 된다. 이것이 이터레이터 디자인 패턴의 본질이다.

<aside>
💡 <리스트 컴프리헨션 + if-else문>

1. if-else문이 왼쪽에 있는 경우 : if-else문은 각 원소에 대한 값을 결정한다.
`output_list = [x**2 if x > 0 else x for x in input_list]`
양수인 원소는 제곱값으로, 음수인 원소는 원래 값 그대로 유지되도록 리스트 생성

2. if-else문이 오른쪽에 있는 경우: if문은 리스트에 포함될 원소를 필터링하는 역활을 하고, 조건에 True인 원소만 리스트에 포함된다.
`output_list = [x for x in input_list if x >0`
[1,3,5]

</aside>

## python 이터레이터 패턴

이터레이터는 일반적으로 `__iter__` 와 `__next__` 매직 메서드를 구현한 객체이지만, 항상 두가지를 모두 구현할 필요는 없다. 각 매직 메서드를 구현한 이터러블 객체를 비교해보자

### 이터러블 인터페이스

일반적으로 이터러블은 반복할 수 있는 것으로 실제 반복을 할때는 이터레이터를 사용한다.

즉, `__iter__` 매직 매서드는 이터레이터를 반환하고, `__next__` 매직 매서드를 통해 반복 로직을 구현하는 것이다.

- 이터러블하지 않은 이터레이터 객체

```python
class SequenceIterator:

    def __init__(self, start=0, step=1):
        self.current = start
        self.step = step

    def __next__(self):
        value = self.current
        self.current += self.step
        return value
```

```python
    >>> si = SequenceIterator(1, 2)
    >>> next(si)
    1
    >>> next(si)
    3
    >>> next(si)
    5
```

위 코드는 오직 한 번에 하나의 값만 가져올 수 있고, `__iter__` 객체가 없기 때문에 반복할 수는 없다.

- 이터러블한 시퀀스 객체

파이썬이 for 루프를 만나면 객체가 `__iter__` 를 구현했는지 확인하고 있으면 그것을 사용하지만, 없다면 댑 ㅣ옵션을 찾는다.

객체가 `__getitem__` 과 `__len__` 매직 메서드가 구현된 시퀀스인 경우에도 반복 가능하다. 이 경우 인터프리터는 IndexError 예외가 발생할 때 까지 순서대로 값을 제공한다.

## Couroutine

코루틴을 위해 추가된 몇가지 메서드에 대해 알아보고, 이를 이용해 리팩토링 하는 방법에 대해서도 알아보자

- close()

이 메서드를 호출하면 제너레이터에서 GeneratorExit 예외가 발생하고, 이 예외를 따로 처리하지 않으면 제너레이터가 더 이상 값을 생성하지 않으며 반복이 중지된다.

코루틴이 특정 자원을 관리하는 경우, 이 예외를 이용해서 코루틴이 보유한 모든 자원을 해제할 수 있다.

```python
def db_conn(db_handler):
	try:
			while True:
				yield db_handler.read_n_records(10)
	except GeneratorExit:
			db_handler.close()
```

제너레이터를 호출할 때 마다 데이터베이스 핸들러에서 얻은 10개의 레코드를 반환하고, close()를 호출하면 데이터 베이스 연결도 함께 종료한다.

- throw()

이 메서드는 현재 제너레이터가 중단된 위치에서 예외를 던진다. 제너레이터가 예외를 처리했으면 해당 except 절에 있는 코드가 호출되고, 예외를 처리하지 않았으면 예외가 호출자에게 전송된다.

- send(value)

현재 제너레이터의 주요 기능은 고정된 수의 레코드를 읽는 것이다. 이제 읽어올 개수를 파라미터로 받도록 수정해보자

```python
def stream_db_records(db_handler):
		retrieved_data = None
    previous_page_size = 10
    try:
        while True:
            page_size = yield retrieved_data
            if page_size is None:
                page_size = previous_page_size

            previous_page_size = page_size

            retrieved_data = db_handler.read_n_records(page_size)
    except GeneratorExit:
        db_handler.close()
```

send() 메서드는 제너레이터와 코루틴을 구분하는 기준이 된다.

send() 메서드를 사용했다는 것은 yield 키워드가 할당 구문의 오른쪽에 나오게 되고, 인자 값을 받아서 다른 곳에 할당할 수 있음을 뜻한다.

`receive = yield produced`

이 경우 yield 키워드는 두 가지 작업을 한다. 

하나는 produced 값을 호출자에게 보내고 그곳에 멈추는 것이다. 호출자는 next() 메서드를 호출하여 다음 라운드가 되었을 때 값을 가져올 수 있다.

다른 하나는 거꾸로 호출자부터 send() 메서드를 통해 전달된 produced 값을 받는 것으로, 이렇게 입력된 값은 receive 변수에 할당 된다

코루틴에 값을 전달하려면 yield 구문이 멈춘 상태에서만 가능하고, 그렇게 되기 위해서는 next()를 최소 한번은 실행시켜야 한다.

다시 예제로 돌아가서 파라미터를 받도록 수정하자

```python
def stream_db_records(db_handler):
    retrieved_data = None
    page_size = 10
    try:
        while True:
						# yield 문의 ()은 해당 문장이 함수 호출이고 page_size와 비교할 것임을 명확히함
            page_size = (yield retrieved_data) or page_size
            retrieved_data = db_handler.read_n_records(page_size)
    except GeneratorExit:
        db_handler.close()
```

제너레이터에서 처음 next()를 호출하면 yield를 포함하는 위치까지 이동하고 현재 상태의 변수 값을 반환 후 그 위치에서 멈춘다. 변수의 초기 값이 None이기 때문에 처음 next()를 반환하면 None을 반환한다. 

여기서 그냥 next()를 호출하면 기본 값인 10으로 page_size 가 정해지고 작업이 진행된다. next()를 호출하는 것은 send(None)과 같기 때문에 기본 값으로 사용하도록 설정된다.

하지만, send(value)를 통해 명시적인 값을 제공하면 yield문의 반환 값으로 page_size 변수에 설정된다. 이제 사용자가 지정한 값이 page_size로 설정된다. 

```python
streamer = stream_db_records(db_handler)
new_page_size = 20

streamer.send(new_page_size)
TypeError : can't send non-None value to a just-started generator
```

앞서한 설명에 따라 send()를 호출하기 전에 next()를 한번은 실행시켜야 한다.

이런 것을 신경쓰지 않고 코루틴을 생성하자 마자 바로 사용할 수 있는 방법이 있다.

```python
@prepare_coroutine
def auto_stream_db_records(db_handler):
    """This coroutine is automatically advanced so it doesn't need the first
    next() call.
    """
    retrieved_data = None
    page_size = 10
    try:
        while True:
            page_size = (yield retrieved_data) or page_size
            retrieved_data = db_handler.read_n_records(page_size)
    except GeneratorExit:
        db_handler.close()
```

`@prepare_coroutine` 라는 데코레이터를 사용하면 코루티을 바로 사용할 수 있다.