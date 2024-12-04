# 5. 약한 참조

내용:

[5.1 들어가며](article5.md#51-들어가며)

[5.2 `WeakValueDictionary` 촌극](article5.md#52-weakvaluedictionary-촌극)

[5.3 약한 참조의 한계](article5.md#53-약한-참조의-한계)


## 5.1 들어가며

객체가 메모리에 유지되거나 유지되지 않도록 만드는 것은 참조의 존재 여부다. 객체 참조 카운트가 0이 되면 가비지 컬렉터는 해당 객체를 제거한다. 그러나 불필요하게 객체를 유지시키지 않으면서 객체를 참조할 수 있으면 도움이 되는 경우가 종종 있다. 캐시가 대표적인 경우다.

약한 참조<sup>weak reference</sup>는 참조 카운트를 증가시키지 않고 객체를 참조한다. 참조의 대상인 객체를 참조 대상<sup>referent</sup>이라고 한다. 따라서 약한 참조는 참조 대상이 가비지 컬렉트되는 것을 방지하지 않는다고 말한다.

약한 참조는 캐시 애플리케이션에서 유용하게 사용된다. 캐시가 참조하고 있다고 해서 캐시된 객체가 계속 남아 있기 원치 않기 때문이다.

[예제 1]은 weakref.ref 인스턴스를 호출해서 참조 대상에 접근하는 방법을 보여준다. 객체가 살아 있으면 약한 참조 호출은 참조된 객체를 반환하고, 그렇지 않으면 `None`을 반환한다.

>**TIP_** [예제 1]은 콘솔 세션이며, 파이썬 콘솔은 `None`이 아닌 표현식의 결과에 언더바(`_`) 변수를 자동으로 할당한다. 이런 성질 때문에 의도한 대로 예제를 보여줄 수는 없었지만, 한편으로는 실제 상황을 잘 보여준다. 메모리를 섬세하게 제어하고자 할 때, 객체에 새로운 참조를 생성하는 암묵적인 할당 때문에 당황하는 경우가 종종 있다. `_` 변수가 그런 사례다. `Traceback` 객체도 예기치 않은 참조를 만들어내는 원인이다.

_예제 1. 약한 참조는 콜러블이다. 객체가 살아 있으면 참조된 객체를 반환하고, 그렇지 않으면 `None`을 반환한다_

```python
>>> import weakref
>>> a_set = {0, 1}
>>> wref = weakref.ref(a_set)  # (1)
>>> wref
<weakref at 0x100637598; to 'set' at 0x100636748>
>>> wref()  # (2)
{0, 1}
>>> a_set = {2, 3, 4}  # (3)
>>> wref()  # (4)
{0, 1}
>>> wref() is None  # (5)
False
>>> wref() is None  # (6)
True
```

(1) 약한 참조 객체 `wref`를 생성하고 다음 행에서 조사한다.

(2) `wref()`를 호출하면 참조된 객체 `{0, 1}`을 반환한다. 콘솔 세션에서 실행하고 있으므로 결과로 나온 `{0, 1}`이 `_` 변수에 바인딩된다.

(3) `a_set`이 더 이상 `{0, 1}` 집합을 참조하지 않으므로 참조 카운트가 줄어든다. 그렇지만 `_` 변수가 여전히 `{0, 1}`을 참조한다.

(4) `wref()`를 호출하면 여전히 `{0, 1}`이 반환된다.

(5) 표현식을 평가할 때 `{0, 1}`이 살아 있으므로 `wref()`는 `None`이 아니다. 그렇지만 `_` 변수는 결과값인 `False`에 바인딩된다. 이제 `_` 변수는 더 이상 `{0, 1}`을 참조하지 않는다.

(6) 이제 `{0, 1}` 객체가 제거되었으므로 `wref()`를 호출하면 `None`이 반환된다.


`weakref` 모듈 문서(https://docs.python.org/3/library/weakref.html)에서는 `weakref.ref` 클래스는 고급 사용자를 위한 저수준 인터페이스며, 일반 프로그래머는 `weakref` 컬렉션과 `finalize()`를 사용하는 것이 좋다고 설명한다. 즉, `weakref.ref` 객체를 직접 만들기보다는 `WeakKeyDictionary`, `WeakValueDictionary`, `WeakSet`, 그리고 내부적으로 약한 참조를 이용하는 `finalize()`를 사용하는 것이 좋다. [예제 1]에서는 `weakref.ref` 객체 하나가 작동하는 방식을 보면서 약한 참조 개념을 익히기 위해 `weakref.ref` 객체를 직접 만들었지만, 대부분의 경우 파이썬 프로그램은 `weakref` 컬렉션을 사용한다.

다음 절에서는 `weakref` 컬렉션에 대해 간단히 살펴본다.

## 5.2 `WeakValueDictionary` 촌극

`WeakValueDictionary` 클래스는 객체에 대한 약한 참조를 값으로 가지는 가변 매핑을 구현한다. 참조된 객체가 프로그램 다른 곳에서 가비지 컬렉트되면 해당 키도 `WeakValueDictionary`에서 자동으로 제거된다. 이 클래스는 일반적으로 캐시를 구현하기 위해 사용된다.

여기에서 설명하는 `WeakValueDictionary` 사용 예는 몬티 파이튼의 고전적인 ‘치즈 가게<sup>Cheese Shop</sup>’ 촌극에서 영감을 받았다. ‘치즈 가게’ 촌극에서는 체다, 모차렐라 등 40여 종의 치즈를 고객이 주문하지만, 그중 어느 것도 재고가 없다.[^1]

[예제 2]는 치즈의 종류를 나타내는 간단한 클래스를 구현한다.

_예제 2. `kind` 속성과 표준 표현 메서드를 가지고 있는 `Cheese` 클래스_

```python
class Cheese:

    def __init__(self, kind):
        self.kind = kind

    def __repr__(self):
        return f'Cheese({self.kind!r})'
```


[예제 3]에서는 `catalog`에 들어 있는 각종 치즈가 `WeakValueDictionary`로 구현되어 있는 `stock` 배열에 로딩된다. 그런데 `catalog`를 제거하자마자 `stock`에 있는 치즈가 하나만 빼고 모두 사라진다. 파르마<sup>Parmesan</sup> 치즈가 다른 치즈보다 오래 남아 있는 이유는 무엇일까?[^2] 예제 코드 다음에 나오는 팁 글상자에 정답이 있다.

_예제 3. 고객: "파는 치즈가 있기는 하나요?"_

```python
>>> import weakref
>>> stock = weakref.WeakValueDictionary()  # (1)
>>> catalog = [Cheese('Red Leicester'), Cheese('Tilsit'),
...            Cheese('Brie'), Cheese('Parmesan')]
...
>>> for cheese in catalog:
...     stock[cheese.kind] = cheese  # (2)
...
>>> sorted(stock.keys())
['Brie', 'Parmesan', 'Red Leicester', 'Tilsit']  # (3)
>>> del catalog
>>> sorted(stock.keys())
['Parmesan']  # (4)
>>> del cheese
>>> sorted(stock.keys())
[]
```

(1) `stock`은 `WeakValueDictionary` 인스턴스다.

(2) `stock`은 치즈명을 `catalog`에 있는 `Cheese` 인스턴스에 대한 약한 참조로 매핑한다.

(3) `stock`에 모든 치즈명이 들어 있다.

(4) `catalog`를 제거한 후, 예상대로 `WeakValueDictionary` 인스턴스인 `stock`에서 대부분의 치즈가 사라졌다. 그런데 하나가 남아 있는 이유는?


>**TIP_** 임시 변수가 객체를 참조함으로써 예상보다 객체의 수명이 늘어날 수 있다. 지역 변수는 함수가 반환되면서 사라지므로 일반적으로 문제가 되지 않는다. 그러나 [예제 3]의 경우 `for` 루프 변수인 `cheese`가 전역 변수이므로, 명시적으로 제거하기 전에는 사라지지 않는다.


`WeakValueDictionary`와 짝꿍인 `WeakKeyDictionary` 클래스는 키가 약한 참조다. `weakref.WeakKeyDictionary` 문서(https://docs.python.org/3/library/weakref.html#weakref.WeakKeyDictionary)의 다음과 같은 설명을 보면 이 클래스를 어디에 사용할지 감을 잡을 수 있다.

>`WeakKeyDictionary`는 애플리케이션의 다른 부분에서 소유하고 있는 객체에 속성을 추가하지 않고 추가적인 데이터를 연결할 수 있다. 이 클래스는 속성 접근을 오버라이드하는 객체에 특히 유용하다.

`weakref` 모듈은 `WeakSet` 클래스도 제공한다(문서에서는 `WeakSet` 클래스를 '약한 참조로 요소를 가리키는 집합 클래스. 어떤 요소에 대한 참조가 더 이상 존재하지 않으면 해당 요소가 제거된다'고 설명한다). 자신의 객체를 모두 알고 있는 클래스를 만들어야 한다면, 각 객체에 대한 참조를 모두 `WeakSet` 형의 클래스 속성에 저장하는 것이 좋다. 그렇게 하지 않고 일반 집합을 사용하면 이 클래스로 생성한 모든 객체는 가비지 컬렉트되지 않을 것이다. 클래스 자체가 객체에 대한 강한 참조를 하므로, 명시적으로 제거하지 않는 한 파이썬 프로세스가 종료될 때까지 객체가 제거되지 않기 때문이다.

지금까지 설명한 컬렉션과 약한 참조는 일반적으로 다룰 수 있는 객체의 종류가 한정되어 있다. 다음 절에서 자세히 알아보자.

## 5.3 약한 참조의 한계

모든 파이썬 객체가 약한 참조의 대상이 될 수 있는 것은 아니다. 기본적인 `list`와 `dict` 객체는 참조 대상이 될 수 없지만, 이 클래스들의 서브클래스는 이 문제를 다음 코드처럼 쉽게 해결할수 있다.

```python
class MyList(list):
    """인스턴스가 약한 참조될 수 있는 리스트 서브클래스"""

a_list = MyList(range(10))

# a_list는 약한 참조의 대상이 될 수 있다.
wref_to_a_list = weakref.ref(a_list)
```

`set` 인스턴스는 참조 대상이 될 수 있다. 그렇기 때문에 [예제 1]에서 `set`이 사용되었다. 사용자 정의형도 아무런 문제없이 참조 대상이 될 수 있다. 그래서 [예제 3]에서 바보 같은 `Cheese` 클래스가 필요했던 것이다. 그러나 `int` 및 `tuple` 인스턴스는 클래스를 상속해도 약한 참조의 대상이 될 수 없다.

이러한 제약사항 대부분은 CPython 구현 방식에 따른 것이므로, 다른 파이썬 구현에서는 적용되지 않을 수도 있다. 이들은 내부 구현 최적화에 의해 발생하는 문제며, 그중 일부에 대해 다음 절에서 설명한다.


[^1]: cheeseshop.python.org는 파이썬 패키지 인덱스(Python Package Index, PyPI) 소프트웨어 저장소의 또 다른 도메인명이기
도 하다. 초기 PyPI는 거의 비어 있었지만, 이 책을 쓰고 있는 현재 파이썬 치즈 가게에는 314,556개의 패키지가 등록되어 있다.

[^2]: 파르마 치즈는 공장에서 최소 1년간 숙성해야 하므로 다른 생치즈보다 오래가지만, 여기서는 그런 이유를 찾는 것이 아니다.