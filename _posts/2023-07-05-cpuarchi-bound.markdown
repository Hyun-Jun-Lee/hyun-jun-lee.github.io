---
title: "CPU 아키텍처 & bound" #Article title.
date: 2023-07-05
category: [OS, OS] #One, more categories or no at all.
tag: [os]
---

__쉽게배우는 운영체제__

# CPU 아키텍처

## 

## 아키텍처 종류

## x86

- intel의 32bit cpu
- 가장 많이 쓰이는 아키텍처
- Windows, Linux, Mac os(BigSur 버전 까지)

## x86_64(amd64)

- amd가 만든 intel 기반의 64bit cpu
- x86과 호환됨
- Windows, Linux, Mac os(BigSur 버전 까지)

## arm

- arm 기반의 32bit CPU
- x86과는 아예 다른 새로운 아키텍처
- Linux, Mac OS (Monterey 부터), Android, iOS, 기타 모든 쪼그만 기기에서 성능 내야하는 경우 (공유기도 해당)

## arm64(arm64/v8)

- arm 기반의 64bit CPU
- 32bit arm과 호환 가능

모바일 앱은 저전력을 필요로 하기 때문에 대부분 arm 기반의 아키텍처로 만들어지고

PC는 맥의 m1 칩을 제외하고는 대부분 x86_64 아키텍처로 만들어진다.

프로그램은 결국 컴파일 된 바이너리이기 때문에, 컴파일을 어던 CPU로 했는지에 따라 결과물이 정해진다.

그래서 x86 아키텍처에서 빌드한 라이브러리를 그냥 arm에 가져오면 실행이 되지 않는 것이다.


# CPU bound / I.O Bound

## CPU Bound

CPU Bound는 일반적으로 CPU 사용 연산이 I/O 연산보다 많은 경우를 말한다.

복잡한 수학 연산이나 논리적 문제 해결을 할 때 많이 사용되고, CPU의 성능에 따라 또는 CPU 성능을 최대로 활용하는 설계 및 개발에 따라 성능이 결정된다.

- 이미지, 비디오 처리 : 픽셀 단위의 연산이 필요하기 때문에 CPU 연산이 많음
- 머신러닝 학습 : 대량의 데이터에 대한 복잡한 계산이 필요하기 때문에 많음
- 암호화/복호화 : 복잡한 수학적 연산이 필요

- 성능 향상법
    - 여러 CPU 코어를 활용할 수 잇도록 병렬처리
    - 캐싱
    - 알고리즘 연산, 복잡도를 낮출수 잇는 코드 최적화
    - CPU 업그레이드
    - Multi Processing

## I/O bound

- 파일 읽고 쓰기
- DB 쿼리 : 대용랑 데이터베이스에서 검색하는 작업은 디스크 I/O 가 많이 발생
- 네트워크 통신

- 성능 향상법
    - Non-blocing, Async
    - I/O 빈도를 낮추는 코드 최적화
    - Multi Threading