---
title: "Producer - Consumer pattern" #Article title.
date: 2023-12-21
category: [PYTHON,] #One, more categories or no at all.
tag: [Architecture,python, threading]
---

# 아키텍처의 필요성

내가 아키텍처의 필요성에 대해 느꼇던 상황은 아래와 같다. 

1. 초기에 서비스의 기능과 기획이 체계적이지 않을 때, 개발을 하다보니 전체적인 체계가 없어서 코드가 매우 난잡해서 새로 합류한 팀원들에게 전달 하기가 어려웠다. 
2. 클래스 별 책임이 제대로 나뉘어져 있지 않다 보니 재사용 할 수 있는 코드가 있음에도 불구하고, 중복되는 또는 비슷한 기능을 수행하는 메서드가 매우 많다.
3. 기능 및 요구사항이 많아짐에 따라 점점 더 서비스가 복잡해져갔고 구조를 정리하는 것이 힘들었다.

아래는 ChatGPT가 대답해준 아키텍처의 필요성

1. 시스템의 구조화: 소프트웨어 아키텍처는 복잡한 시스템을 체계적으로 구조화하고 조직화하는 데 도움을 줍니다. 이를 통해 시스템의 각 부분이 어떻게 상호작용하는지 명확하게 이해할 수 있으며, 개발 프로세스를 더 효율적으로 관리할 수 있습니다.
2. 성능 최적화: 좋은 아키텍처는 시스템의 성능 요구사항을 충족시키는 데 중요합니다. 시스템의 로드를 적절히 분산시키고, 효율적인 데이터 흐름 및 처리를 보장함으로써 전반적인 성능을 최적화할 수 있습니다.
3. 유지보수성 향상: 명확한 아키텍처는 시스템의 유지보수를 용이하게 합니다. 아키텍처가 잘 구성되어 있으면, 시스템의 특정 부분을 수정하거나 업그레이드하는 것이 더 간단하며, 오류를 더 쉽게 식별하고 수정할 수 있습니다.
4. 확장성 및 유연성: 소프트웨어가 성장하고 변화하는 요구사항에 맞춰 확장되어야 할 때, 잘 정의된 아키텍처는 새로운 기능이나 컴포넌트를 쉽게 추가할 수 있는 유연성을 제공합니다. 이는 장기적인 개발 및 운영에서 큰 이점을 가져다 줍니다.
5. 위험 관리: 소프트웨어 프로젝트에는 다양한 위험이 존재합니다. 아키텍처는 이러한 위험을 사전에 식별하고 관리하는 데 도움을 줍니다. 예를 들어, 보안, 신뢰성, 호환성 등의 문제를 미리 고려하여 설계할 수 있습니다.

그리고 실제 업무에 활용하기 위해 내가 발견한 패턴은 Producer-Consumer 패턴이다. 

# Producer - Consumer 패턴

병렬 처리를 해야할 작업이 필요해서 찾아보던 중 발견한 아키텍처 패턴이다. AI 쪽에서 방대한 양의 데이터를 학습시키기 위해 자주 사용하는 아키텍처라고 한다.

어떤 데이터가 생성되면, 그 생성된 데이터를 받아 처리하는 상황에서 멀티 스레드를 활용하여 처리량과 속도를 늘리고자할 적용할 수 있는 패턴이다. 데이터를 생성하는 스레드와 이를 처리하는 스레드를 분리할 수 있으며 각각은 1개 이상의 스레드로 구성될 수 있다. 데이터를 생성하는 스레드를 Producer, 데이터를 처리하는 스레드를 Consumer이다.

RabbitMQ, Kafka 같은 message queue 에 이와 같은 아키텍처 패턴이 적용되어 있다.


