# 7. 고전적 코루틴

내용:

[7.1 코루틴의 종류](article7.md#71-코루틴의-종류)

[7.2 이 글의 배경](article7.md#72-이-글의-배경)

[7.3 코루틴은 제너레이터에서 어떻게 진화했는가?](article7.md#73-코루틴은-제너레이터에서-어떻게-진화했는가)

[7.4 코루틴으로 사용되는 제너레이터의 기본 동작](article7.md#74-코루틴으로-사용되는-제너레이터의-기본-동작)

[7.5 예제: 이동 평균을 계산하는 코루틴](article7.md#75-예제-이동-평균을-계산하는-코루틴)

[7.6 코루틴을 기동하기 위한 데커레이터](article7.md#76-코루틴을-기동하기-위한-데커레이터)

[7.7 코루틴 종료와 예외 처리](article7.md#77-코루틴-종료와-예외-처리)

[7.8 코루틴에서 값 반환하기](article7.md#78-코루틴에서-값-반환하기)

[7.9 `yield from` 사용하기](article7.md#79-yield-from-사용하기)

[7.10 `yield from`의 의미](article7.md#710-yield-from의-의미)

[7.11 사용 사례: 이산 이벤트 시뮬레이션을 위한 코루틴](article7.md#711-사용-사례-이산-이벤트-시뮬레이션을-위한-코루틴)

[7.12 고전적 코루틴을 위한 제너릭 자료형 힌트](article7.md#712-고전적-코루틴을-위한-제너릭-자료형-힌트)

[7.13 요약](article7.md#713-요약)

## 7.1 코루틴의 종류

> 코루틴은 다양하게 구현된다. 파이썬에서 조차 여러 방법으로 구현된다. [중략] 파이썬 3.5부터 코루틴은 언어 자체의 내장 기능이 된다. 그러나 파이썬 3.4에서 기존 언어 기능을 이용해 처음 구현된 코루틴을 이해하면 파이썬 3.5의 네이티브 코루틴을 이해하는 데 도움이 된다.[^1]
>
> \- 제시 지류 데이비스<sup>Jesse Jiryu Davis</sup>, 귀도 반 로섬<sup>Guido van Rossum</sup>
>
> <`asyncio` 코루틴을 이용한 웹 크롤러>

영어 사전에서 ‘yield’ 단어를 찾아보면 ‘생산한다’와 ‘양보한다’는 두 가지 뜻을 볼 수 있다. 파이썬 제너레이터에서 `yield` 키워드를 사용할 때, 이 두 가지 의미가 모두 적용된다. 예를 들어 `yield item` 문장은 `next()`의 호출자가 받을 값을 생성하고, 양보하고, 호출자가 진행하고 또 다른 값을 소비할 준비가 되어 다음번 `next()`를 호출할 때까지 제너레이터 실행을 중단한다. 호출자가 제너레이터에서 값을 꺼내오는 것이다.

파이썬 코루틴은 본질적으로 자신의 `send()` 메서드에 대한 호출로 구동되는 제너레이터다. 코루틴에서 `yield`의 본질적인 의미는 양보하는 것이다. 즉, 프로그램의 다른 부분에 제어권을 넘겨주고 계속 실행하도록 통지받을 때까지 기다린다. 호출자는 `my_coroutine.send(datum)`를 호출해 코루틴에 데이터를 전달할 수도 있다. 그러면 코루틴은 실행을 계속하고 멈췄던 `yield` 표현식의 값으로 `datum`을 가져온다. 일반적인 용법에서는 이런 방식으로 호출자가 코루틴에 계속해서 데이터를 전달한다. 제너레이터와는 대조적으로 코루틴은 일반적으로 데이터의 소비자이지, 생산자는 아니다.

데이터 흐름과 무관하게 `yield`는 협업적 멀티태스킹을 구현하기 위해 사용할 수 있는 일종의 제어 흐름 장치다. 다른 코루틴이 활성화되도록 각 코루틴이 중앙 스케줄러에 제어권을 양보한다.

버전 3.5부터 파이썬은 세 가지 코루틴을 제공한다.

>* **고전적 코루틴**
>
> `my_coro.send(data)` 호출을 통해 전달된 데이터를 소비하고, 표현식 안에서 `yield`를 이용해 데이터를 읽는 제너레이터 함수. 고전적 코루틴은 `yield from`을 이용해 다른 고전적 코루틴에 위임할 수 있다.
>* **제너레이터 기반 코루틴**
>
> `@types.coroutine`으로 데커레이트된 제너레이터 함수로서, 파이썬 3.5에 소개된 `await` 키워드와 호환된다.
>* **네이티브 코루틴**
>
> `async def`로 정의된 코루틴. 고전적 코루틴이 `yield from`을 사용한 방식과 비슷하게 `await` 키워드를 이용해 다른 네이티브 코루틴이나 제너레이터 기반 코루틴에 위임할 수 있다.

네이티브 코루틴과 제너레이터 기반 코루틴은 특히 비동기 I/O 프로그래밍을 위해 만들어졌다. 이 글에서는 `고전적 코루틴'에 대해 집중적으로 알아본다.

네이티브 코루틴이 고전적 코루틴에서 진화하기는 했지만, 네이티브 코루틴이 고전적 코루틴을 완전히 대체한 것은 아니다. 고전적 코루틴은 네이티브 코루틴이 흉내내지 못하는 유용한 동작 특성이 있고, 반대로 네이티브 코루틴에는 고전적 코루틴에 없는 기능을 갖고 있다.

고전적 코루틴은 제너레이터 함수에 대한 계속된 발전의 산물이다. 파이썬에서 코루틴의 진화 과정을 살펴보면 기능과 복잡도가 증가한 단계별로 특징을 파악하는 데 도움이 된다.

어떻게 제너레이터가 코루틴으로 작동하도록 발전했는지 간략히 살펴본 후에 이 글의 핵심으로 넘어간다. 그러고 나서 다음과 같은 내용을 설명한다.

● 코루틴으로 작동하는 제너레이터의 동작과 상태

● 데커레이터를 이용해서 코루틴을 자동으로 기동하기

● 제너레이터 객체의 `close()`와 `throw()` 메서드를 통해 호출자가 코루틴을 제어하는 방법

● 종료할 때 코루틴이 값을 반환하는 방법

● 새로운 `yield from` 구문의 사용법과 의미

● 사용 예: 시뮬레이션의 동시 활동을 관리하기 위한 코루틴


>**NOTE_** 이 글에서는 고전적 코루틴(제너레이터 기반 코루틴)을 이야기하기 위해 간단히 '코루틴'이라고 부르기도 한다. 네이티브 코루틴과 대비할 때는 명시적으로 '고전적 코루틴'이라고 부른다.

## 7.2 이 글의 배경

이 글은 <전문가를 위한 파이썬, 1판>의 16장을 갱신한 내용이다.

_네이티브 코루틴_ 이 소개되면서 2판에서는 해당 장을 제거하기로 결정했다. 이 주제를 공부하는 데 걸리는 시간과 난이도에 비해 예전만큼 중요하지 않기 때문이다. _코루틴_ 은 비동기 프로그래밍 이외에는 거의 사용되지 않으며, 그런 측면에서 _고전적 코루틴_ 은 더 이상 지원되지 않으며, 오로지 _네이티브 코루틴_ 만 지원된다.

그렇지만 _네이티브 코루틴_ 과 `await`를 깊이 있게 이해하려면 _고전적 코루틴_ 객체의 동작과 API, `yield from`의 정확한 의미를 알아야 한다고 필자는 생각한다. [PEP 492](https://peps.python.org/pep-0492/)는 `async`/`await` 구문을 소개했으며, [await 표현식<sup>await expression</sup> 절](https://peps.python.org/pep-0492/#await-expression)에서 다음과 같이 이야기 한다.

> 이것(`await`)은 `yield from` 구현을 이용해 추가적으로 인수를 검증한다.

그렇기 때문에 호기심이 많은 파이썬주의자들에게는 여전히 이 글에 관심을 가질 가치가 있다.

## 7.3 코루틴은 제너레이터에서 어떻게 진화했는가?

고전적 코루틴은 구문적으로 제너레이터와 똑같다. 다만 함수 본체 안에 `yield` 키워드가 있을 뿐이다. 그러나 코루틴 안에서 `yield`는 보통 표현식의 오른쪽에 나타나고(예, `datum = yield`), `yield` 키워드 뒤에 표현식이 없으면 아무런 값도 생성하지 않고 제너레이터가 `None`을 생성할 수도 있다. 코루틴은 호출자로부터 값을 받을 수도 있는데, 이 때는 코루틴을 기동하기 위해 `next(coro)` 대신 `coro.send(datum)`을 사용한다. 일반적으로 호출자는 코루틴에 값을 전달한다. `yield` 키워드를 통해 아무런 값도 주거나 받지 않을 수 있다. `yield`를 그저 제어 흐름의 관점에서 생각해보면, 코루틴이 왜 동시 프로그래밍에 유용한지 알 수 있다.

코루틴의 기반 구조는 파이썬 2.5(2006년)에 구현된 [PEP 342 - 향상된 제너레이터를 통한 코루틴<sup>Coroutines via Enhanced Generators</sup> 제안서](https://peps.python.org/pep-0342/)에 설명되어 있다. 이때부터 `yield` 키워드를 표현식에 사용할 수 있게 되었으며, `send(value)` 메서드가 제너레이터 API에 추가되었다. 제너레이터의 호출자는 `send()`를 이용해 제너레이터 함수 내부의 `yield` 표현식의 값이 될 데이터를 전송할 수 있다.

PEP 342는 `send()` 메서드 외에 `throw()`와 `close()`를 추가했다. `throw()` 메서드는 제너레이터 내부에서 처리할 예외를 호출자가 발생시킬 수 있게 해주며, `close()` 메서드는 제너레이터가 종료되도록 만든다. 이 기능은 다음 절과 [7.7 코루틴 종료와 예외 처리](article7.md#77-코루틴-종료와-예외-처리)에서 설명한다.

최근 코루틴으로의 혁신적인 진화는 파이썬 3.3(2012년)에서 구현된 [PEP 380 - 하위 제너레이터에 위임하기 위한 구문<sup>Syntax for Delegating to a Subgenerator</sup> 제안서](https://peps.python.org/pep-0380/)에 기술되어 있다. PEP 380은 제너레이터 함수에 다음과 같이 두 가지 구문을  변경하고 정의해 훨씬 더 유용하게 코루틴으로 사용할 수 있도록 만들었다.

● 제너레이터가 값을 반환할 수 있다. 이전에는 제너레이터에서 `return` 문으로 값을 반환하면 `SyntaxError` 가 발생했다.

● 기존 제너레이터가 하위 제너레이터<sup>subgenerator</sup>에 위임하기 위해 필요했던 수많은 판에 박힌 코드를 사용할 필요 없이, `yield from` 구문을 이용해 복잡한 제너레이터를 더 작은 내포된 제너레이터로 리팩토링할 수 있게 한다.

이 내용은 [7.8 코루틴에서 값 반환하기](article7.md#78-코루틴에서-값-반환하기)와 [7.9`yield from` 사용하기](article7.md#79-yield-from-사용하기)에서 설명한다.

PEP 380 이후 고전적 코루틴에 대한 큰 변화는 없었다. PEP 492는 네이티브 코루틴을 소개했는데, 여기에 대해서는 본 책의 2판에서 설명한다.

이제 이 책의 확고한 전통을 따라, 먼저 기본적인 사실을 설명하고 예제를 보여준 후 점점 더 어려운 특징으로 넘어가보자.

## 7.4 코루틴으로 사용되는 제너레이터의 기본 동작

[예제 1]은 코루틴의 동작을 보여준다.

_예제 1. 가장 간단한 코루틴 사용 예_
```python
>>> def simple_coroutine():  # (1)
...     print('-> coroutine started')
...     x = yield  # (2)
...     print('-> coroutine received:', x)
...
>>> my_coro = simple_coroutine()
>>> my_coro  # (3)
<generator object simple_coroutine at 0x100c2be10>
>>> next(my_coro)  # (4)
-> coroutine started
>>> my_coro.send(42)  # (5)
-> coroutine received: 42
Traceback (most recent call last):  # (6)
  ...
StopIteration
```
(1) 코루틴은 자신의 본체 안에 yield 문을 가진 일종의 제너레이터 함수로 정의된다.

(2) `yield`를 표현식에 사용한다. 단지 호출자에서 데이터를 받도록 설계하면 `yield`는 값을 생성하지 않는다. `yield` 키워드 뒤에 아무런 표현식이 없을 때 값을 생성하지 않으려는 의도를 암묵적으로 표현한다.

(3) 일반적인 제너레이터와 마찬가지로 함수를 호출해서 제너레이터 객체를 가져온다.

(4) 제너레이터가 아직 실행되지 않았으므로 `yield` 문에서 대기하지 않는다. 따라서 먼저 `next()`를 호출해 제너레이터를 `yield` 문까지 실행함으로써 데이터를 전송할 수 있는 상태를 만든다.

(5) 제너레이터의 `send()` 메서드를 호출해 코루틴 본체 안의 `yield` 문의 값을 `42`로 만든다. 이제 코루틴이 실행을 재개해서 다음 `yield` 문이 나오거나 종료될 때까지 실행한다.

(6) 여기서는 제어 흐름이 코루틴 본체의 끝에 도달하므로, 일반적인 제너레이터와 마찬가지로 `StopIteration` 예외를 발생시킨다.

코루틴은 다음과 같이 네 가지 상태를 가진다. `inspect.getgeneratorstate()` 함수를 이용해 현재 상태를 알 수 있다(이 함수는 다음 네 가지 상태 중 하나를 반환한다).

* **'GEN_CREATED'**

실행을 시작하기 위해 대기하고 있는 상태
* **'GEN_RUNNING'**

현재 인터프리터가 실행하고 있는 상태[^2]
* **'GEN_SUSPENDED'**

현재 `yield` 문에서 대기하고 있는 상태
* **'GEN_CLOSED'**

실행이 완료된 상태

`send()` 메서드에 전달한 인수가 대기하고 있는 `yield` 표현식의 값이 되므로, 코루틴이 현재 대기 상태에 있을 때는 `my_coro.send(42)`와 같은 형태로만 호출할 수 있다. 그러나 코루틴이 아직 기동되지 않은 상태(즉, `'GEN_CREATED'` 상태)인 경우에는 `send()` 메서드를 호출할 수 없다. 그래서 코루틴을 처음 활성화하기 위해 `next(my_coro)`를 호출한다(이 상태에서는 `my_coro.send(None)`을 호출해도 효과가 동일하다).

코루틴 객체를 생성하고 난 직후에 바로 `None`이 아닌 값을 전달하려고 하면 다음과 같은 오류가 발생한다.

```python
>>> my_coro = simple_coroutine()
>>> my_coro.send(1729)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: can't send non-None value to a just-started generator
```

에러 메시지를 통해 원인을 명확히 알 수 있다.

처음 `next(my_coro)`를 호출할 때, 코루틴을 ‘기동<sup>priming</sup>’한다고도 표현한다. 즉, 코루틴이 호출자로부터 값을 받을 수 있도록 처음 나오는 `yield` 문까지 실행을 진행하는 것이다.

`yield` 문을 한 번 이상 호출하는 코드를 보면, 코루틴의 동작을 좀 더 명확히 이해할 수 있다.

[예제 2]를 보자.

_예제 2. 두 번 생성하는 코루틴_
```python
>>> def simple_coro2(a):
...     print('-> Started: a =', a)
...     b = yield a
...     print('-> Received: b =', b)
...     c = yield a + b
...     print('-> Received: c =', c)
...
>>> my_coro2 = simple_coro2(14)
>>> from inspect import getgeneratorstate
>>> getgeneratorstate(my_coro2)  # (1)
'GEN_CREATED'
>>> next(my_coro2)  # (2)
-> Started: a = 14
14
>>> getgeneratorstate(my_coro2)  # (3)
'GEN_SUSPENDED'
>>> my_coro2.send(28)  # (4)
-> Received: b = 28
42
>>> my_coro2.send(99)  # (5)
-> Received: c = 99
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
>>> getgeneratorstate(my_coro2)  # (6)
'GEN_CLOSED'
```
(1) `inspect.getgeneratorstate()`가 `'GEN_CREATED'`를 반환하므로 아직 실행되지 않았음을 알 수 있다.

(2) 첫 번째 `yield` 문까지 진행하면서 `-> Started: a = 14` 메시지를 출력하고 `a`의 값을 생성한 후 `b`에 값이 할당될 때까지 대기한다.

(3) `inspect.getgeneratorstate()`가 `'GEN_SUSPENDED'`를 반환한다. 즉, 코루틴이 `yield` 문에서 대기하고 있는 상태다.

(4) 중단된 코루틴에 `28`이라는 숫자를 보낸다. `yield` 문의 값이 `28`로 평가되고 이 값은 `b`에 할당된다. `-> Received: b = 28` 메시지가 출력되고, `a + b`의 값(`42`)이 생성된다. 그리고 코루틴은 `c`에 할당할 값을 기다리기 위해 중단한다.

(5) 중단된 코루틴에 숫자 `99`를 보낸다. `yield` 문은 `99`로 평가되어 `c` 변수에 바인딩된다. `-> Received: c = 99` 메시지를 출력하고 나서 코루틴이 종료되므로 제너레이터 객체가 `StopIteration` 예외를 발생시킨다.

(6) `inspect.getgeneratorstate()`가 `'GEN_CLOSED'`를 반환하므로 코루틴 실행이 완료된 상태다.

코루틴 실행이 `yield` 키워드에서 중단됨을 잘 알고 있어야 한다. 앞에서 설명한 것처럼 할당문에서는 실제 값을 할당하기 전에 = 오른쪽 코드를 실행한다. 즉, `b = yield a`와 같은 코드에서는 나중에 호출자가 값을 보낸 후에야 변수 `b`가 설정된다. 이러한 방식에 익숙해지려면 신경을 더 써야 하지만, 이 방식을 제대로 알고 있어야 뒤에서 설명할 비동기 프로그래밍에서 `yield`의 용법을 이해할 수 있다.

[그림 1]에서 보는 것처럼 `simple_coro2` 코루틴의 실행은 세 단계로 나눌 수 있다.

1. `next(my_coro2)`는 첫 번째 메시지를 출력하고 `yield a`까지 실행되어 숫자 `14`를 생성한다.
2. `my_coro2.send(28)`은 `28`을 `b`에 할당하고, 두 번째 메시지를 출력하고, `yield a + b`까지 실행되어 숫자 `42`를 반환한다.
3. `my_coro2.send(99)`는 `99`를 `c`에 할당하고, 세 번째 메시지를 출력하고, 코루틴을 끝까지 실행시킨다.

_그림 1. `simple_coro2()` 코루틴을 실행하는 세 단계. 각 단계는 `yield` 표현식에서 끝나며, 다음 단계는 `yield` 표현식의 값을 변수에 할당하는 동일 행에서 시작한다._
![](./figs/flup_0701.png)

이제부터 약간 더 복잡한 코루틴 예제를 살펴보자.

## 7.5 예제: 이동 평균을 계산하는 코루틴

<본책> 9장에서 클로저를 설명하면서, 이동 평균을 계산하는 객체를 만들어보았다. [예제 9-7]에서는 간단한 클래스를 구현했고, [예제 9-13]에서는 클로저를 생성해 `total`과 `count` 변수를 보존하는 고급 함수를 구현했다.

[예제 3]은 코루틴을 이용해 이동 평균을 구하는 방법을 보여준다.[^3]

_예제 3. coroaverager0.py: 이동 평균 코루틴 코드_
```python
def averager():
    total = 0.0
    count = 0
    average = None
    while True:  # (1)
        term = yield average  # (2)
        total += term
        count += 1
        average = total/count
```

(1) 무한 루프이므로 이 코루틴은 호출자가 값을 보내주는 한 계속해서 값을 받고 결과를 생성한다. 이 코루틴은 호출자가 `close()` 메서드를 호출하거나, 이 객체에 대한 참조가 모두 사라져서 가비지 컬렉트되어야 종료된다.

(2) 이 `yield` 문은 코루틴을 중단하고, 지금까지의 평균을 생성하기 위해 사용된다. 나중에 호출자가 이 코루틴에 값을 보내면 루프를 다시 실행한다. 코루틴을 사용하면 `total`과 `count`를 지역 변수로 사용할 수 있다는 장점이 있다. 객체 속성이나 별도의 클로저 없이 평균을 구하는 데 필요한 값들을 유지할 수 있다.

[예제 4]는 `averager()` 코루틴을 활용하는 방법을 보여주는 `doctest`다.

_예제 4. coroaverager0.py: [예제 3]의 이동 평균 코루틴에 대한 `doctest`_
```python
    >>> coro_avg = averager()  # (1)
    >>> next(coro_avg)  # (2)
    >>> coro_avg.send(10)  # (3)
    10.0
    >>> coro_avg.send(30)
    20.0
    >>> coro_avg.send(5)
    15.0
```
(1) 코루틴 객체를 생성한다.

(2) `next()`를 호출해 코루틴을 기동시킨다.

(3) 이제 본격적인 작업이 시작된다. `send()`를 호출할 때마다 현재까지의 이동 평균이 생성된다.

[예제 4]의 `doctest`에서 `next(coro_avg)`를 호출하면 코루틴이 `yield` 문까지 실행되어 `average`의 초깃값이 `None`을 반환한다. 이 값은 콘솔에 나타나지 않는다. 이때 코루틴은 호출자가 값을 보내기를 기다리며 이 `yield` 문에서 중단한다. `coro_avg.send(10)`은 코루틴에 값을 보내 코루틴을 활성화시키고, 이 값을 `term`에 할당하고, `total`, `count`, `average`를 계산하고, `while` 루프를 다시 돌아 `average`를 생성하고, 또 다시 다른 값이 들어오기를 기다린다.

이 코드를 유심히 살펴봤다면, 본체 안에 무한 루프를 가진 `averager` 객체(여기서는 `coro_avg`)를 어떻게 종료시킬 수 있을지 궁금할 것이다. 이에 대해서는 [7.7 코루틴 종료와 예외 처리](article7.md#77-코루틴-종료와-예외-처리)에서 설명한다.

코루틴의 종료에 대해 설명하기 전에, 코루틴을 기동하는 방법에 대해 알아보자. 코루틴은 사용하기 전에 기동해야 하지만, 이런 사소한 작업은 까먹기 쉽다. 이런 문제를 해결하기 위해 특별한 데커레이터를 코루틴에 적용할 수 있다. 그 데커레이터 중 하나를 다음 절에서 설명한다.

## 7.6 코루틴을 기동하기 위한 데커레이터

코루틴은 기동되기 전에는 할 수 있는 일이 많지 않다. `my_coro.send(x)`를 처음 호출하기 전에 반드시 `next(my_coro)`를 호출해야 한다. 코루틴을 편리하게 사용할 수 있도록 기동하는 데커레이터가 종종 사용된다. 대표적으로 [예제 5]의 `@coroutine` 데커레이터가 널리 사용된다.[^4]

_예제 5. coroutil.py: 코루틴을 기동하기 위한 데커레이터_
```python
from functools import wraps

def coroutine(func):
    """데커레이터: `func`를 기동해서 첫 번째 `yield`까지 진행한다."""
    @wraps(func)
    def primer(*args, **kwargs):  # (1)
        gen = func(*args, **kwargs)  # (2)
        next(gen)  # (3)
        return gen  # (4)
    return primer
```
(1) 데커레이트된 제너레이터 함수는 `primer()` 함수로 치환되며, 실행하면 기동된 제너레이터를 반환한다.

(2) 데커레이트된 함수를 호출해서 제너레이터 객체를 가져온다.

(3) 제너레이터를 기동한다.

(4) 제너레이터를 반환한다.

[예제 6]은 `@coroutine` 데커레이터의 사용법을 보여준다. [예제 3]과 비교해보라.

_예제 6. coroaverager1.py: [예제 5]의 `@coroutine` 데커레이터를 사용한 이동 평균 코루틴의 `doctest`와 코드_
```python
"""
이동 평균을 계산하기 위한 코루틴

    >>> coro_avg = averager()  # (1)
    >>> from inspect import getgeneratorstate
    >>> getgeneratorstate(coro_avg)  # (2)
    'GEN_SUSPENDED'
    >>> coro_avg.send(10)  # (3)
    10.0
    >>> coro_avg.send(30)
    20.0
    >>> coro_avg.send(5)
    15.0

"""

from coroutil import coroutine  # (4)

@coroutine  # (5)
def averager():  # (6)
    total = 0.0
    count = 0
    average = None
    while True:
        term = yield average
        total += term
        count += 1
        average = total/count
```
(1) `averager()`를 호출해서 `@coroutine` 데커레이터의 `primer()` 함수 안에서 기동된 제너레이터 객체를 생성한다.

(2) `getgeneratorstate()` 함수가 `'GEN_SUSPENDED'`를 반환하므로, 코루틴이 값을 받을 준비가 되어 있다.

(3) `coro_avg` 객체에 바로 값을 전송할 수 있다. 이것이 바로 데커레이터를 사용하는 이유다.

(4) `@coroutine` 데커레이터를 임포트한다.

(5) `@coroutine` 데커레이터를 `averager()` 함수에 적용한다.

(6) 함수 본체는 [예제 3]과 완전히 똑같다.

코루틴과 함께 사용하도록 설계된 특별 데커레이터를 제공하는 프레임워크가 많이 있지만, 이 프레임워크들이 모두 코루틴을 기동시키는 것은 아니다. 코루틴을 이벤트 루프에 연결하는 등 다른 서비스를 제공하는 프레임워크도 있다. 그중 `Tornado` 비동기 네트워킹 라이브러리에서 제공하는 [@tornado.gen 데커레이터](https://www.tornadoweb.org/en/latest/gen.html)가 있다.

[7.9 yield from 사용하기](article7.md#79-yield-from-사용하기)에서 설명하겠지만, `yield from` 구문은 자동으로 자신을 실행한 코루틴을 기동시키므로, [예제 5]에서 설명한 `@coroutine` 데커레이터와 함께 사용할 수 없다. 파이썬 3.4 표준 라이브러리에서 제공하는 `@asyncio.coroutine` 데커레이터는 `yield from`과 함께 사용할 수 있게 설계되었으므로 코루틴을 기동시키지 않는다. 이 데커레이터는 <본책> 21장에서 설명한다.

이제 코루틴의 핵심 기능에 대해 자세히 살펴보자. 다음 절에서는 코루틴을 종료시키는 메서드와 코루틴에 예외를 던지는 메서드를 설명한다.

## 7.7 코루틴 종료와 예외 처리

코루틴 안에서 발생한 예외를 처리하지 않으면, `next()`나 `send()`로 코루틴을 호출한 호출자에 예외가 전파된다. [예제 7]은 [예제 6]의 데커레이트된 `averager` 코루틴을 사용한 예를 보여준다.

_예제 7. 처리하지 않은 예외에 의한 코루틴 종료_
```python
>>> from coroaverager1 import averager
>>> coro_avg = averager()
>>> coro_avg.send(40)  # (1)
40.0
>>> coro_avg.send(50)
45.0
>>> coro_avg.send('spam')  # (2)
Traceback (most recent call last):
  ...
TypeError: unsupported operand type(s) for +=: 'float' and 'str'
>>> coro_avg.send(60)  # (3)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```
(1) `@coroutine`으로 데커레이트된 `averager()`를 사용하므로, 코루틴에 바로 값을 보낼 수 있다.

(2) 비수치형 값을 보내면 코루틴 안에서 예외가 발생한다.

(3) 코루틴 안에서 예외를 처리하지 않으므로 코루틴이 종료된다. 이후에 코루틴을 다시 활성화하려면 StopIteration 예외가 발생한다.

코루틴의 `total` 변수에 더할 수 없는 `'spam'`이라는 문자열을 전송했으므로 에러가 발생했다.

[예제 7]을 보면 종료하라고 코루틴에 알려주는 구분 표시를 전송해서 코루틴을 종료할 수 있음을 알 수 있다. `None`이나 `Ellipsis`와 같은 내장된 싱글턴 상수는 구분 표시로 사용하기 좋다. `Ellipsis`는 데이터 스트림에서 상당히 보기 드문 표시라는 장점도 있다. 그리고 필자는 `StopIteration`을 사용하기도 한다(객체가 아니라 클래스 자체를 사용한다). 즉, `my_coro.send(StopIteration)` 형태로 사용한다.

파이썬 2.5 이후 제너레이터 객체는 호출자가 코루틴에 명시적으로 예외를 전달할 수 있게 해주는 `throw()`와 `close()` 메서드를 제공한다.

>* **`generator.throw(exc_type[, exc_value[, traceback]])`**

제너레이터가 중단한 곳의 `yield` 표현식에 예외를 전달한다. 제너레이터가 예외를 처리하면, 제어흐름이 다음 `yield` 문까지 진행하고, 생성된 값은 `generator.throw()` 호출 값이 된다. 제너레이터가 예외를 처리하지 않으면 호출자까지 예외가 전파된다.
>* **`generator.close()`**

제너레이터가 실행을 중단한 `yield` 표현식이 `GeneratorExit` 예외를 발생시키게 만든다. 제너레이터가 예외를 처리하지 않거나 `StopIteration` 예외(일반적으로 제너레이터가 실행을 완료할 때 발생한다)를 발생시키면, 아무런 에러도 호출자에 전달되지 않는다. `GeneratorExit` 예외를 받으면 제너레이터는 아무런 값도 생성하지 않아야 한다. 아니면 `RuntimeError` 예외가 발생한다. 제너레이터에서 발생하는 다른 예외는 모두 호출자에 전달된다.

>**TIP_** 제너레이터 객체 메서드에 대한 공식 문서는 파이썬 언어 깊은 곳([6.2.9.1절 '제너레이터-반복자 메서드<sup>Generator-iterator methods</sup>' 문서](https://docs.python.org/3/reference/expressions.html#generatoriterator-methods))에 숨어 있다.

이제 `close()`와 `throw()`가 어떻게 코루틴을 제어하는지 알아보자. [예제 8]은 잠시 후에 나올 예제에 사용할 `demo_exc_handling()` 함수를 보여준다.

_예제 8. coro_exc_demo.py: 코루틴의 예외 처리 방법을 설명하기 위한 제너레이터_
```python
class DemoException(Exception):
    """설명에 사용할 예외 유형"""

def demo_exc_handling():
    print('-> coroutine started')
    while True:
        try:
            x = yield
        except DemoException:  # (1)
            print('*** DemoException handled. Continuing...')
        else:  # (2)
            print(f'-> coroutine received: {x!r}')
    raise RuntimeError('This line should never run.')  # (3)
```
(1) `DemoException` 예외를 따로 처리한다.

(2) 예외가 발생하지 않으면 받은 값을 출력한다.

(3) 이 코드는 결코 실행되지 않는다.

[예제 8]의 마지막 줄은 도달할 수 없다. 위에 나온 무한 루프는 처리되지 않은 예외에 의해서만 중단될 수 있으며, 예외를 처리하지 않으면 코루틴의 실행이 바로 중단되기 때문이다.

`demo_exc_handling()`의 일반적인 연산은 [예제 9]와 같다.

_예제 9. 예외를 발생시키지 않는 `demo_exc_handling()`의 활성화 및 종료_
```python
    >>> exc_coro = demo_exc_handling()
    >>> next(exc_coro)
    -> coroutine started
    >>> exc_coro.send(11)
    -> coroutine received: 11
    >>> exc_coro.send(22)
    -> coroutine received: 22
    >>> exc_coro.close()
    >>> from inspect import getgeneratorstate
    >>> getgeneratorstate(exc_coro)
    'GEN_CLOSED'
```

`DemoException`을 코루틴 안으로 던지면, 이 예외가 처리되어 `demo_exc_handling()` 코루틴은 [예제 10]과 같이 계속 실행된다.

_예제 10. `DemoException`을 `demo_exc_handling()` 안에 던져도 종료되지 않는다_
```python
    >>> exc_coro = demo_exc_handling()
    >>> next(exc_coro)
    -> coroutine started
    >>> exc_coro.send(11)
    -> coroutine received: 11
    >>> exc_coro.throw(DemoException)
    *** DemoException handled. Continuing...
    >>> getgeneratorstate(exc_coro)
    'GEN_SUSPENDED'
```

한편 처리되지 않는 예외를 코루틴 안으로 던지면 [예제 11]과 같이 코루틴이 중단되고, 코루틴의 상태는 `'GEN_CLOSED'`가 된다.

_예제 11. 자신에게 던져진 예외를 처리할 수 없으면 코루틴이 종료된다_
```python
    >>> exc_coro = demo_exc_handling()
    >>> next(exc_coro)
    -> coroutine started
    >>> exc_coro.send(11)
    -> coroutine received: 11
    >>> exc_coro.throw(ZeroDivisionError)
    Traceback (most recent call last):
      ...
    ZeroDivisionError
    >>> getgeneratorstate(exc_coro)
    'GEN_CLOSED'
```

코루틴이 어떻게 종료되든 어떤 정리 코드를 실행해야 하는 경우에는 [예제 12]에서처럼 `try`/`finally` 블록 안에 코루틴의 해당 코드를 넣어야 한다.

_예제 12. coro_finally_demo.py: 코루틴 종료시 어떤 작업을 실행하기 위해 `try`/`finally` 사용하기_
```python
class DemoException(Exception):
    """설명에 사용할 예외 유형"""


def demo_finally():
    print('-> coroutine started')
    try:
        while True:
            try:
                x = yield
            except DemoException:
                print('*** DemoException handled. Continuing...')
            else:
                print(f'-> coroutine received: {x!r}')
    finally:
        print('-> coroutine ending')
```

`yield from` 구조체가 파이썬 3.3에 추가된 이유 중 하나는 중첩된 코루틴에 예외를 던지는 것과 관련 있다. 그리고 코루틴에서 값을 좀 더 편리하게 반환할 수 있게 하기 위한 이유도 있다. 다음 절에서 설명이 이어진다.

## 7.8 코루틴에서 값 반환하기

[예제 13]은 `averager()` 코루틴을 변형해서 값을 반환한다. 이 코루틴은 활성화할 때마다 이동 평균을 생성하지는 않는다. 의미 있는 값을 생성하지는 않지만 최후에 어떤 의미 있는 값을 반환하는(예를 들면 최종 합계를 반환하는 경우) 코루틴도 있음을 설명하기 위해서다.

[예제 13]의 `averager()`가 반환하는 결과는 `namedtuple`로서, 항목 수(`count`)와 평균(`average`)을 담고 있다. 그냥 `average` 값만 반환할 수도 있었지만, 튜플을 반환해서 누적된 데이터(항목 수)도 반환할 수 있음을 보여주고 싶었다.

_예제 13. coroaverager2.py: 결과를 반환하는 `averager()` 코루틴 코드_
```python
from collections import namedtuple

Result = namedtuple('Result', 'count average')


def averager():
    total = 0.0
    count = 0
    average = None
    while True:
        term = yield
        if term is None:
            break  # (1)
        total += term
        count += 1
        average = total/count
    return Result(count, average)  # (2)
```
(1) 값을 반환하려면 코루틴이 정상적으로 종료되어야 한다. 그렇기 때문에 이 `averager` 버전에서는 루프를 빠져나오는 조건을 검사한다.

(2) `count`와 `average`를 가진 `namedtuple`을 반환한다. 제너레이터 함수가 값을 반환하므로 파이썬 3.3 이전 버전에서는 이 구문에서 에러가 발생한다.

새로 만든 `averager()` 가 어떻게 작동하는지 알아보기 위해 [예제 14]처럼 콘솔에서 실행해볼 수 있다.

_예제 14. coroaverager2.py: `averager()`의 동작을 보여주는 `doctest`_
```python
    >>> coro_avg = averager()
    >>> next(coro_avg)
    >>> coro_avg.send(10)  # (1)
    >>> coro_avg.send(30)
    >>> coro_avg.send(6.5)
    >>> coro_avg.send(None)  # (2)
    Traceback (most recent call last):
       ...
    StopIteration: Result(count=3, average=15.5)
```
(1) 이 버전은 값을 생성하지 않는다.

(2) `None`을 보내면 루프를 빠져나오고 코루틴이 결과를 반환하면서 종료하게 된다. 일반적인 제너레이터 객체와 마찬가지로 `StopIteration` 예외가 발생한다. 예외 객체의 `value` 속성에는 반환된 값이 들어 있다.

`return` 문이 반환하는 값은 `StopIteration` 예외의 속성에 담겨 호출자에 밀반입됨에 주의하라. 약간 꼼수를 부렸지만, 실행이 완료되면 `StopIteration` 예외를 발생시키는 기존 제너레이터 객체의 작동 방식을 유지한다.

[예제 15]는 코루틴이 반환한 값을 가져오는 방법을 보여준다.

_예제 15. `StopIteration`을 잡으면 `arverager()`가 반환한 값을 가져올 수 있다_
```python
    >>> coro_avg = averager()
    >>> next(coro_avg)
    >>> coro_avg.send(10)
    >>> coro_avg.send(30)
    >>> coro_avg.send(6.5)
    >>> try:
    ...     coro_avg.send(None)
    ... except StopIteration as exc:
    ...     result = exc.value
    ...
    >>> result
    Result(count=3, average=15.5)
```

이 방법이 PEP 380에 정의되어 있고 `yield from` 구성체가 `StopIteration` 예외를 내부적으로 잡아서 자동으로 처리한다는 사실을 깨닫는다면, 코루틴에서 반환값을 가져오는 우회적인 방법이 타당하다는 생각이 들 것이다. 이것은 `for` 루프 안에서 `StopIteration`을 사용하는 방법과 비슷하다. 예외가 발생했다는 사실을 사용자가 모르도록 루프가 깔끔하게 처리한다. `yield from`의 경우 인터프리터가 `StopIteration` 예외를 처리할 뿐만 아니라 `value` 속성이 `yield from` 표현식의 값이 된다. 불행히도 이 과정은 콘솔에서 대화식으로 테스트할 수 없다. 함수 외부에서 `yield from`이나 `yield`를 사용하면 구문 에러가 발생하기 때문이다.[^5]

다음 절에서는 PEP 380에서 의도한 대로 결과를 생성하기 위해 `averager()` 코루틴을 `yield from`과 함께 사용하는 예를 보여준다.

## 7.9 `yield from` 사용하기

우선 `yield from`이 완전히 새로운 언어 구성체라는 점을 명심해야 한다. `yield`보다 훨씬 더 많은 일을 하므로 비슷한 키워드를 재사용한 것은 오해의 소지가 있다. 다른 언어에서는 이와 비슷한 구성체를 `await`라고 하는데, 핵심을 잘 전달하므로 더 좋은 키워드라고 생각한다. 제너레이터 `gen()`이 `yield from subgen()`을 호출하고, `subgen()`이 이어받아 값을 생성하고 `gen()`의 호출자에 반환한다. 실질적으로 `subgen()`이 직접 호출자를 이끈다. 그러는 동안 `gen()`은 `subgen()`이 종료될 때까지 실행을 중단한다.[^6]

그러나 `await`가 `yield from`을 완전히 대체한 것은 아니다. 각기 고유한 용도가 있으며, `await`는 콘텍스트와 타겟에 대해 더 엄격하다. 즉, `await`는 네이티브 코루틴 안에서만 사용할 수 있으며, 타겟은 _대기 가능한_ 객체이어야 한다. <본책> 21장 설명을 참조하라. 이와 반대로 `yield from`은 어느 함수에서나 사용할 수 있으며(그러면 그 함수가 제너레이터가 된다), 반복 가능한 객체는 어떤 것이든 타겟이 될 수 있다. 다음과 같은 초간단 `yield from` 예제는 `async`/`await` 구문으로 작성할 수 없다.

```python
>>> def gen123():
...     yield from [1, 2, 3]
...
>>> tuple(gen123())
(1, 2, 3)
```

<본책> 17장에서 `yield from`을 `for` 루프 안의 `yield`에 대한 단축문으로 사용할 수 있다고 설명했다.

예를 들어 다음 코드가 있다고 하자.
```python
>>> def gen():
...     for c in 'AB':
...         yield c
...     for i in range(1, 3):
...         yield i
...
>>> list(gen())
['A', 'B', 1, 2]
```

위 코드는 다음과 같이 작성할 수 있다.
```python
>>> def gen():
...     yield from 'AB'
...     yield from range(1, 3)
...
>>> list(gen())
['A', 'B', 1, 2]
```

[예제 16]의 코드는 `yield from`의 실용적인 예를 보여준다(사실 itertools 모듈은 C 언어로 구현된 최적화된 `chain` 제너레이터를 이미 제공하고 있다).

_예제 16. `yield from`으로 반복형 객체를 연결하기_
```python
>>> def chain(*iterables):
...     for it in iterables:
...         yield from it
...
>>> s = 'ABC'
>>> t = tuple(range(3))
>>> list(chain(s, t))
['A', 'B', 'C', 0, 1, 2]
```

약간 복잡하지만 더 유용한 `yield from`의 예는 데이비드 비즐리, 브라이언 K. 존스의 『Python Cookbook, 3E』(O’Reilly ) 4.14절 ‘중첩 시퀀스를 단일 시퀀스로 변환하기’에서 볼 수 있다(예제 코드는 [깃허브](https://github.com/dabeaz/python-cookbook/blob/master/src/4/how_to_flatten_a_nested_sequence/example.py)에 있다).

`yield from x` 표현식이 `x` 객체에 대해 첫 번째로 하는 일은 `iter(x)`를 호출해 `x`의 반복자를 가져오는 것이다. 이는 모든 반복형이 `x`에 사용될 수 있다는 의미다.

그러나 값을 생성하는 내포된 `for` 루프를 대체하는 게 `yield from`이 하는 일의 전부라면 `yield from` 구성자가 파이썬에 추가되지 않았을 것이다. `yield from`의 진정한 가치는 단순한 반복형을 이용해서는 설명할 수 없고, 중첩된 제너레이터를 복잡하게 사용하는 예제가 필요하다. 그렇
기 때문에 `yield from`을 제안한 PEP 380의 제목이 ‘하위 제너레이터에 위임하기 위한 구문’이다.

`yield from`의 주요한 특징은 가장 바깥쪽 호출자와 가장 안쪽에 있는 하위 제너레이터 사이에 양방향 채널을 열어준다는 것이다. 따라서 이 둘이 값을 직접 주고받으며, 중간에 있는 코루틴이 판에 박힌 듯한 예외 처리 코드를 구현할 필요 없이 예외를 직접 던질 수 있다. 이전에는 불가능했던 이 새로운 방식 덕분에 코루틴 위임<sup>coroutine delegation</sup>을 할 수 있게 되었다.

`yield from`을 사용하려면 코드를 상당히 많이 준비해야 한다. 필요한 작동 부위를 설명하기 위해 PEP 380은 다음과 같이 주요 용어들을 상당히 구체적으로 정의하고 있다.

>* **대표 제너레이터<sup>delegating generator</sup>**

`yield from <반복형>` 표현식을 담고 있는 제너레이터 함수
>* **하위 제너레이터<sup>subgenerator</sup>**

`yield from` 표현식 중 `<반복형>`에서 가져오는 제너레이터. PEP 380의 제목 ‘하위 제너레이터에 위임하기 위한 구문’에서 말하는 하위 제너레이터가 바로 이것이다.
>* **호출자<sup>caller</sup>**

PEP 380은 대표 제너레이터를 호출하는 코드를 '호출자'라고 표현한다. 문맥에 따라서 필자는 대표 제너레이터와 구분하기 위해 호출자 대신 '클라이언트'라는 용어를 사용하기도 한다. 하위 제너레이터 입장에서 보면 대표 제너레이터도 호출자이기 때문이다.

>**TIP_** PEP 380에서는 하위 제너레이터를 ‘반복자’라고 부르기도 한다. 그러나 대표 제너레이터도 일종의 반복자이므로 이런 용어는 혼란스럽다. 따라서 필자는 PEP의 제목에 따라 ‘하위 제너레이터’라고 부르는 것을 선호한다. 그러나 `yield from`이 `__next__()`, `send()`, `close()`, `throw()`를 구현하는 제너레이터를 처리하도록 만들어졌지만, `__next__()`만 구현한 간단한 반복형도 하위 제너레이터가 될 수 있으며, `yield from`도 이 반복형을 처리할 수 있다.

[예제 17]은 `yield from`이 작동하는 환경을 제공하며, [그림 2]는 이 예제에서 주요 부분을 설명한다.[^6]

그림 2. 대표 제너레이터가 `yield from`에서 중단하고 있는 동안, 호출자는 하위 제너레이터에 데이터를 직접 전송하고, 하위 제너레이터는 다시 데이터를 생성해서 호출자에 전달한다. 하위 제너레이터가 실행을 완료하고 인터프리터가 반환된 값을 첨부한 `StopIteration`을 발생시키면 대표 제너레이터가 실행을 재개한다.
![](/article7/figs/flup_0702.png)

coroaverager3.py 스크립트는 가상의 중1 남학생과 여학생의 몸무게와 키를 담은 딕셔너리를 읽는다. 예를 들어 `'boys;m'` 키는 남학생 9명의 키에 매핑되고(미터 단위), `'girls;kg'` 키는 여학생 10명의 몸무게에 매핑된다(킬로그램 단위). 이 스크립트는 지금까지 보아온 `averager()` 코루틴 각각에 데이터를 제공하고, 다음과 같이 결과를 출력한다.
```python
$ python3 coroaverager3.py
 9 boys  averaging 40.42kg
 9 boys  averaging 1.39m
10 girls averaging 42.04kg
10 girls averaging 1.43m
```

[예제 17]은 이 문제에 대한 가장 간단한 해결책은 아니지만, `yield from`을 사용하는 방법을 잘 보여준다. 이 예제는 ['파이썬 3.3의 새로워진 기능<sup>What’s New in Python 3.3</sup>'](https://docs.python.org/3/whatsnew/3.3.html#pep-380)에 나온 예제에서 영감을 얻었다.

_예제 17. coroaverager3.py: `yield from`을 이용해 `averager()`를 구동하고 보고서 생성하기_
```python
from collections import namedtuple

Result = namedtuple('Result', 'count average')


# 하위 제너레이터
def averager():  # (1)
    total = 0.0
    count = 0
    average = None
    while True:
        term = yield  # (2)
        if term is None:  # (3)
            break
        total += term
        count += 1
        average = total/count
    return Result(count, average)  # (4)


# 대표 제너레이터
def grouper(results, key):  # (5)
    while True:  # (6)
        results[key] = yield from averager()  # (7)


# 호출자
def main(data):  # (8)
    results = {}
    for key, values in data.items():
        group = grouper(results, key)  # (9)
        next(group)  # (10)
        for value in values:
            group.send(value)  # (11)
        group.send(None)  # 중요! (12)

    # print(results)  # 디버깅하려면 이 줄의 주석을 해제하라.
    report(results)


# 실행 결과 보고서
def report(results):
    for key, result in sorted(results.items()):
        group, unit = key.split(';')
        print(f'{result.count:2} {group:5}',
              f'averaging {result.average:.2f}{unit}')


data = {
    'girls;kg':
        [40.9, 38.5, 44.3, 42.2, 45.2, 41.7, 44.5, 38.0, 40.6, 44.5],
    'girls;m':
        [1.6, 1.51, 1.4, 1.3, 1.41, 1.39, 1.33, 1.46, 1.45, 1.43],
    'boys;kg':
        [39.0, 40.8, 43.2, 40.8, 43.1, 38.6, 41.4, 40.6, 36.3],
    'boys;m':
        [1.38, 1.5, 1.32, 1.25, 1.37, 1.48, 1.25, 1.49, 1.46],
}


if __name__ == '__main__':
    main(data)
```
(1) [예제 13]의 `averager()` 코루틴과 동일하다. 여기서는 하위 제너레이터다.

(2) `main()` 안의 클라이언트가 전송하는 각각의 값이 여기에 있는 `term` 변수에 바인딩된다.

(3) 종료 조건이다. 이 조건이 없으면 이 코루틴을 호출하는 `yield from`은 영원히 중단된다.

(4) 반환된 `Result`는 `grouper()`의 `yield from` 표현식의 값이 된다.

(5) `grouper()`는 대표 제너레이터다.

(6) 이 루프는 반복할 때마다 하나의 `averager()` 객체를 생성한다. 각 `averager()` 객체는 하나의 코루틴으로 작동한다.

(7) `grouper()`가 값을 받을 때마다, 이 값은 `yield from`에 의해 `averager()` 객체로 전달된다. `grouper()`는 클라이언트가 전송한 값들을 `averager()` 객체가 소진할 때까지 여기에서 중단된다. `averager()` 객체가 실행을 완료하고 반환한 값은 `results[key]`에 바인딩된다. 그러고 나서 `while` 루프가 `averager()` 객체를 하나 더 만들고 계속해서 값을 소비하게 한다.

(8) `main()`이 클라이언트 코드(PEP 380에서는 '호출자'라고 한다)다. 이 함수가 코드 전체를 실행한다.

(9) `group`은 결과를 저장할 `results` 딕셔너리와 특정 `key`로 `grouper()`를 호출해서 반환된 제너레이터 객체다. 이 객체는 코루틴으로 작동한다.

(10) 코루틴을 기동시킨다.

(11) 값을 하나씩 `grouper()`에 전달한다. 이 값은 `averager()`의 `term = yield` 문장의 `yield` 값이 된다. `grouper()`는 이 값을 볼 수 없다.

(12) `None`을 `grouper()`에 전달하면 현재 `averager()` 객체가 종료하고 `grouper()`가 실행을 재개하게 만든다. 그러면 `grouper()`는 또 다른 `averager()` 객체를 생성해서 다음 값들을 받는다.

[예제 17]에서 (12)번 설명이 가리키는 행의 `중요!` 주석은 `group.send(None)`의 중요함을 강조한다. `None`을 전송해야 현재의 `averager()` 객체가 종료되고 다음번 객체를 생성하게 된다. 이 코드를 제거하면 스크립트는 아무것도 출력하지 않는다. 이때 `main()` 함수의 거의 마지막 부근에 있는 `print(results)` 행의 주석을 해제하고 실행해보면 `results` 딕셔너리가 비어 있음을 알 수 있다.

>**NOTE_** 왜 `results`에 아무런 값도 수집되지 않았는지 그 원인을 직접 알아내면 `yield from`의 작동 방식을 이해하는 데 큰 도움이 될 것이다. coroaverager3.py의 소스코드는 [이 책의 리포지토리](https://github.com/fluentpython/example-code/blob/master/16-coroutine/coroaverager3.py)에 있다. 원인은 잠시 후에 설명한다.

여기서는 [예제 17]의 전반적인 작동 과정을 알아보고, `main()`에서 중요하다고 표시한 `group.send(None)`을 제거하면 어떤 일이 생기는지 설명한다.

● 바깥쪽 `for` 루프를 반복할 때마다 `group`이라는 이름의 `grouper()` 객체를 새로 생성한다. `group`이 대표 제너레이터다.

● `next(group)`을 호출해서 `grouper()` 대표 제너레이터를 기동시킨다. 그러면 `while True` 루프로 들어가서 하위 제너레이터 `averager()`를 호출한 후 `yield from`에서 대기한다.

● 내부 `for` 루프에서 `group.send(value)`를 호출해서 하위 제너레이터 `averager()`에 직접 데이터를 전달한다. 이때 `grouper()`의 현재 `group` 객체는 여전히 `yield from`에서 멈추게 된다.

● 내부 `for` 루프가 끝났을 때 `grouper()` 객체는 여전히 `yield from`에 멈춰있으므로, `grouper()` 본체 안의 `results[key]`에 대한 할당은 아직 실행되지 않는다.

● 바깥쪽 `for` 루프에서 마지막으로 `group.send(None)`을 호출하지 않으면, 하위 제너레이터인 `averager()`의 실행이 종료되지 않으므로, 대표 제너레이터인 `group`이 다시 활성화되지 않고, 결국 `results[key]`에 아무런 값도 할당되지 않는다.

● 바깥쪽 `for` 루프의 꼭대기로 올라가서 다시 반복하면, 새로운 `grouper()` 객체가 생성되어 `group` 변수에 바인딩된다. 기존 `grouper()` 객체는 더 이상 참조되지 않으므로 가비지 컬렉트된다(이때 아직 실행이 종료되지 않은 `averager()` 하위 제너레이터 객체도 가비지 컬렉트된다).

> **CAUTION_** 이 실험에서 챙겨야 할 핵심 내용을 정리해보자. 하위 제너레이터가 실행을 종료하지 않으면 대표 제너레이터는 영원히 `yield from`에 멈춰 있게 된다. 그렇다고 해서 전체 프로그램의 진행이 중단되지는 않는다. `yield`와 마찬가지로 `yield from`이 제어권을 클라이언트(즉, 대표 제너레이터의 호출자)에 넘겨주기 때문이다. 실행이 계속되더라도 일부 작업은 끝나지 않은 상태로 남아 있게 된다.

### 7.9.1 코루틴들의 파이프라인

[예제 17]은 대표 제너레이터와 하위 제너레이터가 하나씩만 있는 가장 간단한 형태의 `yield from` 사례를 보여준다. 대표 제너레이터가 일종의 파이프 역할을 하므로, 대표 제너레이터가 하위 제너레이터를 호출하기 위해 `yield from`을 사용하고, 그 하위 제너레이터는 대표 제너레이터가 되어 또 다른 하위 제너레이터를 호출하기 위해 `yield from`을 사용하는 과정을 반복해서 아주 긴 파이프라인도 만들 수 있다. 이 파이프라인은 결국 `yield`를 사용하는 간단한 제너레이터에서 끝나야 한다. 그러나 [예제 16]에서처럼 어떠한 반복형 객체에서도 끝날 수도 있다.

모든 `yield from` 체인은 가장 바깥쪽 대표 제너레이터에 `next()`와 `send()`를 호출하는 클라이언트에 의해 주도된다. 이 메서드들은 `for` 루프를 통해 암묵적으로 호출할 수도 있다.

이제 PEP 380에서 공식적으로 설명하고 있는 `yield from` 구성체에 대해 자세히 알아보자.

## 7.10 `yield from`의 의미

>**NOTE_** 이 부분은 이 책 1판에서 가장 어려운 부분 중 하나였다. `yield from`이 주로 레거시 비동기 프로그래밍 코드에서 사용되었고 이제는 `await`가 선호되고 있는 상황을 생각하면 이 부분에 신경쓸 필요가 있을까 하는 의문이 들 수도 있다. 그러나 `async`/`await`가 내부적으로 어떻게 돌아가는지 이해하고 싶다면 `yield from`을 알아야 한다. 기반 메커니즘은 똑같다. [PEP 492 - async와 await 구문을 이용한 코루틴<sup>Coroutines with async and await syntax</sup>](https://peps.python.org/pep-0492/)에서는 '`await`가 추가적으로 인수를 검증하면서 `yield from`을 이용한다'고 이야기한다.[^7] PEP 492도 PEP 380을 알고 있다고 가정하고 yield from이나 await의 동작에 대해 똑같은 수준으로 자세히 설명하고 있지 않다.

PEP 380을 개발하는 동안 이 제안서의 작성자인 그렉 이윙<sup>Greg Ewing</sup>은 제안한 구성체의 복잡도에 대한 질문을 받았다. 그는 '거의 언제나 사람들에게 중요한 정보는 제일 앞 문장 하나에 들어있다'고 대답하면서, 그 당시 다음과 같이 작성되어 있던 PEP 380 초안의 일부를 인용한다.

>그 반복자가 또 다른 제너레이터인 경우, 하위 제너레이터의 본체가 `yield from` 표현식의 대상 안에 들어가는 것과 동일한 효과가 발생한다. 게다가 하위 제너레이터는 값을 가진 `return` 문을 이용해서 값을 반환할 수 있고, 그 값은 `yield from` 표현식의 값이 된다.[^8]

이러한 위로의 말은 더 이상 PEP에 들어 있지 않다. 극단적인 사례를 모두 설명하지 않기 때문이다. 그렇지만 처음으로 접근하는 설명으로서는 나쁘지 않다.

여기서는 두 단계로 `yield from`을 파고 들어간다. 먼저 기본적인 동작을 보면서 여러 사용 예를 살펴본다. 그러고 나서 하위 제너레이터가 완료되기 전에 종료되거나 예외가 발생하는 경우에 무슨 일이 생기는지 살펴볼 것이다.

### 7.10.1 `yield from`의 기본 동작

승인된 PEP 380에서는 [제안 부분](https://peps.python.org/pep-0380/#proposal)에서 6개의 항목으로 `yield from`의 동작을 설명한다. 다음은 제안서의 내용을 거의 그대로 옮겼지만, '반복자'라는 말을 '하위 제너레이터'로 변경하고, 말을 약간 다듬었다. 먼저 [예제 17]의 coroaverager3.py에서 보여준 다음 네 가지 특징을 알아보자.

● 하위 제너레이터가 생성하는 값은 모두 대표 제너레이터의 호출자(즉, 클라이언트)에 바로 전달된다.

● `send()`를 통해 대표 제너레이터에 전달한 값은 모두 하위 제너레이터에 직접 전달된다. 값이 `None`이면 하위 제너레이터의 `__next__()` 메서드가 호출된다. 전달된 값이 `None`이 아니면 하위 제너레이터의 `send()` 메서드가 호출된다. 호출된 메서드에서 `StopIteration` 예외가 발생하면 대표 제너레이터의 실행이 재개된다. 그 외의 예외는 대표 제너레이터에 전달된다.

● 제너레이터나 하위 제너레이터에서 `return expr` 문을 실행하면, 제너레이터를 빠져나온 후 `StopIteration(expr)` 예외가 발생한다.

● 하위 제너레이터가 실행을 마친 후 발생한 `StopIteration` 예외의 첫 번째 인수가 `yield from` 표현식의 값이 된다.

PEP 380에서 설명한 `yield from`의 나머지 특징 두 가지는 예외와 종료에 관련되어 있다. 여기에 대해서는 [7.10.2 `yield from`에서의 예외 처리](article7.md#7102-yield-from에서의-예외-처리)에서 다루므로 일단은 '정상적인' 운영 조건에서의 `yield from`의 동작에 대해 알아보자.

`yield from`의 세부적인 의미는 미묘하다. 특히 예외를 처리하는 부분이 그렇다. 그렉 이윙은 이렇게 미묘한 작동을 PEP 380에 잘 설명하고 있다.

그렉 이윙은 또한 파이썬과 비슷한 의사코드를 이용해서 `yield from`의 동작을 문서화했다. 필자는 개인적으로 PEP 380의 의사코드를 분석해보는 것이 도움이 된다고 생각한다. 그러나 의사코드의 길이가 40줄이나 되므로 한눈에 바로 이해하기는 쉽지 않을 것이다.

그런 의사코드를 분석해야 할 때는 먼저 `yield from`의 가장 기본적이고 일반적인 사례를 가정해서 따라가는 것이 좋다.

`yield from`이 대표 제너레이터 안에 있다고 생각하자. 클라이언트 코드는 대표 제너레이터를 움직이고, 대표 제너레이터는 하위 제너레이터를 움직인다. 그리고 관련된 논리를 단순화하기 위해 클라이언트가 대표 제너레이터의 `throw()`나 `close()`를 결코 호출하지 않는다고 가정하자. 그리고 하위 제너레이터가 실행을 마칠 때까지 예외를 발생시키지 않고, 실행을 마친 후에는 인터프리터가 `StopIteration` 예외를 발생시킨다고 가정하자.

[예제 17]의 coroaverager3.py는 이러한 단순한 가정을 기반으로 작성한 코드다. 사실 실제 코드에서는 대표 제너레이터가 실행을 완료하는 경우도 많다. 어쨌든 이렇게 단순한 세상에서는 `yield from`이 어떻게 작동하는지 살펴보자.

[예제 18]의 의사코드는 대표 제너레이터에서 다음 문장 하나를 확장한 것과 대등하다.

```python
RESULT = yield from EXPR
```

이제 [예제 18]의 처리 흐름을 차분히 따라가 보라.

_예제 18. 대표 제너레이터 안의 `RESULT = yield from EXPR` 문에 해당하는 단순화한 의사코드. 여기서는 단순화한 가정에 따라 `throw()`와 `close()` 메서드를 지원하지 않고, `StopIteration` 예외만 지원한다_
```python
_i = iter(EXPR)  # (1)
try:
    _y = next(_i)  # (2)
except StopIteration as _e:
    _r = _e.value  # (3)
else:
    while 1:  # (4)
        _s = yield _y  # (5)
        try:
            _y = _i.send(_s)  # (6)
        except StopIteration as _e:  # (7)
            _r = _e.value
            break

RESULT = _r  # (8)
```
(1) 반복자 `_i`(하위 제너레이터)를 가져오기 위해 `iter()` 함수를 적용하므로, 어떠한 반복형도 `EXPR`로 사용할 수 있다.

(2) 하위 제너레이터를 기동시킨다. 반환된 값은 `_y`에 저장되어 최초의 생성 값으로 사용된다.

(3) `StopIteration`이 발생하면 예외 객체에서 `value` 속성을 꺼내 `_r`에 할당한다. 이 값이 가장 간단한 경우의 `RESULT` 값이 된다.

(4) 이 루프를 실행하는 동안 대표 제너레이터는 실행이 중단되고 단지 호출자와 하위 제너레이터 간의 통로 역할만 수행한다.

(5) 하위 제너레이터에서 생성된 값을 그대로 생성하고, 호출자가 보낼 `_s`를 기다린다. 이 코드 안에서 유일하게 사용된 `yield` 문이라는 점에 주의하라.

(6) 호출자가 보낸 `_s`를 하위 제너레이터에 전달하면서 하위 제너레이터의 실행을 진행시킨다.

(7) 하위 제너레이터가 `StopIteration` 예외를 발생시키면, 예외 객체 안의 `value` 속성을 가져와서 `_r`에 할당하고, 루프를 빠져나온 후, 대표 제너레이터의 실행을 재개한다.

(8) `r`이 전체 `yield from` 표현식의 값이 되어 `RESULT`에 저장된다.

이 간단한 의사코드에서 필자는 PEP 380에 사용된 의사코드의 원래 변수명을 유지했다. 여기에 사용된 변수는 다음과 같다.
>* **`_i` (iterator)**

하위 제너레이터
>* **`_y` (yielded)**

하위 제너레이터가 생성한 값
>* **`_r` (result)**

최종 결과값(즉, 하위 제너레이터가 종료된 후 `yield from` 표현식의 값)
>* **`_s` (sent)**

호출자가 대표 제너레이터에 보낸 값. 하위 제너레이터에 전달된다.
>* **`_e` (exception)**

예외(이 간단한 의사코드에서는 `StopIteration` 객체만 발생한다.)

`throw()`와 `close()`를 처리하지 않는 것 외에도, 단순화한 의사코드에서는 클라이언트에 의한 `next()`와 `send()` 호출을 하위 제너레이터에 전달하기 위해 `send()`만 사용하고 있다. 일단 처음 구조를 파악할 때는 `next()`와 `send()`의 구분에 신경 쓸 필요 없다. 앞에서 이야기한 것처럼 `yield from`이 [예제 18]의 단순화한 의사코드에서 보여준 것만 처리하는 경우에는 [예제 17]의 coroaverager3.py도 제대로 작동한다.

다음 소절에서는 클라이언트가 취소하거나 처리되지 않은 예외가 발생해서 하위 제너레이터가 조기 종료할 때 `yield from`의 동작을 보여준다.

### 7.10.2 `yield from`에서의 예외 처리

[7.10.1 `yield from`의 기본 동작](article7.md#7101-yield-from의-기본-동작)에서는 PEP 380에서 제안한 `yield from`의 처음 네 가지 특징 및 이 동작을 설명하는 의사코드를 살펴보았다. 그러나 현실은 더 복잡하다. 클라이언트가 호출하는 `throw()`와 `close()`를 처리해 하위 제너레이터에 전달해야 하기 때문이다. PEP 380에서 제안한 나머지 두 가지 특징을 약간 편집하면 다음과 같다.

● 대표 제너레이터에 던져진 `GeneratorExit` 이외의 예외는 하위 제너레이터의 `throw()` 메서드에 전달된다. `throw()` 메서드를 호출해서 `StopIteration` 예외가 발생하면 대표 제너레이터의 실행이 재개된다. 그 외의 예외는 대표 제너레이터에 전달된다.

● `GeneratorExit` 예외가 대표 제너레이터에 던져지거나 대표 제너레이터의 `close()` 메서드가 호출되면 하위 제너레이터의 `close()` 메서드가 호출된다. 그 결과 예외가 발생하면 발생한 예외가 대표 제너레이터에 전달된다. 그렇지 않으면 대표 제너레이터에서 `GeneratorExit` 예외가 발생한다.

그리고 하위 제너레이터가 `throw()`나 `close()`를 지원하지 않는 단순한 반복형일 수 있으므로 이런 경우에는 `yield from` 장치에서 처리해야 한다. 즉, 하위 제너레이터가 이 메서드들을 구현하지 않으면, 하위 제너레이터에서 예외가 발생하고, 이 예외를 `yield from` 장치에서 처리해야 한다. 하위 제너레이터는 호출자가 의도하지 않았던 예외도 발생시킬 수 있으며, 이 예외도 `yield from` 구현에서 처리해야 한다. 마지막으로, 최적화하기 위해 호출자가 호출하는 `next()`와 `send(None)`은 모두 하위 제너레이터의 `next()`를 호출한다. 호출자가 `None`이 아닌 값으로 `send()`를 호출할 때만 하위 제너레이터의 `send() 메서드가 사용된다.

여러분의 편의를 위해 PEP 380에서 설명하는 `yield from`을 확장한 의사 코드 전체를 여기에서 보여준다. [예제 19]는 그대로 복사한 코드에 필자가 번호 설명을 추가한 것이다.

`yield from` 의사코드 논리의 대부분은 최대 4단계까지 들어가는 6개의 `try`/`except` 블록에서 구현되므로 이해하기 약간 복잡하다. 그 외 `while` 키워드가 1개, `if`가 1개, `yield`가 1개 사용되었다. `while`, `yield`, `next()`, `send()`에 주의해서 살펴보면 전체적인 구조를 이해하는 데 도움이 될 것이다.

[예제 19]에 나온 코드 전체는 대표 제너레이터 본체 안의 다음 한 줄 코드를 확장한 것임을 기억하라.

```python
RESULT = yield from EXPR
```

_예제 19. 대표 제너레이터에서 `RESULT = yield from EXPR` 문과 대등한 의사코드_
```python
_i = iter(EXPR)  # (1)
try:
    _y = next(_i)  # (2)
except StopIteration as _e:
    _r = _e.value  # (3)
else:
    while 1:  # (4)
        try:
            _s = yield _y  # (5)
        except GeneratorExit as _e:  # (6)
            try:
                _m = _i.close
            except AttributeError:
                pass
            else:
                _m()
            raise _e
        except BaseException as _e:  # (7)
            _x = sys.exc_info()
            try:
                _m = _i.throw
            except AttributeError:
                raise _e
            else:  # (8)
                try:
                    _y = _m(*_x)
                except StopIteration as _e:
                    _r = _e.value
                    break
        else:  # (9)
            try:  # (10)
                if _s is None:  # (11)
                    _y = next(_i)
                else:
                    _y = _i.send(_s)
            except StopIteration as _e:  # (12)
                _r = _e.value
                break

RESULT = _r  # (13)
```
(1) 반복자 `_i`(하위 제너레이터)를 가져오기 위해 `iter()` 함수를 적용하므로, 어떠한 반복형도 `EXPR`로 사용할 수 있다.

(2) 하위 제너레이터를 기동시킨다. 반환된 값은 `_y`에 저장되어 최초의 생성 값으로 사용된다.

(3) `StopIteration`이 발생하면 예외 객체에서 `value` 속성을 꺼내 `_r`에 할당한다. 이 값이 가장 간단한 경우의 `RESULT` 값이 된다.

(4) 이 루프를 실행하는 동안 대표 제너레이터는 실행이 중단되고 단지 호출자와 하위 제너레이터 간의 통로 역할만 수행한다.

(5) 하위 제너레이터에서 생성된 값을 그대로 생성하고, 호출자가 보낼 `_s`를 기다린다. 이 코드 안에서 유일하게 사용된 `yield` 문이라는 점에 주의하라.

(6) 대표 제너레이터와 하위 제너레이터의 종료를 처리한다. 모든 반복형이 하위 제너레이터가 될 수 있으므로, 하위 제너레이터에 `close()` 메서드가 없을 수도 있다.

(7) 호출자가 `throw()`로 던진 예외를 처리한다. 하위 제너레이터가 `throw()` 메서드를 구현하지 않을 수도 있는데, 이 경우 대표 제너레이터에서 예외가 발생한다.

(8) 하위 제너레이터가 `throw()` 메서드를 가진 경우, 호출자로부터 받은 예외를 이용해서 호출한다. 하위 제너레이터는 예외를 처리하거나(이 경우 루프가 계속 실행된다), `StopIteration` 예외를 발생시키거나(이 예외에서 `_r` 결과를 추출하고 루프를 종료한다), 여기에서 처리할 수 없는 예외를 발생시킬 수도 있다(이 예외는 대표 제너레이터로 전파된다).

(9) 생성할 때 예외가 발생하지 않은 경우다.

(10) 하위 제너레이터를 계속 실행시킨다.

(11) 최근에 호출자로부터 받은 값이 `None`이면 하위 제너레이터의 `next()`를 호출하고, 그 외의 경우에는 `send()`를 호출한다.

(12) 하위 제너레이터가 `StopIteration` 예외를 발생시키면, 예외 객체 안의 `value` 속성을 가져와서 `_r`에 할당하고, 루프를 빠져나온 후, 대표 제너레이터의 실행을 재개한다.

(13) `_r`이 전체 `yield from` 표현식의 값이 되어 `RESULT`에 저장된다.

[예제 19]의 앞 부분 중 (2)번 설명을 보면 하위 제너레이터가 기동됨을 알 수 있다.[^9] 따라서 [7.6 코루틴을 기동하기 위한 데커레이터](article7.md#76-코루틴을-기동하기-위한-데커레이터)에서 설명한 자동 기동 데커레이터는 `yield from`과 함께 사용할 수 없다.

이 절을 시작할 때 인용했던 [메시지](https://mail.python.org/pipermail/python-dev/2009-March/087385.html)에서 그렉 이윙은 `yield from`의 의사코드에 대해 다음과 같이 설명한다.

>의사코드로 확장한 것을 분석해서 `yield from` 구문의 의미를 이해하라는 것은 아니다. 의사코드는 단지 언어 변호사에게 필요한 모든 세부사항을 확정하기 위한 것일 뿐이다.

여러분의 학습 스타일에 따라 의사코드 확장에 대해 꼼꼼히 따져보는 것이 그리 도움이 되지 않을 수도 있다. `yield from`을 활용한 실제 코드를 연구하는 것이 의사코드 구현을 뚫어지게 쳐다보는 것보다 도움이 될 것이다. 그러나 필자가 본 거의 모든 `yield from` 예제는 `asyncio` 모듈을 이용한 비동기 프로그래밍과 관련되어 있어서, 활성화된 이벤트 루프에 의존해서 작동한다. 그리고 비동기 코드들은 이제 `yield from` 대신 `await`를 사용한다.

이제 코루틴 사용법의 고전적인 예제인 시뮬레이션 구현으로 넘어가자. 이 예제에서는 `yield from`을 사용하지 않지만, 단일 스레드에서 동시에 발생하는 활동을 관리하기 위해 코루틴을 사용하는 방법을 보여준다.

## 7.11 사용 사례: 이산 이벤트 시뮬레이션을 위한 코루틴

> 코루틴은 시뮬레이션, 게임, 비동기 입출력, 그 외 이벤트 주도 프로그래밍이나 협업적 멀티태스킹 등의 알고리즘을 자연스럽게 표현한다.[^10]
> _ 귀도 반 로섬, 필립 J. 이바이<sup>Phillip J. Eby</sup>
> PEP 342 - 향상된 제너레이터를 통한 코루틴<sup>Coroutines via Enhanced Generators</sup>

이 절에서는 코루틴과 표준 라이브러리 객체만 이용해서 구현한 아주 간단한 시뮬레이션을 설명한다. 시뮬레이션은 컴퓨터 과학 문헌에서 코루틴을 응용하는 고전적인 사례다. 최초의 객체지향 언어인 Simula는 시뮬레이션을 간결하게 표현하기 위해 코루틴 개념을 소개했다.

>**NOTE_** 다음에 나오는 시뮬레이션 예제들은 학문적 연구를 위한 것이 아니다. 코루틴은 `ayncio` 패키지의 핵심 기반이다. 여기에서 설명하는 시뮬레이션은 스레드 대신 코루틴을 이용해서 동시에 발생하는 활동을 구현하는 방법을 보여준다. 이 내용은 <본책> 21장에서 `asyncio`를 처리할 때 많은 도움이 될 것이다.

예제에 들어가기 전에 시뮬레이션에 대해 간단히 정리해보자.

### 7.11.1 이산 이벤트 시뮬레이션에 대해

이산 이벤트 시뮬레이션<sup>discrete event simulation</sup>(DES)은 시스템을 일련의 이벤트로 모델링한다. DES에서 시뮬레이션 '시계'는 고정된 값만큼 진행하는 것이 아니라, 모델링된 다음 이벤트의 시뮬레이션된 시각으로 바로 진행한다. 예를 들어 상위 수준 관점에서 택시 운행을 시뮬레이션하는 경우, 승객을 태우는 이벤트 다음의 이벤트는 승객을 내리는 이벤트다. 운행 시간이 5분이든 50분이든 중요하지 않다. 승객을 내리는 이벤트가 발생하면 운행을 마친 시간으로 시계를 갱신한다. DES의 경우 1년치 택시 운행을 1초 안에 시뮬레이션할 수 있다. 이 방식은 고정된 (그리고 일반적으로 아주 짧은) 시간만큼 시계를 연속적으로 진행하는 연속 시뮬레이션과 대비된다.

이산 이벤트 시뮬레이션에 대한 설명을 들으며, 바로 턴제 게임<sup>turn-based game</sup>을 떠올렸을 것이다. 게임 상태는 플레이어가 말을 움직일 때만 변경되고, 어디로 움직일지 고민하는 동안 시뮬레이션 시계는 멈춘다. 한편 실시간 게임<sup>real-time game</sup>은 연속 시뮬레이션으로서, 시뮬레이션 시계가 계속 실행되면서 일초에도 몇 번씩 게임 상태가 갱신되므로, 느린 플레이어는 정말 불리한 입장에 놓인다.

이 두 종류의 시뮬레이션은 모두 이벤트 루프에 의해 작동하는 콜백이나 코루틴 등의 객체지향 프로그래밍 기법을 이용해서 다중 스레드나 단일 스레드로 구현할 수 있다. 실시간으로 동시에 발생하는 행동을 표현하기 위해 여러 스레드를 이용해서 연속 시뮬레이션을 구현하는 방법이 단연코 가장 자연스러울 것이다. 한편 코루틴은 DES를 작성하기 위한 추상화에 딱 맞는다. SimPy[^11]는 시뮬레이션의 각 프로세스를 표현하기 위해 하나의 코루틴을 사용하는 파이썬용 DES 패키지다.

>**TIP_** 시뮬레이션 분야에서 '프로세스'라는 용어는 모델 객체의 행동을 말하며, OS 프로세스를 말하는 것이 아니다. 시뮬레이션 프로세스를 OS 프로세스로 구현할 수도 있지만, 모델 객체의 행동은 일반적으로 스레드나 코루틴으로 구현한다.

시뮬레이션에 관심이 있다면 SimPy는 연구해볼 가치가 있다. 그러나 이 절에서는 표준 라이브러리 기능만 사용해서 구현한 아주 간단한 DES를 설명한다. 여기서는 코루틴을 이용해서 동시에 수행되는 행동을 프로그래밍하는 것을 이해하는 데 주안점을 두고 있다. 다음 절은 신경 써서 공부해야 하지만, 마치고 나면 `asyncio`, `Twisted`, `Tornado` 등의 라이브러리가 단일 스레드를 이용해서 어떻게 동시에 일어나는 활동을 관리하는지 깊이 있게 이해할 수 있을 것이다.

### 7.11.2 택시 회사 시뮬레이션

우리가 구현할 시뮬레이션 프로그램 taxi_sim.py에서는 아주 많은 택시를 생성한다. 각 택시는 일정 횟수의 운행을 마친 후 집으로 돌아간다. 택시는 차고를 나와 승객을 찾으면서 '배회'한다. 이 상태는 승객을 태울 때까지 계속되며, 그러고 나서 운행이 시작된다. 승객이 내리고 나면, 택시는 다시 배회 상태로 들어간다.

택시가 배회하고 운행하는 시간은 지수 분포를 이용해서 생성한다. 간결하게 출력하기 위해 분 단위 시간을 사용하지만, 실수형 시간을 이용해서 시뮬레이션할 수도 있다.[^12] 각 상태의 상태 변화는 이벤트로 나타난다. [그림 3]은 프로그램을 실행한 하나의 예를 보여준다.

[그림 3]에서는 세 대의 택시에 의한 운행이 서로 얽혀 있다는 점에 주의하라. 택시 운행을 파악하기 쉽게 직접 화살표를 그려 넣었다. 각 화살표는 승객이 탄 시점에 시작해서 승객이 내린 시점에 끝난다. 이 모습은 동시에 수행하는 행동을 관리하기 위해 코루틴을 사용할 수 있는 방법을 잘 보여준다.

그 외 [그림 3]과 관련해 다음 사항에 주의하라.
● 각 택시는 앞 차가 출발한지 5분 후에 차고를 출발한다.

● 첫 승객을 태우기까지, 택시0은 2분(`time=2`), 택시1은 3분(`time=8`), 택시2는 5분(`time=15`) 걸렸다.

● 택시0은 두 번의 운행을 했다(파란 선). 첫 번째 운행은 `time=2`에 시작해서 `time=18`에 끝났고, 두 번째는 `time=28`에 시작해서 `time=65`에 끝났다(이 시뮬레이션에서 가장 긴 운행).

● 택시1은 네 번의 운행을 하고(녹색 선) `time=110`에 집으로 돌아갔다.

● 택시2는 여섯 번의 운행을 하고(빨간 선) `time=109`에 집으로 돌아갔다. 마지막 운행은 `time=97`에서 시작해서 1분 만에 끝났다.[^13]

● 택시1이 `time=8`에 시작해서 처음 운행하는 동안, 택시2는 `time=10`에 차고를 나와서 두 번의 운행을 마쳤다.

● 이 간단한 시뮬레이션에서, 마지막 이벤트가 `time=110`에 발생하면서 모든 이벤트가 기본 설정된 180분 안에 완료되었다.


_그림 3. 택시 세 대로 간단히 실행해본 taxi_sim.py. `-s 3` 인수는 디버깅이나 시연을 위해 프로그램이 정확히 동일한 결과를 생성할 수 있도록 난수 생성기의 씨앗값<sup>seed value</sup>을 설정한다. 색깔별 화살표는 택스 주행을 알아보기 좋게 해준다_
![](/article7/figs/flup-0703.png)

대기하고 있는 이벤트를 남겨 놓고 시뮬레이션이 끝날 수도 있다. 이벤트가 남아 있는 경우에는 다음과 같은 메시지가 출력된다.
```python
*** end of simulation time: 3 events pending ***
```

taxi_sim.py의 전체 소스 코드는 [번역자의 1판 소스코드 리포지토리](https://github.com/KweonKang/fluent-python-1e/tree/master/16-coroutine)에 있다. 이 장에서는 코루틴을 공부하기 위해 필
요한 부분만 보여준다. 이 코드에서는 코루틴인 `taxi_process()`와 시뮬레이션의 핵심 루프를 실행하는 `Simulator.run()`이 가장 중요하다.

[예제 20]은 `taxi_process()` 코드를 보여준다. 이 코루틴에서는 다른 곳에서 정의된 `compute_delay()` 함수(시간 간격을 분 단위로 반환)와 `Event` 클래스 객체를 사용한다. `Event` 클래스는 다음과 같이 `namedtuple`로 정의되어 있다.

```python
Event = collections.namedtuple('Event', 'time proc action')
```

`Event` 객체 안의 `time`은 이벤트가 발생할 시각, `proc`은 택시 프로세스 객체의 식별자, `action`은 행동을 설명하는 문자열이다.

이제 [예제 20]에 있는 `taxi_process()` 코드를 자세히 살펴보자.

_예제 20. taxi_sim.py: 각 택시의 행동을 구현하는 `taxi_process()` 코루틴_
```python
def taxi_process(ident, trips, start_time=0):  # (1)
    """각 단계 변화마다 이벤트를 생성하며 시뮬레이터에 제어권을 넘긴다."""
    time = yield Event(start_time, ident, 'leave garage')  # (2)
    for i in range(trips):  # (3)
        time = yield Event(time, ident, 'pick up passenger')  # (4)
        time = yield Event(time, ident, 'drop off passenger')  # (5)

    yield Event(time, ident, 'going home')  # (6)
    # 택시 프로세스의 끝 # (7)
```
(1) `taxi_process()`는 각 택시마다 한 번씩 호출되어 택시의 행동을 나타내는 제너레이터 객체를 생성한다. `ident`는 택시 번호(여기서는 0, 1, 2), `trips`는 집에 돌아가기 전에 이 택시가 수행할 운행 횟수, `start_time`은 택시가 차고를 나오는 시각이다.

(2) 첫 번째 생성된 `Event`는 `'leave garage'`다. `yield`를 실행하므로 코루틴이 중단되고, 시뮬레이션의 핵심 루프가 예약된 다음 이벤트를 진행할 수 있게 해준다. 이 프로세스를 다시 활성화해야 할 때가 되면, 핵심 루프는 `time`에 할당된 현재 시각을 보내준다.

(3) 각 운행마다 이 블록이 한 번씩 반복 실행된다.

(4) 승객을 태우는 `Event`가 생성되고, 코루틴은 여기에서 실행을 일시 중단한다. 이 코루틴을 다시 활성화할 때가 되면 핵심 루프가 다시 현재 시각을 보낸다.

(5) 승객을 내리는 `Event`가 생성되고, 다시 활성화되어야 할 시각을 핵심 루프에서 보내줄 때까지 코루틴은 여기에서 실행을 일시 중단한다.

(6) 주어진 횟수만큼 운행을 한 후에는 `for` 루프가 끝나고, 마지막 `'going home'` 이벤트가 생성된다. 코루틴은 여기에서 마지막으로 일시 중단된다. 재활성화될 때 핵심 루프가 코루틴에 시각을 보내지만, 사용하지 않을 것이므로 이 값은 저장하지 않는다.

(7) 코루틴이 끝까지 실행되면 제너레이터 객체가 `StopIteration` 예외를 발생시킨다.

파이썬 콘솔에서 `taxi_process()`를 호출해 여러분이 직접 택시를 운행해볼 수도 있다. 어떻게 하는지 [예제 21]을 보자.

_예제 21. `taxi_process()` 코루틴 돌려보기_
```python
>>> from taxi_sim import taxi_process
>>> taxi = taxi_process(ident=13, trips=2, start_time=0)  (1)
>>> next(taxi)  (2)
Event(time=0, proc=13, action='leave garage')
>>> taxi.send(_.time + 7)  (3)
Event(time=7, proc=13, action='pick up passenger')  (4)
>>> taxi.send(_.time + 23)  (5)
Event(time=30, proc=13, action='drop off passenger')
>>> taxi.send(_.time + 5)  (6)
Event(time=35, proc=13, action='pick up passenger')
>>> taxi.send(_.time + 48)  (7)
Event(time=83, proc=13, action='drop off passenger')
>>> taxi.send(_.time + 1)
Event(time=84, proc=13, action='going home')  (8)
>>> taxi.send(_.time + 10)  (9)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```
(1) `ident=13`으로 설정된 택시를 나타내는 제너레이터 객체를 생성한다. 이 택시는 `t=0`에 시작해서 두 번 운행한다.

(2) 코루틴을 기동시킨다. 그러면 코루틴은 초기 이벤트를 생성한다.

(3) 이제 현재 시각을 코루틴에 보낼(`send`) 수 있다. 콘솔에서 `_` 변수는 마지막 결과값에 바인딩된다. 여기에서는 7을 더했으므로, 택시가 7분 후에 첫 번째 승객을 태우게 된다.

(4) `for` 루프를 처음 실행하면서 생성된다.

(5) 현재 운행 중이므로 `_.time + 23`을 보내면 첫 번째 승객을 위한 운행 시간이 23분 걸린다는 의미다.

(6) 그러고 나서 택시가 5분간 배회한다.

(7) 마지막 운행은 48분 걸린다.

(8) 두 번의 운행을 마친 후 `for` 루프가 종료되어 `'going home'` 이벤트가 생성된다.

(9) 코루틴에 `send()`를 호출하면 코루틴의 실행이 완료된다. 코루틴이 반환될 때 인터프리터가 `StopIteration` 예외를 발생시킨다.

[예제 21]에서는 콘솔을 이용해서 시뮬레이션 핵심 루프를 흉내 냈다. 택시 코루틴이 생성한 `Event` 객체의 `time` 속성을 가져와서, 임의의 숫자를 더한 후, 그 합계를 코루틴을 다시 활성화하기 위해 `taxi.send()`를 호출하는 데 사용했다. 시뮬레이션에서는 `Simulation.run()` 메서드의 핵심 루프에서 `taxi` 코루틴을 다시 활성화시킨다. 시뮬레이션 '시계'는 `sim_time` 변수에 저장되어 이벤트가 생성될 때마다 갱신된다.

`Simulator` 클래스의 인스턴스를 생성하기 위해 taxi_sim.py의 `main()` 함수는 다음과 같이 `taxis` 딕셔너리를 만든다.

```python
    taxis = {i: taxi_process(i, (i + 1) * 2, i * DEPARTURE_INTERVAL)
             for i in range(num_taxis)}
    sim = Simulator(taxis)
```

`DEPARTURE_INTERVAL`은 `5`이며, 예제에서 `num_taxis`가 3이므로, 다음 코드와 동일하다.

```python
    taxis = {0: taxi_process(ident=0, trips=2, start_time=0),
             1: taxi_process(ident=1, trips=4, start_time=5),
             2: taxi_process(ident=2, trips=6, start_time=10)}
    sim = Simulator(taxis)
```

따라서 `taxis` 딕셔너리의 변수에는 서로 다른 매개변수를 가진 별개의 제너레이터 객체 3개가 들어간다. 예를 들어 택시1은 `start_time=5`에 시작해서 운행을 4번 한다. 이 `taxis` 딕셔너리만 있으면 `Simulator` 인스턴스를 생성할 수 있다.

[예제 22]는 `Simulator` 클래스의 `__init__()` 메서드를 보여준다. `Simulator`의 주요 데이터 구조체는 다음과 같다.

>* **`self.events`**

`Event` 객체를 담고 있는 `PriorityQueue` 객체. `PriorityQueue`는 항목을 넣고 나서 정렬된 순서대로 꺼내온다. `Eventnamedtuple` 객체의 경우 `time` 속성으로 정렬한다.
>* **`self.procs`**

각 프로세스 번호를 시뮬레이션의 활성화된 프로세스로 매핑한다. 이 시뮬레이션에서는 택시 하나를 나타내는 제너레이터 객체를 하나의 프로세스로 표현한다. 이 변수는 앞에서 본 `taxis` 딕셔너리에 바인딩된다.

_예제 22. taxi_sim.py: `Simulator` 클래스 초기화 메서드_
```python
class Simulator:

    def __init__(self, procs_map):
        self.events = queue.PriorityQueue()  # (1)
        self.procs = dict(procs_map)  # (2)
```
(1) `PriorityQueue`는 예정된 이벤트를 시간순으로 정렬해서 보관한다.

(2) 딕셔너리형의 `procs_map` 인수를 받지만, 여기에서 다시 딕셔너리 객체를 만들어 사본을 보관한다. 시뮬레이션이 실행되면 집으로 돌아가는 택시들이 `self.procs`에서 제거되지만, 클라이언트가 전달한 객체를 변경하면 안 되기 때문이다.

우선순위 큐<sup>Priority Queue</sup>는 이산 이벤트 시뮬레이션의 핵심 기반이다. 이벤트를 무작위 순서로 만들어 큐에 넣은 후, 예정된 시각순으로 이벤트를 꺼내 와야 하기 때문이다. 예를 들어 다음과 같이 두 개의 이벤트를 큐에 넣는다고 생각해보자.

```python
Event(time=14, proc=0, action='pick up passenger')
Event(time=11, proc=1, action='pick up passenger')
```
즉, 택시0은 14분에 첫 번째 승객을 태우는 반면, 택시1은 11분(10분에 차고에서 나오므로, 차고에서 나온 지 1분 후)에 승객을 태운다. 이 두 이벤트를 우선순위 큐에 넣으면 시뮬레이터의 핵심 루프는 `Event(time=11, proc=1, action='pick up passenger')` 객체를 먼저 가져오게 된다.

이제 시뮬레이션의 핵심 알고리즘인 `Simulator.run()` 메서드를 살펴보자. 이 메서드는 `main()` 함수에서 다음과 같이 `Simulator` 객체를 생성한 직후에 호출된다.

```python
    sim = Simulator(taxis)
    sim.run(end_time)
```

[예제 23]의 `Simulator` 클래스에 번호 설명이 붙어 있지만, `Simulator.run()`의 알고리즘을 상위 수준에서 정리하면 다음과 같다.

>1 택시를 나타내는 프로세스를 반복한다.
>> a. `next()`를 호출해서 각 택시에 대한 코루틴을 기동시킨다. 그러면 각 택시에 대한 첫 번째 이벤트가 생성된다.
>>b. `Simulator`의 `self.events` 큐에 각각의 이벤트를 저장한다.
>
>2 `sim_time < end_time`인 동안 시뮬레이션 핵심 루프를 실행한다.
>>a. `self.events`가 비어 있으면 루프를 빠져나간다.
>>b. `self.events`에서 `current_event`를 가져온다. 이 객체는 `PriorityQueue`에서 가장 작은 `time` 값을 가진 `Event` 객체다.
>>c. `Event`를 출력한다.
>>d. `current_event`의 `time` 속성으로 시뮬레이션 시각을 설정한다.
>>e. `current_event`의 `proc` 속성으로 알아낸 코루틴에 시각을 보낸다. 그러면 코루틴이 `next_event`를 생성한다.
>>f. `next_event`를 `self.events` 큐에 추가해서 스케줄링한다

`Simulator` 클래스의 전체 코드는 [예제 23]과 같다.

예제 23. taxi_sim.py: 기본 뼈대를 갖춘 `Simulator` 이산 이벤트 시뮬레이션 클래스. `run()` 메서드를 중심으로 살펴보라.
```python
class Simulator:

    def __init__(self, procs_map):
        self.events = queue.PriorityQueue()
        self.procs = dict(procs_map)

    def run(self, end_time):  # (1)
        """시간이 끝날 때까지 이벤트를 스케줄링하고 출력한다."""
        # 각 택시의 첫 번째 이벤트를 스케줄링한다.
        for _, proc in sorted(self.procs.items()):  # (2)
            first_event = next(proc)  # (3)
            self.events.put(first_event)  # (4)

        # 시뮬레이션 핵심 루프
        sim_time = 0  # (5)
        while sim_time < end_time:  # (6)
            if self.events.empty():  # (7)
                print('*** end of events ***')
                break

            current_event = self.events.get()  # (8)
            sim_time, proc_id, previous_action = current_event  # (9)
            print('taxi:', proc_id, proc_id * '   ', current_event)  # (10)
            active_proc = self.procs[proc_id]  # (11)
            next_time = sim_time + compute_duration(previous_action)  # (12)
            try:
                next_event = active_proc.send(next_time)  # (13)
            except StopIteration:
                del self.procs[proc_id]  # (14)
            else:
                self.events.put(next_event)  # (15)
        else:  # (16)
            msg = '*** end of simulation time: {} events pending ***'
            print(msg.format(self.events.qsize()))
```
(1) `run()`은 시뮬레이션의 `end_time`을 인수로 받는다.

(2) `sorted()` 함수를 사용해서 키로 정렬된 `self.procs`를 가져온다. 키는 신경쓰지 않으므로 `_`를 할당한다.

(3) `next(proc)`은 각 코루틴을 첫 번째 `yield`까지 실행해서 데이터를 전송할 준비를 한다. 이때 `Event`가 생성된다.

(4) 각 이벤트를 `self.events` 우선순위 큐에 넣는다. [예제 20]에서 본 것처럼 각 택시에 처음 보내는 이벤트는 `'leave garage'`다.

(5) 시뮬레이션 시계 `sim_time`을 0으로 설정한다.

(6) 시뮬레이션의 핵심 루프다. `sim_time`이 `end_time`보다 작으면 반복한다.

(7) 큐 안에 남아 있는 이벤트가 없을 때도 핵심 루프를 빠져나온다.

(8) 우선순위 큐에서 `time` 값이 가장 작은 이벤트를 가져와서 `current_event`에 저장한다.

(9) `Event` 데이터를 언패킹한다. 이 코드는 이벤트가 발생한 시각을 반영해서 시뮬레이션 시계인 `sim_time`을 갱신한다.[^14]

(10) 택시의 ID에 따라 들여 써서 이벤트를 출력한다.

(11) `self.procs` 딕셔너리에서 활성화된 택시에 대한 코루틴을 가져온다.

(12) 이전 행동(`'pick up passenger'`나 `'drop off passenger'`)에 `compute_duration()`을 호출한 결과에 `sim_time`을 더해서 다음 활성화 시각을 계산한다.

(13) 택시 코루틴에 시각을 전송한다. 코루틴은 `next_event`를 반환하거나, 실행이 완료된 경우 `StopIteration` 예외를 발생시킨다.

(14) `StopIteration` 예외가 발생하면 `self.procs` 딕셔너리에서 해당 코루틴을 제거한다.

(15) 예외가 발생하지 않으면 `next_event`를 큐에 넣는다.

(16) 시뮬레이션 시간이 초과되어서 루프가 종료된 경우에는 대기 중인 이벤트 수를 출력한다.


[예제 23]의 `Simulator.run()` 메서드에는 `if`가 아닌 문에서 `else` 블록을 사용한 코드가 두 군데 있다(<본책> 18장 참조).
● 핵심 `while` 루프에 `else` 블록이 연결되어 있다. 이벤트가 소진된 경우가 아니라 `end_time`에 도달한 경우이므로 별도의 메시지를 출력한다.

● `while` 루프 안의 마지막 `try` 문은 현재 택시 프로세스에 `next_time`을 전송해서 `next_event`를 가져온다. 그러고 나서 이 연산이 성공하면 `else` 블록에서 `next_event`를 `self.events` 큐에 추가한다.

필자는 이런 `else` 블록이 없었다면 `Simulator.run()` 코드의 가독성이 약간 떨어졌을 것이라고 생각한다.

이 예제에서는 이벤트를 처리하고 코루틴에 데이터를 전송해서 코루틴을 실행시키는 루프를 보여주고자 했다. 이 구조는 <본책> 21장에서 설명할 `asyncio`의 기반 개념이 된다.

이제 제너릭 코루틴 자료형에 대한 설명으로 코루틴 설명을 마치고자 한다.

## 7.12 고전적 코루틴을 위한 제너릭 자료형 힌트

<본책> 15장에서 `typing.Generator`가 역변성 형 매개변수를 가진 얼마 안되는 표준 라이브러리 자료형 중 하나라고 이야기했었다. 이제 고전적 코루틴을 알아보았으니 이 제너릭 형에 대해 감을 잡을 준비가 되었다.

값을 생성만 하고 `None` 이외의 값은 전달하지 않는 제너레이터의 경우 어노테이션에 권장되는 자료형은 `Iterator[T_co]`이다.

자신의 명칭에도 불구하고 `typing.Generator`는 사실 값을 생성할 뿐만 아니라 `send()`를 통해 값을 받고, `StopIteration(value)` 해킹을 통해 값도 반환할 수 있는 고전적 코루틴을 어노테이트하기 위해 사용된다.

파이썬 3.6의 typing.py 모듈에 `typing.Generator`는 다음과 같이 [선언](https://github.com/python/cpython/blob/6f743e7a4da904f61dfa84cc7d7385e4dcc79ac5/Lib/typing.py#L2060)되어 있다.[^15]
```python
T_co = TypeVar('T_co', covariant=True)
V_co = TypeVar('V_co', covariant=True)
T_contra = TypeVar('T_contra', contravariant=True)

# 중략

class Generator(Iterator[T_co], Generic[T_co, T_contra, V_co],
                extra=_G_base):
```

이 제너릭 자료형 선언은 Generator 형 힌트가 다음 예제처럼 세 개의 형 인수를 요구한다는 것을 의미한다.

```python
my_coro : Generator[YieldType, SendType, ReturnType]
```

형식 인수의 형변성을 보아, `YieldType`과 `ReturnType`은 공변적이지만, `SendType`은 역변적임을 알 수 있다.

이유를 이해하려면, `YieldType`과 `ReturnType`이 '출력' 자료형임을 기억하라. 둘 다 코루틴 객체에서 나오는 데이터를 설명한다. 즉 코루틴 객체로 사용될 때의 제너레이터 객체를 의미하는 것이다.

실수를 생성하는 코루틴을 기대하는 코드라면 정수를 생성하는 코루틴도 받을 수 있음을 생각하면, 이 인수들이 공변적인 것이 타당하다. 그렇기 때문에 `Generator`는 자신의 `YieldType` 인수에 대해 공변적이다. 또 다른 공변적인 `ReturnType` 인수에도 똑같은 이유가 적용된다.

<본책> 15.7.4절에 소개한 표기법을 이용하면 첫 번째와 세 번째 인수의 공변성은 나란히 `:>` 기호로 표현된다.

```
                       float :> int
Generator[float, Any, float] :> Generator[int, Any, int]
```

`YieldType`과 `ReturnType`은 형변성 첫 번째 규칙의 예다.

반면 `SendType`은 '입력' 인수다. 즉, 코루틴 객체의 `send()` 메서드 인수의 형이다. 코루틴에 실수를 보내려는 코드는 `int`를 받는 코루틴을 사용할 수 없다. `int`가 `float`의 슈퍼타입이 아니기 때문이다. 달리 말하면, `float`은 `int`와 일치하지 않는다. 그러나 `complex` 형을 `SendType`으로 하는 코루틴은 사용할 수 있다. `complex`가 `float`의 슈퍼타입이므로, `float`이 `complex`와 일치하기 때문이다.

`:>` 표기법을 이용하면 두 번째 인수의 역변성이 잘 보인다.
```
                     float :> int
Generator[Any, float, Any] <: Generator[Any, int, Any]
```

이 것은 형변성 두 번째 규칙의 예다.

즐거운 형변성에 대한 설명으로 이 글을 마치고자 한다.

## 7.13 요약

귀도 반 로섬은 제너레이터를 이용해서 코드를 작성하는 세 가지 스타일이 있다고 했다.

>'풀' 스타일(반복자), '푸시' 스타일(이동 평균 예제), '작업’이 있다(데이비드 비즐리의 코루틴 튜토리얼을 읽어보았는가?... )[^16]

이 글에서는 '푸시' 스타일로 사용되는 고전적 코루틴과 간단한 '작업'(시뮬레이션 예제에서의 택시 프로세스)을 설명했다. 네이티브 코루틴은 여기에서 설명한 고전적 코루틴에서 진화했다.

이동 평균 예제는 일반적인 코루틴 사용법(받은 항목을 처리하는 누산기로 사용)을 잘 보여준다. 코루틴을 자동으로 기동해주는 데커레이터를 이용하면 코루틴을 편리하게 사용할 수 있다. 그러나 기동해주는 데커레이터와 호환되지 않는 코루틴도 있다는 점을 명심하라. 특히 `yield from subgenerator()`는 하위 제너레이터가 기동되지 않았다고 가정하고 자동으로 기동해주므로 주의해야 한다.

누산기 코루틴은 `send()` 메서드가 호출될 때마다 중간 결과를 생성할 수 있지만, 값을 반환할 수 있을 때 훨씬 더 유용해진다. 코루틴이 값을 반환하는 기능은 PEP 380을 구현한 파이썬 3.3에 추가되었다. 제너레이터 안에서 `return the_result` 문을 실행하면 `StopIteration(the_result)` 예외가 발생되어 호출자가 예외 객체의 `value` 속성에서 `the_result`를 가져오는 방법을 예를 들어 알아보았다. 코루틴 결과를 가져오기 위해 다소 성가신 방법이긴 하지만 PEP 380에 소개된 `yield from` 구문이 자동으로 처리해준다.

간단한 반복형을 이용해 `yield from`에 대해 설명하고 난 후, `yield from`을 제대로 사용할 때의 세 가지 구성 요소(대표 제너레이터, 하위 제너레이터, 클라이언트)를 사용한 예제를 다루었다. 대표 제너레이터는 자신의 본체 안에서 `yield from`을 사용하고, 하위 제너레이터는 `yield from`으로 활성화되며, 클라이언트 코드가 대표 제너레이터의 `yield from`으로 설정된 통로를 통해 하위 제너레이터에 값을 전송함으로써 전반적인 운영을 주도한다. 그리고 PEP 380에서 영어 및 파이썬과 비슷한 의사코드로 설명한 `yield from`의 내부 작동 과정을 자세히 살펴보았다.

이산 이벤트 시뮬레이션 예제를 통해 동시성을 지원하기 위해 스레드와 콜백을 대신해서 제너레이터를 사용하는 방법을 보여주었다. 간단하기는 하지만, 택시 시뮬레이션을 통해 `Tornado`나 `asyncio` 같은 이벤트 주도 프레임워크가 단일 스레드에서 핵심 루프를 사용해서 코루틴을 병렬로 실행할 수 있는지 감을 잡을 수 있었다. 코루틴을 사용하는 이벤트 주도 프로그래밍 환경에서는 코루틴이 반복적으로 핵심 루프에 제어권을 넘겨주어 핵심 루프가 다른 코루틴을 활성화하고 실행할 수 있게 해줌으로써 작업을 동시에 실행한다. 이런 방식의 협업 멀티태스킹 환경에서는 코루틴이 중앙 스케줄러에 자발적으로 제어권을 넘겨준다. 이와 반대로, 스레드는 선점형 멀티태스킹을 구현한다. 선점형 멀티태스킹은 스케줄러가 언제든 스레드를 중단하고 다른 스레드를 실행할 수 있다.

[^1]: 마이클 디베르나도<sup>Michael DiBernado</sup> 편집, 제시 지류 데이비스<sup>A. Jesse Jiryu Davis</sup>와 귀도 반 로섬<sup>Guido van Rossum</sup>의 ['`asyncio` 코루틴을 이용한 웹 크롤러' 장](https://aosabook.org/en/500L/a-web-crawler-with-asyncio-coroutines.html#coroutines)

[^2]: 이 상태는 다중스레드 애플리케이션에서만 볼 수 있다. 제너레이터 객체가 자신에게 `inspect.getgeneratorstate()` 함수를 호출할 때도 반환되지만, 그리 유용한 방법은 아니다.

[^3]: 이 예제는 제이콥 홈<sup>Jacob Holm</sup>이 `Python-ideas` 리스트에 올린 ['yield from: 완료를 보장한다<sup>Yield-From: Finalization guarantees</sup>' 메시지](https://mail.python.org/pipermail/python-ideas/2009-April/003841.html)에서 힌트를 얻었다. [메시지 003912](https://mail.python.org/pipermail/python-ideas/2009-April/003912.html)에서 제이콥 홈은 자신의 생각을 더 자세히 설명한다.

[^4]: 웹에서 이와 유사한 데커레이터를 많이 볼 수 있다. 이 데커레이터는 샤오빈 탕<sup>Chaobin Tang</sup>이 만든 ['코루틴으로 구성된 ActiveState의 Pipeline 비법'](https://code.activestate.com/recipes/578265-pipeline-made-of-coroutines/)에서 가져와 응용한 것이다. 샤오빈 탕도 데이비드 비즐리로부터 이 코드에 대한 영감을 얻었다고 한다.

[^5]: [ipython-yf](https://github.com/tecki/ipython-yf)라는 iPython 확장은 iPython 콘솔에서 `yield from`을 직접 평가할 수 있게 해준다. ipython-yf는 `asyncio`를 이용해 비동기 코드를 테스트하기 위해 사용된다. 파이썬 버그 트래커에서 ['Issue #22412: asyncio가 활성화된 명령행으로<sup>Towards an asyncio-enabled command line</sup>'](http://bugs.python.org/issue22412)를 참조하라.

[^6]: [그림 2]의 그림은 [폴 소콜로브스키<sup>Paul Sokolovsky</sup>의 다이어그램](https://www.fluentpython.com/extra/classic-coroutines/images/yield-from.pdf)에서 영감을 얻었다.

[^7]: [await 표현식<sup>await expression</sup> 절](https://peps.python.org/pep-0492/#await-expression)

[^8]: `Python-Dev` 리스트 ['PEP 380 하위 제너레이터에 `yield from` 적용하기<sup>yield from a subgenerator</sup>' 메시지에 대한 댓글](https://mail.python.org/pipermail/python-dev/2009-March/087385.html) (2009년 3월 21일)

[^9]: 2009년 4월 5일 `Python-ideas` 리스트에 보낸 [메시지](https://mail.python.org/pipermail/python-ideas/2009-April/003954.html)에서 닉 코글란<sup>Nick Coghlan</sup>은 `yield from`이 암묵적으로 기동하는 것이 좋은 생각인지에 대한 의문을 제기했다.

[^10]: [PEP 342](https://peps.python.org/pep-0342/)의 '동기<sup>Motivation</sup>'절 도입부

[^11]: [SimPy 공식 문서](https://simpy.readthedocs.io/en/latest/)를 참조하라. 잘 알려져 있지만 시뮬레이션과는 상관없는 기호 수학 라이브러리인 [SymPy](https://www.sympy.org/en/index.html)와 혼동하지 않도록 주의하라.

[^12]: 필자는 택시사업에 대해 잘 모르므로, 이 예는 단지 하나의 예로 생각하기 바란다. 지수 분포는 DES에서 널리 사용된다. 아주 짧은 운행도 볼 수 있을 것이다. 이는 비오는 날 승객이 다음번 골목길까지 가기 위해 택시를 이용한 경우라고 생각하라. 비오는 날에 택시를 탈 수 있다니, 정말 이상적인 도시이긴 하다.

[^13]: 누군가 지갑을 놓고 나온 모양이다.

[^14]: 일반적으로 이산 이벤트 시뮬레이션에서 이런 형태를 주로 볼 수 있다. 시뮬레이션 시계를 루프를 반복할 때마다 고정된 크기로 증가시키지 않고, 이벤트가 발생할 때마다 갱신한다.

[^15]: 파이썬 버전 3.7 이후부터 `collections.abc` 안의 추상 베이스 클래스(ABC)에 대응되는 `typing.Generator`와 여타 자료형들은 해당 ABC에 의해 래핑되므로 typing.py 소스 파일에서 이들의 제너릭 파라미터들을 볼 수 없다. 그렇기 때문에 여기서 파이썬 3.6 소스 코드를 언급한 것이다.

[^16]: `Python-ideas` 메일링 리스트의 ['yield from: 완료를 보장한다<sup>Yield-From: Finalization guarantees</sup>’](https://mail.python.org/pipermail/python-ideas/2009-April/003884.html)에서 발췌. 귀도가 말한 데이비드 비즐리의 튜토리얼은 ['코루틴 및 동시성에 대한 흥미로운 강의<sup>A Curious Course on Coroutines and Concurrency</sup>’](https://www.dabeaz.com/coroutines/)다.