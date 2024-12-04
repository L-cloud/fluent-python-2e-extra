# 6. 함수 매개변수의 인트로스펙션

내용:

[6.1 함수 인트로스펙션 개요](./article6.md#61-함수-인트로스펙션-개요)

[6.2 매개변수에 대한 정보 읽기](article6.md#62-매개변수에-대한-정보-읽기)

[6.3 뒷이야기: Bobo에 관하여](article6.md#63-뒷이야기-bobo에-관하여)

## 6.1 함수 인트로스펙션 개요

파이썬 함수는 특징을 모두 갖춘 완전한 객체다. 그렇기 때문에 다음 예제에서 보는 것처럼 `__doc__` 등의 속성을 갖고 있다.

```python
>>> def factorial(n):
...     """returns n!"""
...     return 1 if n < 2 else n * factorial(n - 1)
...
>>> factorial.__doc__
'returns n!'
```

파이썬 콘솔에서 `help(factoria)`를 실행하면 함수 시그너처와 `__doc__` 설명이 출력된다.

함수는 `__doc__` 이외에도 많은 속성을 갖고 있다. `factorial`에 대해 `dir()` 함수를 실행하면 다음과 같이 출력된다.

```python
>>> dir(factorial)
['__annotations__', '__builtins__', '__call__', '__class__',
'__closure__', '__code__', '__defaults__', '__delattr__',
'__dict__', '__dir__', '__doc__', '__eq__', '__format__',
'__ge__', '__get__', '__getattribute__', '__globals__',
'__gt__', '__hash__', '__init__', '__init_subclass__',
'__kwdefaults__', '__le__', '__lt__', '__module__',
'__name__', '__ne__', '__new__', '__qualname__',
'__reduce__', '__reduce_ex__', '__repr__', '__setattr__',
'__sizeof__', '__str__', '__subclasshook__']
```

이 속성들 대부분은 일반적으로 파이썬 객체들이 공통적으로 갖고 있다. 이 절에서는 함수를 객체로서 다루는 것과 관련된 속성들을 알아본다. 먼저 `__dict__`를 보자.

평범한 사용자 정의 클래스의 인스턴스들이 그렇듯이 함수는 `__dict__` 속성을 이용해 자신에 할당된 속성을 저장한다. 기본적인 형태의 설명을 추가하기에 좋다. 일반적으로 함수에 무작위적인 속성을 추가하는 일은 흔치 않지만, 장고<sup>Django</sup>는 이 기능을 사용하는 프레임워크 중 하나다. 예를 들어 '장고 관리자 사이트' 문서(https://docs.djangoproject.com/en/5.1/ref/contrib/admin/#the-display-decorator)에서 설명하는 `short_description`, `boolean`, `admin_order_field` 속성이 그렇다. 장고 문서에서 가져온 다음 예제는 메서드를 선택했을 때 장고 관리자 UI의 레코드 목록에 보일 설명을 결정하기 위해 `Model` 서브클래스의 메서드에 `short_description` 속성을 추가한다.

```python
    def is_published(self, obj):
        return obj.publish_date is not None
    is_published.short_description = 'Is Published?'
```

이제 일반 파이썬 사용자 정의 객체가 아니라 함수에 고유한 속성들을 살펴보자. 두 집합의 차집합을 구하면 함수에 고유한 속성들의 목록을 간단히 알아낼 수 있다(예제 1).

_예제 1. 일반 인스턴스에 없는 함수의 속성 나열하기_

```python
>>> class C: pass  # (1)
>>> obj = C()  # (2)
>>> def func(): pass  # (3)
>>> sorted(set(dir(func)) - set(dir(obj))) # (4)
['__annotations__', '__call__', '__closure__', '__code__', '__defaults__',
'__get__', '__globals__', '__kwdefaults__', '__name__', '__qualname__']
>>>
```

(1) 최소한의 사용자 정의 클래스

(2) 이 클래스의 인스턴스 생성

(3) 최소한의 함수 생성

(4) 차집합을 이용해 클래스의 인스턴스에는 없지만 함수에는 존재하는 속성들을 정렬한 리스트를 생성한다.

[표 1]은 [예제 1]에 나열된 속성들을 간략히 정리해 설명한다.

_표 1. 사용자 정의 함수의 속성들_

| **속성명** | **자료형** | **설명** |
|---|---|---|
| `__annotations__` | `dict` | 매개변수 및 반환값에 대한 주석 |
| `__call__` | `method` | 콜러블 객체 프로토콜에 따른 `()` 연산자 구현 |
| `__closure__` | `tuple` | 자유 변수 등 함수 클로저(`None`인 경우가 종종 있다) |
| `__code__` | `code` | 바이트코드로 컴파일된 함수 메타데이터 및 함수 본체 |
| `__defaults__` | `tuple` | 형식 매개변수의 기본값 |
| `__get__` | `method` | 읽기 전용 디스크립터 프로토콜 구현(본책 23장 '속성 디스크립터' 참조) |
| `__globals__` | `dict` | 함수가 정의된 모듈의 전역 변수들 |
| `__kwdefaults__` | `dict` | 키워드 전용 형식 매개변수의 기본값 |
| `__name__` | `str` | 함수명 |
| `__qualname__` | `str` | `random.choice()`와 같은 전체 함수 명칭(PEP 3155(https://peps.python.org/pep-3155/) 참조)

## 6.2 매개변수에 대한 정보 읽기

함수 인트로스펙션을 활용한 흥미로운 사례는 Bobo HTTP 마이크로 프레임워크(https://github.com/zopefoundation/bobo)에서 볼 수 있다. 어떻게 작동하는지 살펴보기 위해 Bobo 튜터리얼 'Hello World' 애플리케이션을 수정한 [예제 2]를 보자.

>**NOTE_** `Bobo`는 이미 1997년부터 파이썬 웹 프레임워크에 있는 보일러플레이트 코드를 줄이기 위해 매개변수 인트로스펙션의 사용을 개척했기에 여기에서 설명한다. 이제는 방법이 일반적으로 사용된다. 이 개념을 사용하는 최신 프레임워크 사례로는 FastAPI(https://fastapi.tiangolo.com/)가 있다.

_예제 2. `Bobo`는 `hello()`가 `person` 인수를 요구한다는 것을 알고 있으며 인수를 HTTP 요청에서 가져온다._

```python
import bobo

@bobo.query('/')
def hello(person):
    return f'Hello {person}!'
```

`bobo.query()` 데커레이터는 `hello()`와 같은 평범한 함수와 프레임워크에서 제공하는 요청 처리 메커니즘을 결합시킨다. 데커레이터는 본책 9장 '데커레이터와 클로저'에서 설명하며, 이 예제에서는 설명하지 않는다. 여기서 중요한 점은 `Bobo`가 `hello()` 함수의 내부를 조사해 이 함수가 작동하려면 `person` 이라는 매개변수가 필요하다는 것을 알아낸다는 것이다. 그러고 나서 요청에서 해당 이름의 매개변수를 가져와 `hello()`에 전달하므로 프로그래머는 요청 객체를 건드릴 필요가 전혀 없다.

`Bobo`를 설치하고 개발 서버를 [예제 2]의 스크립트로 지정한 후(예를 들어 `bobo -f hello.py`), http://localhost:8080/ URL에 요청하면 403 HTTP 코드와 함께 `‘Missing form variable person’` 메시지가 나온다. `hello`를 호출하려면 `person` 인수가 필요한데, 그런 이름이 요청에 들어 있지 않다는 것을 `Bobo`가 알고 있기 때문이다. [예제 3]은 터미널 세션에서 `curl`을 이용해 이 과정을 보여준다.

_예제 3. 요청에서 빠진 함수 인수가 있으면 `Bobo`는 `‘403 forbidden response’`를 발생시킨다. HTTP 헤더를 표준 출력 화면에 보여주기 위해 `curl -i` 명령을 사용했다._

```bsh
$ curl -i http://localhost:8080/
HTTP/1.0 403 Forbidden
Date: Mon, 31 May 2021 16:34:19 GMT
Server: WSGIServer/0.2 CPython/3.9.5
Content-Type: text/html; charset=UTF-8
Content-Length: 103

<html>
<head><title>Missing parameter</title></head>
<body>Missing form variable person</body>
</html>
```

그러나 http://localhost:8080/?person=Jim URL로 요청하면 [예제 4]와 같이 `‘Hello Jim!’` 문자열로 응답한다.

_예제 4. `OK` 응답을 받으려면 `person` 매개변수를 전달해야 한다_

```bsh
$ curl -i http://localhost:8080/?person=Jim
HTTP/1.0 200 OK
Date: Mon, 31 May 2021 16:35:40 GMT
Server: WSGIServer/0.2 CPython/3.9.5
Content-Type: text/html; charset=UTF-8
Content-Length: 10

Hello Jim!
```

함수에 어떤 매개변수가 필요한지, 매개변수에 기본값이 있는지 없는지 `Bobo`는 어떻게 알 수 있을까?

함수 객체 안의 `__defaults__` 속성에는 위치<sup>positional</sup> 인수와 키워드<sup>keyword</sup> 인수의 기본값을 가진 튜플이 들어 있다. 키워드 전용 인수의 기본값은 `__kwdefaults__` 속성에 들어 있다. 그러나 인수명은 `__code__` 속성에 들어 있는데, 이 속성은 여러 속성을 담고 있는 `code` 객체를 가리킨다. `clip()` 함수는 텍스트 문자열을 공백에서 잘라내 가능하면 `max_len`보다 작거나 같은 결과를 반환한다. 예제 코드 리포지토리(https://github.com/fluentpython/example-code-2e/blob/master/15-more-types/clip_annot.py)에 있는 clip_annot.py의 doctest에는 이 함수가 어떻게 작동해야 하는지 보여준다. 여기에서는 함수 본체보다는 시그너처에 집중해서 살펴본다.

이 속성들을 사용하는 방법을 알아보기 위해 [예제 5]의 clip_annot.py 모듈에 들어 있는 `clip()` 함수를 살펴보자.

_예제 5. 원하는 길이 가까이에 있는 공백에서 잘라서 문자열을 단축하는 함수_

```python
def clip(text, max_len=80):
    """max_len 앞이나 뒤의 마지막 공백에서 잘라낸 텍스트를 반환한다"""
    text = text.rstrip()
    if len(text) <= max_len or ' ' not in text:
        return text
    end = len(text)
    space_at = text.rfind(' ', 0, max_len + 1)
    if space_at >= 0:
        end = space_at
    else:
        space_at = text.find(' ', max_len)
        if space_at >= 0:
            end = space_at
    return text[:end].rstrip()
```

[예제 6]은 [예제 5]에 나온 `clip()` 함수의 `__defaults__`, `__code__.co_varnames`, `__code__.co_argcount` 속성값을 보여준다.

_예제 6. 함수 인수에 대한 정보 추출하기_

```python
>>> from clip import clip
>>> clip.__defaults__
(80,)
>>> clip.__code__  # doctest: +ELLIPSIS
<code object clip at 0x...>
>>> clip.__code__.co_varnames
('text', 'max_len', 'end', 'space_at')
>>> clip.__code__.co_argcount
2
```

정보가 그다지 사용하기 편하게 배치되어 있지 않다. 인수명이 `__code__.co_varnames`에 들어있지만, 여기에는 함수 본체에서 생성한 지역 변수명도 들어 있다. 따라서 앞에서 `__code__.co_argcount` 개의 변수가 인수명이다. 이때 `__code__.co_argcount`에는 앞에 `*`나 `**`가 붙은 인수가 포함되어 있지 않다. 인수의 기본값은 `__defaults__` 튜플의 위치에 따라 알 수 있으므로, 인수를 뒤에서부터 추적하면서 각각의 기본값과 대응시켜야 한다. 위 예제에는 2개의 인수 `text`와 `max_len`이 있고 기본값이 80이므로, 이 기본값은 마지막 인수인 `max_len`에 해당한다. 처리과정이 보기 좋지 않다.

다행히도 `inspect` 모듈을 사용하면 더 깔끔하게 처리할 수 있다. [예제 7]을 보자.

_예제 7. 함수 시그너처 추출하기_

```python
>>> from clip import clip
>>> from inspect import signature
>>> sig = signature(clip)
>>> sig
<Signature (text, max_len=80)>
>>> str(sig)
'(text, max_len=80)'
>>> for name, param in sig.parameters.items():
...     print(param.kind, ':', name, '=', param.default)
...
POSITIONAL_OR_KEYWORD : text = <class 'inspect._empty'>
POSITIONAL_OR_KEYWORD : max_len = 80
```

이 결과가 훨씬 더 보기 좋다. `inspect.signature()`는 `inspect.Signature` 객체를 반환하며, 이 객체에 들어 있는 `parameters` 속성을 이용하면 정렬된 매핑 객체 `inspect.Parameter`를 읽을 수 있다. 각 `Parameter` 객체 안에는 `name`, `default`, `kind` 등의 속성이 들어 있다. `inspect._empty`라는 특별한 값은 매개변수에 기본값이 없음을 나타낸다. `None`이 정당하면서도 널리 사용되는 기본값인 점을 고려하면 이런 특별한 값이 있는 게 당연하다.

`kind` 속성은 `_ParameterKind` 클래스에 정의된 다음 다섯 가지 값 중 하나를 가진다.

| 값 | 설명 |
|---|---|
| `POSITIONAL_OR_KEYWORD` | 위치 인수나 키워드 인수로 전달할 수 있는 매개변수(파이썬 함수 매개변수 대부분이 여기에 속한다.) |
| `VAR_POSITIONAL` | 위치 매개변수의 튜플 |
| `VAR_KEYWORD` | 키워드 매개변수의 딕셔너리 |
| `KEYWORD_ONLY` | 키워드 전용 매개변수(파이썬 3) |
| `POSITIONAL_ONLY` | 위치 전용 매개변수. 현재 파이썬 함수 선언 구문에서는 지원되지 않지만, 키워드로 전달한 매개변수를 받지 않는 `divmod()`처럼 C 언어로 구현된 기존 함수가 여기에 속한다.|

`name`, `default`, `kind` 외에 `inspect.Parameter` 객체에는 `annotation` 속성이 있다. 이 속성은 일반적으로 `inspect._empty`지만, 파이썬 3에서 제공하는 새로운 애너테이션 구문을 통해 제공된 함수 시그너처 메타데이터가 들어갈 수 있다(애너테이션에 대해서는 본책 8장. '함수에서의 자료형 힌트'를 참조하라).

`inspect.Signature` 객체에는 `bind()` 메서드가 정의되어 있다. `bind()` 메서드는 임의 개수의 인수를 받고, 인수를 매개변수에 대응시키는 일반적인 규칙을 적용해서 그것을 시그너처에 들어있는 매개변수에 바인딩한다. `bind()` 메서드는 프레임워크에서 실제 함수를 호출하기 전에 인수를 검증하기 위해 사용할 수 있다. [예제 8]은 인수를 검증하는 예를 보여준다.

_예제 8. 본책 [예제 7-9]의 `tag()`에서 가져온 함수 시그너처를 인수들의 딕셔너리에 바인딩하기_

```python
>>> import inspect
>>> sig = inspect.signature(tag)  #(1)
>>> my_tag = {'name': 'img', 'title': 'Sunset Boulevard',
...           'src': 'sunset.jpg', 'class_': 'framed'}
>>> bound_args = sig.bind(**my_tag)  #(2)
>>> bound_args
<BoundArguments (name='img', class_='framed',
  attrs={'title': 'Sunset Boulevard', 'src': 'sunset.jpg'})>  #(3)
>>> for name, value in bound_args.arguments.items():  #(4)
...     print(name, '=', value)
...
name = img
class_ = framed
attrs = {'title': 'Sunset Boulevard', 'src': 'sunset.jpg'}
>>> del my_tag['name']  #(5)
>>> bound_args = sig.bind(**my_tag)  #(6)
Traceback (most recent call last):
  ...
TypeError: missing a required argument: 'name'
```

(1) 본책 [예제 7-9]에서 정의한 `tag()` 함수의 시그너처를 가져온다.
(2) 인수들을 담은 딕셔너리를 `bind()` 메서드에 전달한다.
(3) `inspect.BoundArguments` 객체가 생성된다.
(4) `dict` 형인 `bound_args.arguments` 안에 들어 있는 항목들을 반복해서 인수의 이름과 값을 출력한다.
(5) `my_tag`에서 필수 인수인 `name`을 제거한다.
(6) 이제 `sig.bind(**my_tag)`를 호출해서 다시 바인딩하면 `name` 매개변수가 빠져 있다는 메시지를 담은 `TypeError`가 발생한다.

`inspect`를 사용하는 이 예제를 통해, 함수 호출 시 인수를 매개변수에 바인딩하기 위해 인터프리터가 사용하는 파이썬 데이터 모델 메커니즘이 작동하는 방식을 알 수 있다. 프레임워크 및 도구(IDE 등)는 이 정보를 이용해 코드를 검증할 수 있다.

>**WARNING_** `inspect` 모듈은 'PEP 484 - 자료형 힌트<sup>Type Hint</sup>' 이전부터 존재했다. 자료형 힌트를 런타임에 제대로 인스펙트하는 것은 더 복잡하며, 파이썬 3.9까지는 잘 지원되지 않는다.

## 6.3 뒷이야기: `Bobo`에 관하여

필자가 파이썬에 입문하게 된 것은 전적으로 `Bobo` 때문이었다. 펄과 자바를 시도해본 후 객체지향 방식으로 웹 애플리케이션을 구현할 방법을 찾다가 `Bobo`를 발견했다. 1998년에 필자가 참여한 최초의 파이썬 웹 프로젝트(US에 본사를 둔 미디어 회사 International Data Group의 브라질 협력사 IDG Now라는 IT 뉴스 포털 프로젝트였다)에 `Bobo`를 사용했다.

1997년 `Bobo`는 라우트를 설정할 필요 없이 URL을 객체의 계층 구조에 직접 매핑하는 객체 출판 개념을 개척했다. 필자는 이 개념의 아름다움에 푹 빠졌다. 게다가 `Bobo`는 요청을 처리하기 위해 사용되는 메서드나 함수의 시그너처를 분석해서 자동으로 HTTP 쿼리를 처리하는 기능을 갖고 있다.

`Bobo`는 짐 펄톤<sup>Jim Fulton</sup>에 의해 개발되었는데, 짐 펄톤은 Plone CMS, SchoolTool, ERP5 등 대규모 파이썬 프로젝트의 기반이 된 `Zope` 프레임워크를 개발할 때 그의 리더 역할 덕분에 ‘The Zope Pope’로 알려져 있다. 그리고 짐 펄톤은 파이썬에서 쉽게 사용할 수 있고 ACID(원자성<sup>atomicity</sup>, 일관성<sup>consistency</sup>, 격리성<sup>isolation</sup>, 내구성<sup>durability</sup>)를 제공하도록 설계된 객체 트랜잭션 데이터베이스인 ZODB(Zope Object Database)의 개발자이기도 하다.

`Bobo`는 `Zope`의 핵심이다. 1998년 후반에 `Zope`이 출시되었을 때 필자는 아주 반가웠다. `Bobo`를 소프트웨어 개발 아르바이트에 사용하고 싶었기 때문이다. 그러나 잘 코미디 단체 이름을 딴 잘 알려지지 않은 언어와 포루투갈어로 '바보'를 뜻하는 무명의 프레임워크를 브라질에서 판매하기 쉽지 않았기 때문이다. 알려지지 않기는 `Zope`도 마찬가지지만, 적어도 바보같은 이름은 아니었다. 게다가 아주 멋진 ZODB를 포함하고 있다.

그 후 짐 펄톤은 WSGI와 최신 파이썬(파이썬 3 포함)을 지원하기 위해 `Bobo`를 처음부터 다시 개발했다.