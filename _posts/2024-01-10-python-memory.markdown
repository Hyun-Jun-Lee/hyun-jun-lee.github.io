---
title: "python memory"
date: 2024-01-10
category: [PYTHON]
tag: [python, memory]
---

일 하던 중 메모리 부족 문제가 있었는데, 이를 해결 하던 중 내가 주로 쓰는 언어인 python은 어떻게 메모리를 관리하는지에 대해 내부 작동 구조가 궁금해졌다. 

예전에 취직 전에 공부한 적이 있는 것 같은데, 역시 이론 공부는 내가 실전에서 당해봤을 때 가장 잘 깨닫는 것 같다.

## Pyhon은 어떻게 메모리를 관리하고 있을까?

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





### reference
- https://medium.com/dmsfordsm/garbage-collection-in-python-777916fd3189
- https://medium.com/dmsfordsm/garbage-collection-in-python-777916fd3189