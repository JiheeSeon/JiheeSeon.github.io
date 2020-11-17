---
title: Celery를 통한 서버의 비동기 처리
author: JiheeSeon
layout: post
---

## Django — Celery  — Redis

### Intro

Django에서는 클라이언트의 요청을 view의 함수를 호출해 return문으로 응답한다. 통상적으로는 사용자에게 응답을 한 뒤 함수의 수명주기가 끝남에 따라 특정 작업을 수행할 수가 없다. 현재 진행 중이었던 프로젝트의 경우 클라이언트에게 노래의 제목을 받으면, 노래 파일과 메타 정보를 제공하면 된다. 그러나 이후 실행될 흐름 상, 노래에 대한 전처리를 하고, clustering을 위해 모델을 돌리는 등 각종 무거운 작업이 수반되어야 한다. 이 모든 작업을 하고 클라이언트에게 응답을 한다면, 족히 십분은 넘게 걸릴 것으로 상상도 할 수 없는 일이다.

클라이언트에게 응답하고나서 해당 처리를 백그라운드로 수행할 수 있도록 하는 방안을 모색하게 되었고, 그에 대한 해답은 celery에 있었다. celery는 python과 연동하여 비동기 task를 클라이언트에게 응답 후에 실행할 수 있도록 지원하는 툴로, 라이브러리로 import 하여 바로 사용하면 된다. 그렇다면 celery의 대략적인 동작 방식이 어떻길래 이와 같은 비동기 처리가 가능한 것일까?

### Message Broker —- Worker —- Task Queue

### 

celery와 함께 많이 나오는 개발 컨셉(용어)들이 있다. worker, message broker, task queue 등..
간략히 말하면, 서버에서 백그라운드로 시행하고자 하는 작업(Task)를 redis나 rabbitmq와 같은 broker에게 넘기면, 해당 브로커가 뒤에서 태스크를 처리할 일꾼, worker에게 이를 분담하여 넘겨준다. python의 경우 celery를 많이 사용하는데, celery worker가 task를 다 실행하고나면 그 결과를 함수 return에 보내주거나, bakend를 별도로 지정하면 이 결과를 데이터로 저장할 수있다. 데이터베이스를 지정하면, MSA에서 필요한 서로 다른 서버간의 태스크 결과를 공유할 수 있다는 점을 생각해볼 수있다.
###


결국 셀러리를 통해 하고자 하는 작업은
```

1. 비동기로 구현하고자 하는 부분을 함수의 형태로 만든다
2. @app.tasks decorator를 붙여 태스크로 지정한다.
3. 태스크를 하는 worker를 시작한다. (백그라운드로도 가능)
4. 태스크 함수를 호출한다. task_func_name.apply_asnyc((arg1, arg2), option= ~~)
5. 태스크의 반환값을 확인하여 다른 작업을 실행한다.
```


### TODO for async tasks

- [ ]  메시지 브로커를 골라 설치한다.
- [ ]  셀러리를 설치하고 태스크를 만든다.
- [ ]  워커를 시작하고 태스크를 호출한다.
- [ ]  태스크가 여러 상태로 전이하는 형태를 보고, 반환값을 확인한다.

같은 네트워크의 서버 컨테이너들은 서로 다른 컨테이너의 비동기 작업이 완료되었는지를 확인할 수 있어야 한다.

크게 task 요청을 하기 위해 delay, apply_async 두 함수를 사용할 수 있다. 일반적으로는 delay 함수를 많이 사용하나, apply_async는 보다 세부적인 지원 가능한 옵션이 있어, 이러한 옵션을 지정해야 할 때 사용하면 된다.

<a href="https://ibb.co/hXxYBWW"><img src="https://i.ibb.co/B4HnyZZ/1.png" alt="1" border="0"></a>


apply_async는 다양한 파라미터를 지원하는데, 위의 옵션 외에도 여러 옵션이 지원된다. 세부적인 옵션은 공식문서에서 확인할 수 있다. [https://docs.celeryproject.org/en/stable/reference/celery.app.task.html](https://docs.celeryproject.org/en/stable/reference/celery.app.task.html)

이 중 내가 사용한 파라미터 위주로 설명을 덧붙이면,

1. args : 비동기로 수행하고자 하는 task 함수의 인자 
비동기로 워커들에게 처리를 맡길 함수의 인자로, packed 형태에 맞추기 위해  tuple로 묶어서 넣어준다.

2. task_id : celery를 통해 비동기로 수행하고자 하는 task의 id
태스크의 결과값을 백엔드에 저장할 경우 key로 적용한다. 별도로 지정하지 않을 경우 아래와 같이 내부에서 생성된 임의의 id로 저장된다. 내 경우 태스크를 수행하는 컨테이너와 이 결과값을 이용해 전처리를 해야하는 컨테이너가 서로 다를 수 있기 때문에, 임의의 값이 아닌 사용자의 토큰 값을 키로 식별해 서로 다른 컨테이너 간 통신을 지원해야 했다. 이와 같이 백엔드에 기록할 때의 key 값을 지정하고 싶은 경우 task_id 옵션을 사용할 수 있다.

<a href="https://imgbb.com/"><img src="https://i.ibb.co/kMQyJqs/2.png" alt="2" border="0"></a>

3. shadow : log 기록 옵션
다음과 같은 log에서 찍히는 값을 사용자 지정 옵션으로 조정할 수 있도록 한다 task id를 할 경우와 위치가 다른 것을 볼 수 있다.

<a href="https://ibb.co/Jv0vYCv"><img src="https://i.ibb.co/zXjX3NX/3.png" alt="3" border="0"></a>


4. expires : 소멸 시간
현재 시각 기준으로 하루 이따가 result backend에 쓰인 기록이 소멸되기를 원한다면 다음과 같이 사용할 수 있다. expires=datetime.now() + timedelta(days=1)