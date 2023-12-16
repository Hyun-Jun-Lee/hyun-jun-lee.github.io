---
title: "Django ORM 구조와 원리 최적화 전략" #Article title.
date: 2023-07-08
category: [PYTHON, DJANGO] #One, more categories or no at all.
tag: [pycorn, django, python]
---

__PyCon korea 2020 김성렬__


# Django ORM 구조와 원리 최적화 전략

## QuerySet을 통해 알아보는 ORM 특징

### 1. Lazy Loading

정말 필요한 시점에 SQL을 호출함

QuerySet을 정의하고 사용하지 않으면 실제 SQL은 호출되지 않음

```sql
# 이 시점에는 ORM 정의만 되어있음
users : QuerySet = User.objects.all()
# 실제 SQL이 실행되는 시점은 list()로 묶은 시점
user_list = list(users)
```

이런 ORM의 특성 때문에 아래와 같은 사용은 매우 비효율적

```sql
# 불필요하게 SQL을 두번 호출하게 되는 사례
users: QuerySet = User.objects.all()

## 여기서 ORM은 LIMIT 1 을 포함한 SQL을 호출
first_user: User = users[0]

## 이전에 호출한 SQL을 무시하고 새롭게 다시 호출해오게 됨
user_list: list(User) = list(users)
```

### 2. QuerySet Caching

호출하는 순서만 바꿔도 SQL을 더 적게 호출 가능

```sql
# 위 사례에서 순서만 바꾼 Case
users: QuerySet = User.objects.all()

## 여기서 모든 user를 가져오는 sql이 호출되었고, 모든 user 데이터가 캐싱되어있는 상태
user_list: list(User) = list(users)

## SQL을 다시 호출하지 않고 users에 캐싱된 값을 활용
first_user: User = users[0]
```

### 3. Eager Loading

SQL로 한번에 많은 데이터를 불러와야할 때 선택하는 방법으로 `select_related` , `prefetch_related` 를 지원함

```sql
# User - Userinfo 1:1 관계

## 아직 SQL이 선언되지 않은 상태
users: QuerySet = User.objects.all()  

## LazyLoading 특성 때문에 모든 User의 정보를 호출해 왔지만
## Userinfo는 당장 필요하지 않기 때문에 호출해오지 않은 상태
for user in users:
		## for문 마다 Userinfo 정보를 호출해오기 위해 SQL 실행됨
    user.userinfo
```

위와 같이 SQL이 n번 호출 될때, n+1개의 SQL이 호출되게 되는 상황을 `n+1 problem` 이라고 함

# QuerySet 상세

## QuerySet 구성 요소

- QuerySet은 한 개의 Query와 0 ~ N개의 추가 Query로 구성되어 있다.

```sql
from django.db.model.sql import Query  

class QuerySet:
  ## Main Query
	query: Query = Query()  
	
	## SQL의 수행 결과 저장 및 재사용(QuerySet Cache)
	## QuerySet 호출할 때 해당 프로퍼티 데이터가 없으면 SQL을 재호출
	_result_cache: list[Dict[Any, Any]] = dict()

	## 추가 QuerySet이 될 타겟들 저장
	_prefetch_related_lookups: Tuple(str) = ()

	## SQL 결과값을 파이썬이 어떤 자료구조로 반환 받을 지 선언하는 프로퍼티
  ## values() : dict, values_list() : list 반환, 추가 옵션 없을 경우 Django Model 반환
	_iterable_class = ModelIterable
```

## select_related(), prefetch_related()

- `select_related` : join을 통해 즉시 데이터를 로딩하는 방식으로 정방향 참조 필드에 사용
- `prefetch_related`  : 추가 쿼리를 더 호출해 데이터를 즉시 가져오는 방식

- 손님 : 주문 = 1 : N
- 주문 : 상품  = N : M
    - 주문 입장에서 손님은 정방향 참조모델, 상품은 역방향 참조모델
    
    ```sql
    order_list = (
    		## User 정보 join
        Order.objects.select_related("order_owner")
    		## where 조건 절
        .filter(order_owner__username="username4")
    		## 해당 추가 쿼리를 통해 모든 상품 정보까지 호출
        .prefetch_related("product_set")
    )
    ```
    

`prefecth_related` 는 추가 쿼리셋으로 함수안에 선언한 개수 만큼 쿼리가 추가적으로 호출됨

```sql
## b_model, c_models 정보를 가져오기 위해 2번 더 sql 호출 
queryset = AModel.objects.prefetch_related("b_model_set", "c_models")
```

<aside>
💡 QuerySet 연습 문제

</aside>

```python
# 1
company_queryset : QuerySet = (Company.objects.filter(name='apple').prefetch_related('product_set'))

#2 
order_product = (OrderedProduct.objects.select_related('related_order','related_product').filter(related_order=1))
```

```sql
-- 1
SELECT *
FROM company
WHERE name='apple';

SELECT *
FROM product
WHERE product.company_id = 1;

-- 2
SELECT *
FROM orderedproduct
	Join order ON orderedproduct.order_id = order.id
	JOIN product ON orderedproduct.product_id = product.id
WHERE order_id=1 

```

## SQL Preformance를 체크하는 TestCase

django docs에서는 `assertNumQueries()` 를 추천하지만 그러면 API가 수정 될 때 마다 달라지는 SQL 갯수를 체크해줘야 하기 때문에 손이 많이 간다. 그래서 매번 체크해줘야 하기 때문에 꼼꼼히 볼 수도 있지만, 오히려 경각심을 낮출수도 있다.

