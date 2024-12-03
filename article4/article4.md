# 4. `struct`로 이진 레코드 파싱하기

`struct` 모듈은 바이트 필드를 파이썬 객체의 튜플로 파싱하고, 그 반대로 튜플을 패킹된 바이트 시퀀스로 변환하는 기능을 제공한다. `struct`는 `bytes`, `bytearray`, `memoryview` 객체와 함께 사용할 수 있다.

`struct` 모듈은 강력하고 편리하지만, 그 전에 다른 방법을 없을지 고민해보는 편이 좋다. 이 내용이 이번 글의 첫 번째 꼭지다.

내용:

[4.1 `struct`를 사용해야 하나?](article4.md#41-struct를-사용해야-하나)

[4.2 `struct` 기본 지식](article4.md#42-struct-기본-지식)

[4.3 `struct`와 `memoryview`](article4.md#43-struct와-memoryview)

## 4.1 `struct`를 사용해야 하나?

실세계에서 사용하고 있는 고유한 이진 레코드들은 불안정해서 손상되기 쉽다. [4.2 `struct` 기본 지식](article4.md#42-struct-기본-지식)에 있는 아주 간단한 예제는 여러 문제점 중 하나를 보여준다. 문자열 필드는 바이트 크기로만 제한할 수 있어서 공백 문자가 들어가거나 널 문자로 끝난 문자열 뒤에 무작위 쓰레기 값으로 채워질 수 있다. 엔디안 문제도 있다. 정수나 실수를 표현하는 바이트의 순서는 CPU 아키텍처에 따라 달라진다.

기존 이진 포맷에 쓰거나, 이진 포맷을 읽어야 한다면 직접 해결책을 구현하기 보다는 바로 사용할 수 있는 라이브러리를 찾아보는 편을 권장한다.

회사내 파이썬 시스템 간에 이진 데이터를 교환해야 한다면 피클(`pickle`) 모듈이 가장 손쉬운 방법이지만, 파이썬 버전에 따라 기본적으로 사용하는 이진 포맷이 다를 수 있고, 피클을 읽으면 임의의 코드가 실행될 수 있으므로 외부에 사용하기에 안전하지 않다.

다른 언어로 구현된 프로그램과 데이터를 교환해야 한다면 JSON이나 MessagePack(https://msgpack.org/)이나 Protocol Buffers(https://protobuf.dev/)와 같은 다중 플랫폼 이진 직렬화 포맷을 사용하는 편이 좋다.

## 4.2 `struct` 기본 지식

[예제 1]에 정의된 구조체를 이용해 C 언어로 구현된 프로그램이 생성한 도시 지역 데이터를 담은 파일을 읽는다고 생각해보자.

_예제 1. MetroArea: C 언어 구조체_

```c
struct MetroArea {
    int year;
    char name[12];
    char country[2];
    float population;
};
```

`struct.unpack`을 이용해 그 포맷의 레코드 하나를 읽는 코드는 다음과 같다.

_예제 2. 파이썬 콘솔에서 C 구조체 읽기_

```python
>>> from struct import unpack, calcsize
>>> FORMAT = 'i12s2sf'
>>> size = calcsize(FORMAT)
>>> data = open('metro_areas.bin', 'rb').read(size)
>>> data
b"\xe2\x07\x00\x00Tokyo\x00\xc5\x05\x01\x00\x00\x00JP\x00\x00\x11X'L"
>>> unpack(FORMAT, data)
(2018, b'Tokyo\x00\xc5\x05\x01\x00\x00\x00', b'JP', 43868228.0)
```

`unpack()`이 `FORMAT` 문자열에 명시된 필드 네 개가 있는 튜플을 반환하는 방법을 잘 살펴보라. `FORMAT`에 있는 문자와 숫자는 `struct` 모듈 문서(https://docs.python.org/3.8/library/struct.html) 에 설명된 포맷 문자(https://docs.python.org/3.8/library/struct.html#format-characters) 들이다.

[표 1]은 [예제 2]의 포맷 문자열 요소를 설명한다.

_표 1. 포맷 문자열 `'i12s2sf'`의 요소_

| **부분** | **크기** |  **C 자료형** | **파이썬 자료형** | **내용 제한** |
|----|----|----|----|----|
| `i` | 4 바이트 | `int` | `int` | 32 비트: `-2,147,483,648`에서 `2,147,483,647`까지의 범위 |
| `12s` | 12 바이트 | `char[12]` | `bytes` | 길이 12 |
| `2s` | 2 바이트 | `char[2]` | `bytes` | 길이 2 |
| `f` | 4 바이트 | `float` | `float` | 32 비트: 대략 ± 3.4×10<sup>38</sup>의 범위 |

metro_areas.bin 파일의 배치에 관련해 [예제 1]이 명확히 표현하지 않는 부분이 하나 있다. `name`과 `country` 필드는 크기만 다른 것이 아니다. `country` 필드는 언제나 2 글자로 이루어진 국가 코드를 나타내지만, `name`은 널 문자(`b'\0'`)를 포함해 최대 12 바이트로 구성된 시퀀스다. [예제 2]에서 `Tokyo` 바로 뒤에 널 문자가 나온다.[^1]

이제 metro_areas.bin에서 레코드를 모두 추출하는 스크립트를 살펴보고, 다음과 같이 간단한 리포트를 만들어보자.

```bsh
$ python3 metro_read.py
2018    Tokyo, JP       43,868,228
2015    Shanghai, CN    38,700,000
2015    Jakarta, ID     31,689,592
```

[예제 3]은 편리한 `struct.iter_unpack()` 함수의 사용예를 보여준다.

_예제 3. metro\_read.py: metro\_areas.bin 파일 안의 레코드를 모두 나열한다_

```python
from struct import iter_unpack

FORMAT = 'i12s2sf'                             # (1)

def text(field: bytes) -> str:                 # (2)
    octets = field.split(b'\0', 1)[0]          # (3)
    return octets.decode('cp437')              # (4)

with open('metro_areas.bin', 'rb') as fp:      # (5)
    data = fp.read()

for fields in iter_unpack(FORMAT, data):       # (6)
    year, name, country, pop = fields
    place = text(name) + ', ' + text(country)  # (7)
    print(f'{year}\t{place}\t{pop:,.0f}')
```

(1) `struct` 포맷

(2) `bytes` 필드를 디코딩하는 편리 함수. `str`을 반환한다.[^2]

(3) 널 문자로 끝나는 C 문자열을 처리한다. `b'\0'`으로 분할하고 첫 번째 부분을 가져온다.

(4) `bytes`를 `str`로 디코딩한다.

(5) 파일을 이진 모드로 열고 전체를 읽는다. `data`는 `bytes` 객체다.

(6) `iter_unpack()`은 포맷 문자열에 매칭되는 각 바이트 시퀀스에 대해 튜플 하나를 생성하는 제너레이터를 반환한다.

(7) `name`과 `country` 필드에 대해 `text()` 함수가 처리를 약간 더 한다.


`struct` 모듈은 널 문자로 끝나는 문자열을 지정하는 방법을 제공하지 않는다. 앞 예제의 `name`과 같은 필드를 처리할 때 언패킹한 후 반환된 바이트 시퀀스에서 처음 나오는 `b'\0'`와 그 뒤의 모든 바이트들을 버려야 한다. 첫 번째 `b'\0'` 다음부터 필드의 끝까지 쓰레기 값이 들어있을 가능성이 높다. [예제 2]에서  쓰레기 값을 볼 수 있다.

다음 절에서는 메모리뷰를 설명한다. 메모리뷰는 `struct`를 사용한 프로그램을 실험하고 디버깅하기 쉽게 해준다.

## 4.3 `struct`와 `memoryview`

파이썬에서 제공하는 `memoryview` 자료형은 바이트 시퀀스를 생성하거나 저장하지 않고, 대신 바이트 시퀀스를 복사할 필요 없이 이진 시퀀스, 패킹된 배열, 혹은 파이썬 이미지 라이브러리<sup>Python Image Library</sup>(PIL)에 접근을 공유할 수 있게 해준다.[^3]

[예제 4]는 `memoryview`와 `struct`를 함께 사용해 GIF 이미지의 너비와 높이를 추출한다.

_예제 4. `memoryview`와 `struct`를 이용해 GIF 이미지 헤더 조사하기_

```python
>>> import struct
>>> fmt = '<3s3sHH'  # (1)
>>> with open('filter.gif', 'rb') as fp:
...     img = memoryview(fp.read())  # (2)
...
>>> header = img[:10]  # (3)
>>> bytes(header)  # (4)
b'GIF89a+\x02\xe6\x00'
>>> struct.unpack(fmt, header)  # (5)
(b'GIF', b'89a', 555, 230)
>>> del header  # (6)
>>> del img
```

(1) `struct` 포맷. `<`: 리틀 엔디안. `3s3s`: 3 바이트 시퀀스 두 개. `HH`: 16 비트 정수 두 개

(2) 메모리 안에 있는 파일 내용으로부터 `memoryview`를 생성한다.

(3) 그러고 나서 이전에 생성한 `memoryview`로부터 슬라이싱해서 또 다른 `memoryview`를 만든다. 이 때 아무런 값도 복사되지 않는다.

(4) 출력해보기 위해 `bytes`로 변환한다. 여기서는 10 바이트가 복사된다.

(5) `memoryview`를 `(형식, 버전, 너비, 높이)` 튜플로 언패킹한다.

(6) `memoryview` 인스턴스에 연결된 메모리를 해제하기 위해 참조를 제거한다.


`memoryview`를 슬라이싱하면 아무런 바이트도 복사하지 않고 새로운 `memoryview`를 반환한다.[^4]

여기에서는 `memoryview`와 `struct` 모듈에 대해 자세히 다루지 않지만, 이진 데이터를 다룬다면 '메모리 뷰'(https://docs.python.org/3/library/stdtypes.html#memory-views) 문서와 '`struct` - `bytes`를 패킹된 이진 데이터로 해석하기'(https://docs.python.org/3/library/struct.html) 문서를 살펴보기를 추천한다.


[^1]: `\0`과 `\x00` 모두 파이썬 `str`과 `bytes` 리터럴 안의 널 문자(`chr(0)`)에 대한 올바른 이스케이프 시퀀스다.

[^2]: 이 예제는 이 책에서 처음으로 함수 시그너처에 자료형 힌트를 사용한다. 이런 간단한 자료형 힌트는 코드의 가독성을 높이고 직관적으로 이해할 수 있게 해준다.

[^3]: Pillow(https://pillow.readthedocs.io/en/latest/)는 PIL 포크 중에서 가장 활발히 관리되고 있다.

[^4]: 이 책의 테크 리뷰어중 하나인 레오나르도 로챌<sup>Leonardo Rochael</sup>은 `mmap` 모듈을 이용해 이미지 파일을 메모리-맵 파일<sup>memory-mapped file</sup>로 열었다면 메모리를 훨씬 적게 사용했을 것이라고 지적했다. `mmap` 모듈은 이 책의 범위를 벗어나지만, 이진 파일을 읽고 변경하는 작업을 자주 수행한다면 '`mmap` - 메모리 맵 파일 지원'(https://docs.python.org/3/library/mmap.html) 문서를 공부해두기를 바란다.