![image](https://github.com/BIS-KIT/BISKIT-Backend/assets/76996686/5e57f682-2725-4882-a11b-b0fd10cba781)

위 사진처럼 Producer는 각각의 작업을 queue에 전달하는 역활을 하고 Consumer에서는 queue에 있는 작업을 할당받아 수행하는 역활을 한다.그리고 Producer와 Consumer을 멀티 쓰레드로 구성하여 병렬처리를 한다.

## Example Producer-Consumer pattern

이런 멀트 스레딩 환경에서  Producer와 Consumer의 동기화 문제가 발생할 수 있다. 
예를들어, producer가 queue가 감당할 수 없을 만큼의 빠르게 데이터를 가져와서 빈 공간이 생길 때 까지 대기하거나 consumer가 작업얼 너무 빠르게 처리해서 해야 할 일없이 기다리고 있을 수 있다. 

이런 상황을 해결하기 위한 예시를 real python에서 찾앗다.

```
import random 

SENTINEL = object()

def producer(pipeline):
    """Pretend we're getting a message from the network."""
    for index in range(10):
        message = random.randint(1, 101)
        logging.info("Producer got message: %s", message)
        pipeline.set_message(message, "Producer")

    # Send a sentinel message to tell consumer we're done
    pipeline.set_message(SENTINEL, "Producer")

def consumer(pipeline):
    """Pretend we're saving a number in the database."""
    message = 0
    while message is not SENTINEL:
        message = pipeline.get_message("Consumer")
        if message is not SENTINEL:
            logging.info("Consumer storing message: %s", message)

if __name__ == "__main__":
    format = "%(asctime)s: %(message)s"
    logging.basicConfig(format=format, level=logging.INFO,
                        datefmt="%H:%M:%S")
    # logging.getLogger().setLevel(logging.DEBUG)

    pipeline = Pipeline()
    with concurrent.futures.ThreadPoolExecutor(max_workers=2) as executor:
        executor.submit(producer, pipeline)
        executor.submit(consumer, pipeline)
```

위 예시에서 producer는 랜덤한 메세지를 pipline의 set_message()를 호출해 consumer에게 보내는 역활을 하고 SENTINAL 이라는 값을 이용해 10개의 메세지를 consumer에게 보낸 후 중지하라는 신호를 보낸다. 

consumer는 pipline에서 메세지를 읽어오고 출력한다. 그리고 SENTINEL 값이 주어지면 스레드가 종료된다.

가장 중요한 Pipeline 코드는 아래와 같다.

```
class Pipeline:
    """
    Class to allow a single element pipeline between producer and consumer.
    """
    def __init__(self):
        self.message = 0
        self.producer_lock = threading.Lock()
        self.consumer_lock = threading.Lock()
        self.consumer_lock.acquire()

    def get_message(self, name):
        logging.debug("%s:about to acquire getlock", name)
        self.consumer_lock.acquire()
        logging.debug("%s:have getlock", name)
        message = self.message
        logging.debug("%s:about to release setlock", name)
        self.producer_lock.release()
        logging.debug("%s:setlock released", name)
        return message

    def set_message(self, message, name):
        logging.debug("%s:about to acquire setlock", name)
        self.producer_lock.acquire()
        logging.debug("%s:have setlock", name)
        self.message = message
        logging.debug("%s:about to release getlock", name)
        self.consumer_lock.release()
        logging.debug("%s:getlock released", name)
```


- message: 전달할 메시지를 저장
- producer_lock: threading.Lock 객체로, 생산자 스레드가 메시지에 접근하는 것을 제한
- consumer_lock: 또 다른 threading.Lock 객체로, 소비자 스레드가 메시지에 접근하는 것을 제한


init() 메서드에서는 3개의 member를 초기화하고,  consumer_lock에서 acquire()를 호출 한다. 현재 producer는 메세지를 추가할 수 있지만, consumer는 메세지가 준비될 때 까지 기다려야하는 상황이다. 

get_message()에서 consumer_lock.acquire()를 호출하고, 이는 consumer가 메세지가 준비될 때 까지 기다리게한다. 
consumer가 consumer_lock 을 얻으면, self.message의 값을 복사한 다음 producer_lock.release()를 호출해서 잠금을 해제하고, producer가 pipline에 다음 메세지를 넣을 수 있도록 한다.

set_message()에서는 producer가 producer_lock.acquire()를 호출하고 self.message 값을 복사한 다음, consumer_lock.release()를 호출해서 consumer가 다음 메시지를 읽을 수 있게 한다.

위와 같은 코드를 실행하면 결과 값이 아래와 같이 나온다.

```
$ ./prodcom_lock.py
Producer got data 43
Producer got data 45
Consumer storing data: 43
Producer got data 86
Consumer storing data: 45
Producer got data 40
Consumer storing data: 86
Producer got data 62
Consumer storing data: 40
Producer got data 15
Consumer storing data: 62
Producer got data 16
Consumer storing data: 15
Producer got data 61
Consumer storing data: 16
Producer got data 73
Consumer storing data: 61
Producer got data 22
Consumer storing data: 73
Consumer storing data: 22
```

## with Queue

동시에 여러개의 값을 pipeline에서 처리하기 위해 Queue를 활용한다. 

```
if __name__ == "__main__":
    format = "%(asctime)s: %(message)s"
    logging.basicConfig(format=format, level=logging.INFO,
                        datefmt="%H:%M:%S")
    # logging.getLogger().setLevel(logging.DEBUG)

    pipeline = Pipeline()
    event = threading.Event()
    with concurrent.futures.ThreadPoolExecutor(max_workers=2) as executor:
        executor.submit(producer, pipeline, event)
        executor.submit(consumer, pipeline, event)

        time.sleep(0.1)
        logging.info("Main: about to set event")
        event.set()
```

threading.Event() 객체는 하나의 객체가 이벤트를 호출할 수 있게하고, 다른 스레드 들이 해당 이벤트를 기다리게한다. 여기서 중요한 부분은 이벤트를 기다리는 스레드들이 현재 진행중인 작업을 중단할 필요 없이 이벤트의 상태를 확인하면된다는 것이다.

```
def producer(pipeline, event):
    """Pretend we're getting a number from the network."""
    while not event.is_set():
        message = random.randint(1, 101)
        logging.info("Producer got message: %s", message)
        pipeline.set_message(message, "Producer")

    logging.info("Producer received EXIT event. Exiting")
```

이제 producer는 event.is_set()이 될 때까지 반복되고, 더 이상 SENTINEL 값이 필요없다. 즉 이벤트가 설정될 때 까지 계속해서 메세지를 생성하고 파이프라인으로 보낸다. 

```
def consumer(pipeline, event):
    """Pretend we're saving a number in the database."""
    while not event.is_set() or not pipeline.empty():
        message = pipeline.get_message("Consumer")
        logging.info(
            "Consumer storing message: %s  (queue size=%s)",
            message,
            pipeline.qsize(),
        )

    logging.info("Consumer received EXIT event. Exiting")
```

consumer는 producer와 같이 이벤트가 is_set()될 때까지 뿐만 아니라 pipeline에 들어 있는 값이 없을 때 까지 반복된다. 여기서 queue에 비어있을 때 까지 consumer가 작업을 처리하는 것이 중요한데, 아직 pipeline 에 메세지가 남아있는데 종료를 하게 되면 새로운 메세지가 아직 전송되지 않은 메세지를 덮어쓰게되고, producer는 queue가 가득차서 메세지를 추가할 수가 없다.

```
class Pipeline(queue.Queue):
    def __init__(self):
        super().__init__(maxsize=10)

    def get_message(self, name):
        logging.debug("%s:about to get from queue", name)
        value = self.get()
        logging.debug("%s:got %d from queue", name, value)
        return value

    def set_message(self, value, name):
        logging.debug("%s:about to add %d to queue", name, value)
        self.put(value)
        logging.debug("%s:added %d to queue", name, value)
```

pipeline은 Queue의 서브클래스가 되고 초기화 시 queue의 최대 크기를 설정한다.
기존 lock 처리는 모두 Queue 자체 내부에 포함되었기 때문에 코드가 훨신 간단해졌다.

이제 위 코드를 실행시키면 병렬 작업이 되어 아래와 같이 실행된다.

```
$ ./prodcom_queue.py
Producer got message: 32
Producer got message: 51
Producer got message: 25
Producer got message: 94
Producer got message: 29
Consumer storing message: 32 (queue size=3)
Producer got message: 96
Consumer storing message: 51 (queue size=3)
Producer got message: 6
Consumer storing message: 25 (queue size=3)
Producer got message: 31

[many lines deleted]

Producer got message: 80
Consumer storing message: 94 (queue size=6)
Producer got message: 33
Consumer storing message: 20 (queue size=6)
Producer got message: 48
Consumer storing message: 31 (queue size=6)
Producer got message: 52
Consumer storing message: 98 (queue size=6)
Main: about to set event
Producer got message: 13
Consumer storing message: 59 (queue size=6)
Producer received EXIT event. Exiting
Consumer storing message: 75 (queue size=6)
Consumer storing message: 97 (queue size=5)
Consumer storing message: 80 (queue size=4)
Consumer storing message: 33 (queue size=3)
Consumer storing message: 48 (queue size=2)
Consumer storing message: 52 (queue size=1)
Consumer storing message: 13 (queue size=0)
Consumer received EXIT event. Exiting
```

첫번째 consumer 출력을 보면 메시지 하나를 꺼냈는데, queue_size=3 인 것을 보면 마지막 5번째 메세지가 아직 pipeline에 추가되지 않았다는 것을 알 수 있다. 즉, producer는 메세지를 5개 생성하고 4개까지 pipeline에 추가한 다음 스레드가 교체된 것이다.

`Producer received EXIT event. Exiting` 를 보면 메인 스레드가 이벤트를 "EXIT" 이벤트를 생성하며 producer가 종료되는 것을 알 수 있다. 이벤트가 설정되어 producer는 종료됬지만, consumer는 아직 pipeline이 모두 비워지지 않았기 때문에 작업이 진행된다.

# Reference

- https://realpython.com/intro-to-python-threading/#producer-consumer-threading