해당 강의에서는 `CaptureQueriesContext` 추천

```python
from django.test.utils import CaptureQueriesContext
from rest_framework.test import APIClient

def test_check_n_plus_1_problem():
	from django.db import connection
	
	with CaptureQueriesContext(connection) as expected_num_queries:
		APIClient.get(path='/restaurants/")
	
	# 주문이 두 개 더 추가된 이후 API에서 발생하는 SQL Count
	Order.objects.create(
		total_pricee=1000,
	)
	Order.objects.create(
		total_pricee=5000,
	)
	
	with CaptureQueriesContext(connection) as checked_num_queries:
		APIClient.get(path='/restaurants/")

	# 이제 주문이 두 개 더 발생했다고 SQL이 2개 더 생성되었는지 여부를 확인한다.
	# 주문이 N개 생성되었다고 해서 SQL이 N개 더 생성되면 안된다! 
	# 즉, 아래의 두 쿼리셋의 길이가 같아야 한다.
	assert len(checked_num_queries) == len(expected_num_queries)
```

# 실수하기 쉬운 QuerySet의 특성들

## 1. prefetch_related() 와 filter()는 완전 별개다

```python
company_qs = Company.objects.prefetch_related("product_set").filter(name="company_name1", product__name__isnull=False)
```

`filter()` 는 한 개의 쿼리에 해당하는 데이터만 제어 하고 `prefetch_realted()` 는 추가 쿼리셋에 있는 데이터를 제어한다.

그런데 위의 경우 `.filter(name="company_name1", product__name__isnull=False` 때문에 `product` 를 join 해야하고

`.prefetch_related("product_set")` 때문에 product 정보를 가져오는 쿼리를 한번 더 호출 해야한다.

- 해결방법 1. QuerySet이 알아서 join으로 product를 가져오게 하는 방법
    
    ```python
    company_qs = Company.objects.filter(
        name="company_name1", product__name__isnull=False
    )
    ```
    
- 해결방법 2. `Prefetch()`
    
    ```python
    ompany_qs = Company.objects
    		.filter(name="company_name1")
    		.prefetch_related(
    			"product_set",
    		  Prefetch(queryset=Product.objects.filter(product__name__isnull=False)
    			),
    			)
    ```
    
- 추천하는 QuerySet 작성 순서
    - `annotate()`
    - `select_related()`
    - `filter()`
    - `prefetch_related()`
    - 이런 순서가 실제 SQL 순서와 가장 유사하기 때문!
    - 위에서 본 실수를 방지하기 위해 `prefetch_related()` 는 `filter()` 이후에 작성
    

## 2. Cache를 재활용 하지 못하는 QuerySet 호출

- 회사가 가진 상품들의 정보를 모두 Eager Loding 하라!

```python
company_list = list(Company.objects.prefetch_related("product_set").all())
company = company_list[0]

# SQL이 추가 발생하지 않음(이미 Eager Loading 했기 때문)
company.product_set.all()  

# 이런 경우 SQL이 추가 발생
company.product_set.filter(name="불닭볶음")  

# SQL을 추가로 발생시키지 않기 위한 방법 - list comprehension
fire_noodle_product_list = [
    product for product in company.product_set.all() if product.name == "불닭볶음"
]
```

## 3. SubQuery 발생 조건

서브쿼리는 슬로우 쿼리를 많이 야기함

서브 쿼리 옵션이 있긴 하지만, 해당 옵션을 주지 않은 경우에도 가끔 발생할 수 있다.

- Queryset in Queryset인 경우
    
    ```python
    company_queryset: QuerySet = Company.objects.filter(id__lte=20).values_list("id", flat=True)
    # company_queryset이 아직 실행되기 전이기 때문에 조건절로 들어 갔을 때 서브쿼리가 수행됨
    # 이럴 경우를 대비해 미리 list() 옵션으로 QuerySet이 미리 실행될 수 있도록 해야함
    product_queryset: QuerySet = Product.objects.filter(product_owned_company__id__in=company_queryset)
    ```
    
- `excldue()` 의 함정
    - 역방향 참조 모델 정상동작 ex
    
    ```python
    normal_joined_queryset = Order.objects.filter(description__isnull=False, product_set_included_order__name='asd')
    ```
    
    - 서브쿼리가 발생하는 동작 ex
        - `filter()` 에 넣어주었던 옵션을 `excldue()` 에 옮겨줫을 뿐인데 JOIN이 아닌 서브쿼리가 발생
    
    ```python
    normal_joined_queryset = Order.objects.filter(description__isnull=False).exclude(product_set_included_order__name='asd')
    ```
    
    - 그러나 정방향 참조 모델의 경우는 의도한 대로 JOIN을 제대로 수행함

## 4. values(), values_list() 주의점

`values()` , `values_list()` 를 사용하면 Eager Loading 옵션이 무시 되는 특성이 있다.

왜냐하면 DB의 row 단위로 데이터를 반환하기 때문에 객체와 객체 간의 매핑이 일어나지 않기 때문이다.

그래서 정말 JOIN 해야만 가져올 수 있는 데이터를 명시해야 JOIN이 실행 된다

```python
list(Product.objects.select_related('product_owned_company').filter(id=1).values(product_owned_company)
```

# QuerySet 사용 팁

1. 수행하려는 SQL 보다 가져오려하는 데이터 리스트를 먼저 떠올리기
2. 수행하려는 SQL이 QuerySet으로 한계가 있따면 RawQuerySet을 활용해보기
3. SQL의 성능을 위해서라면 NativeSQL 사용을 망설이지 말기