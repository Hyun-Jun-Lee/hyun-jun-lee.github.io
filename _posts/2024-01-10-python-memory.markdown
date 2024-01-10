---
title: "Pyhon은 어떻게 메모리를 관리하고 있을까?"
date: 2024-01-10
category: [PYTHON]
tag: [python, memory, cpython]
---

일 하던 중 메모리 부족 문제가 있었는데, 이를 해결 하던 중 내가 주로 쓰는 언어인 python은 어떻게 메모리를 관리하는지에 대해 내부 작동 구조가 궁금해졌다. 

예전에 취직 전에 공부한 적이 있는 것 같은데, 역시 이론 공부는 내가 실전에서 당해봤을 때 가장 잘 깨닫는 것 같다.

## Pyhon은 어떻게 메모리를 관리하고 있을까?

### python 작동 방식

python은 인터프리터 언어로 알고 있어서 컴파일 하지 않고 한줄 씩 읽어서 실행한다고만 알고 있었다. 여기서 좀 더 자세히 보자면 python은 코드를 0,1 코드로 변환시키는 컴파일링과 한줄 씩 읽으며 해석하는 인터프리터가 혼합되어 있다.(기본 제공 인터프리터는 Cpython)

우리가 python 코드를 작성하고 실행시키면 해당 코드는 컴퓨터가 이해할 수 있는 언어인 0,1 로 변환되고, 이 것을 complie이라고 하고  이 때 소요되는 시간을 compile time 이라고 한다. 즉, python은 순수한 인터프리터 언어가 아니라 컴파일 과정이 있다. 이 과정에서 파이썬 코드는 `.pyc` 파일로 저장되는 중간 형태의 바이트코드로 변환된다.
이 바이트코드는 기계어와 달리 PVM이 해석할 수 있는 중간 수준의 코드이다.

컴파일된 바이트 코드는 Python Virtual Machine, PVM에 의해 해석되고 실행되는데 이 때 PVM은 바이트코드를 한 줄씩 읽고 해당 명령을 수행한다. 이 과정이 인터프린팅 단계이고, 이 때 실행환경을 runtime이라고 부른다. 

pthon이 다른 컴파일 언어 C, Go, Java 등이 비해 실행 속도가 느린 이유가 위 과정 처럼, 바이트코드를 해석하는 과정에서 추가적인 시간이 소요되기 때문이다.

### Cpython memory 관리

- 정적 메모리 할당
    - 컴파일 되는 시점에 필요한 메모리의 양이 결정되고 할당되는 것으로, 프로그램 실행 전에 모든 메모리 크기가 결정된다.
    - stack 구조로 구현되어 가장 최근에 할당된 메모리가 제일 먼저 해제된다.
    - 메모리 크기가 고정되어 있어서 관리가편하지만 유연성이 떨어진다.
- 동적 메모리 할당
    - 프로그램이 실행되는 동안 필요할 때 메모리를 할당하고 해제하는 방식
    - Heap 메모리 영역에 할당
        - Heap 메모리 영역 : heap 메모리 영역은 runtime에 사용자가 필요에 따라 메모리를 할당하고 해제할 수 있는, '자유 메모리 블록' 의 집합.
    - 개발자가 직접 할당, 해제 관리

python은 정적 메모리에 정의된 함수, 변수의 이름이 들어가고 동적 메모리 영역에는 모든 객체가 할당된다. 

- 해당 과정 예시 : https://yomangstartup.tistory.com/105

pyhon은 객체가 생성될 때 마다 그 객체를 잠조하는 변수의 개수를 추적하는 방식으로 메모리 관리를 한다. 예를 들어, 객체 A를 참조하는 변수의 개수가 0이라면, 해당 객체의 메모리는 자동으로 해제된다.

```python
class Node:
    def __init__(self, value):
        self.value = val
        self.next = None

node1 = Node(1)
node2 = Node(2)
node1.next = node2
node2.next = node1
```

이런 방식은 위 예시 코드와 같이 node1 과 node2가 서로를 참조하고 있을 때, node1 객체가 더 이상 사용되지 않더라도 node2 객체가 참조하고 있기 때문에 메모리 누수가 발생할 수 있다.

python은 이런 문제를 해결하기 위해 `Garbage collection(가비지 컬렉션)` 기능을 제공한다. 

### python garbage colletion

#### Reference Counting

앞서 객체를 참조하는 변수의 개수를 추적하는 방식으로 메모리를 관리한다고 햇는데 이를 Reference Counting 방식이라고 한다. 

파이썬에서 객체를 만들면 type(list, dict 등)과 reference count가 생성되고, 해당 객체가 참조 될 때 마다 reference count가 증가하고 참조 해제될 때 마다 감소되며 0이 되면 메모리가 해제된다. 

```python
import sys

a = 'hi'
sys.getrefcount(a)

2
```

위 예시를 보면 a이 reference count가 2라고 나오는데 왜냐하면 변수 'a'를 생성할 때 +1, 해당 변수 'a'를 sys.getrefcount()에 전달할 때 +1 되었기 때문이다.

#### gc module

파이썬은 개발자가 호출하지 않아도 자동으로 가비지 컬렉션을 수행하는데, 수동으로 실행하기 위해서는 아래와 같이 하면 된다.

```python
import gc

class Node:
    def __init__(self, value):
        self.value = val
        self.next = None

node1 = Node(1)
node2 = Node(2)
node1.next = node2
node2.next = node1

# before
print(len(gc.get_objects()))

gc.collect()

# after
print(len(gc.get_objects()))
```

python이 자동으로 가비지 컬렉터를 실행하기 때문에, 수동으로 하는 것이 가장 최적화가 잘 될 듯하다.
특히 비동기 작업을 많이 하는 환경에서 제대로 메모리 관리를 하기 위해서는 개발자가 직접 관리해주는 것이 필수일 듯하다.

보통 아래 2가지 방법으로 수동 가비지 컬렉션을 수행한다.

- Time-based : 가비지 컬렉터를 고정된 시간 간격으로 호출
- Event-based : 이벤트 발생 시 가비지 컬렉터호출

수동으로 가비지 컬렉션을 실행시키는 것은 생각보다 주의해야할 사항이 많으므로 차라리 물리 자원을 확장시키는 것을 추천한다는 reference가 많더라
관련해서 찾다보니 인스타그램에서 파이썬 가비지 컬렉터 비활성화 시도한 글을 찾앗다.

https://luavis.me/python/dismissing-python-garbage-collection-at-instagram


### reference
- https://medium.com/dmsfordsm/garbage-collection-in-python-777916fd3189
- https://medium.com/dmsfordsm/garbage-collection-in-python-777916fd3189