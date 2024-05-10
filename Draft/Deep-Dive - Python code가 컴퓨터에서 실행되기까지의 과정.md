
> [!NOTE] Sub-Title
> 높은 단계에서만 이해하던 실행단계를 깊게 들어가보자.

- [ ] 본문의 기술 관련 내용들을 Tree 구조로 변경하여 대상 독자에 맞는 얘기만 하자.
- [ ] 대상 독자를 정해 봅시다.
- [ ] 설명을 보충해야 할 부분을 직접 하기 보다는 그 부분을 잘 서술한 다른 블로그에 대한 참조를 넣자.

# 이 글을 읽을 독자는 누구인가?
Python을 잘 사용할 줄 알고, 언어의 기반 개념들(예를 들어, 인터프리터, 동적 타이핑)을 알고 있으나, 실제 Runtime에서 어떤 수행 과정을 거쳐 실제 컴퓨터에서 코드가 실행되는지 모르는 분들에게 이 글이 조금이나마 도움이 되길 바랍니다.

# 왜 이 글을 쓰게 되었나?
코딩을 하던 중 특정 메서드가 어떤 로직으로 구현되어 있는지 보고 싶었어요. 하지만 최하단부까지 내려가 보았을 때 해당 구현체는 단순 interface만 존재한다는 것을 확인하였어요. 그래서 이와 같이 interface만 존재하는 코드가 어떻게 실행되는지 알아보려고 해요.

코딩을 할 때면 `import/from`을 통해 불러온 Python 표준 라이브러리의 내부 로직을 보고 싶을 때가 있죠. 저는 이번에 datetime 모듈을 살펴보았어요.

여러 곳에서 활발히 쓰이는 datetime 모듈의 함수 중 `stftime()`가 실제로 어떤 과정을 거쳐 제가 원하는 값을 반환하는지 궁금했어요.

함수의 동작을 보려면 해당 함수가 구현되어 있는 파일에 작성된 로직을 직접 봐야 알 수 있어요.

그래서 저는 datetime 모듈의 `strftime()`이 구현되어 있는 곳까지 탐험을 나서려 합니다. 

자, 가봅시다!

# 0. datetime 모듈의 `strftime()` 구현부까지 다이빙

datetime 모듈의 `strftime('%A')`가 어떤 동작을 통해 `datetime` 객체에 해당하는 요일을 문자열로 출력하는지 궁금했어요. 그래서 하나씩 내부로 들어가보았죠.

제가 작성한 코드는 다음과 같아요. 매우 간결하죠.

```python
# file: work_with_datetime.py  
  
from datetime import datetime  
  
datetime.now().strftime('%A') # 'Wednesday' -- 실행 시점의 요일을 반환
```

이 코드는 간단히 datetime 모듈의 `datetime.now()`를 호출해 실행 시점에 해당하는 시간을 반환하도록 하였어요. `stftime('%A')`는 `now()`의 반환값인 `datetime` 객체의 요일을 반환하도록 해요.

이어서 `strftime()` 내부로 들어가볼게요. 그 내용은 다음과 같아요.

```python
# file: datetime.py  
  
class date:  
    ...  
    def strftime(self, fmt):  
        ...  
        return _wrap_strftime(self, fmt, self.timetuple())
```

`strftime()`은 datetime.py의 `date` 클래스에 속해 있는 메서드이며, 내부적으로 세부 기능이 감싸져 있는 형태의 `_wrap_strftime()`을 반환하고 있네요. (함수에 언더스코어(`_`)가 prefix로 붙은 것을 보아, datetime.py에서 내부적으로만 사용하는 함수로 이해할 수 있어요)

계속해서 `_wrap_strftime()`으로 들어가볼게요.

```python
# file: datetime.py  
  
import time as _time  
...  
def _wrap_strftime(object, format, timetuple):  
    ...  
    return _time.strftime(newformat, timetuple)
```

오! `_wrap_strftime()`은 Python의 또 다른 표준 라이브러리인 time의 함수인 `strftime()`을 wrapping한 것이었네요. 하지만 여전히 세부적인 로직은 볼 수 없네요. 생각보다 간단하지 않군요 :)

이번에는 time 모듈의 strftime()으로 내려가볼게요!

```python
# file: time.py  
  
def strftime(format, p_tuple=None): # real signature unknown; restored from __doc__  
    """  
    strftime(format[, tuple]) -> string  
      
    Convert a time tuple to a string according to a format specification.  
    See the library reference manual for formatting codes. When the time tuple  
    is not present, current time as returned by localtime() is used.  
      
    Commonly used format codes:  
      
    ...  
    %A  Locale's full weekday name.
    ...  
      
    Other codes may be available on your platform.  See documentation for  
    the C library strftime function.  
    """  
    return ""
```

이렇게 가장 밑에 있는 `strftime()`의 최하단 선언부를 찾았습니다!

해당 함수의 [[docstring]]을 보아, 제가 최상위 코드에서 `strftime()`에 전달한 파라미터 `‘%A’`는 `datetime` 객체의 요일을 반환한다는 설명이네요.

하지만 여전히 로직은 보이지 않아요. 제가 보기엔 이 코드가 Python의 마지막 단계로 보이는데 말이죠. 이게 어떻게 된걸까요?

# 1. 여기서 드는 궁금증: 빈 문자열을 반환하는 `strftime()`이 어떻게 동작하는 거지?

지금까지의 단계는 다음과 같아요.

**_User Script -> `datetime.strftime()` -> `datetime._wrap_strftime()` -> `time.strftime()`_**

여기까지 내려오면서 저의 궁금증(`strftime()`는 어떤 로직으로 작성되어 있는가)은 아직 풀리지 않았어요. 추가로 단계의 맨 마지막 부분인 time 모듈의 `strftime()`가 구현된 부분을 보았을 때 또 다른 의문이 생겼어요. 

그 의문은 다음과 같아요.
**_`time.strftime()`는 빈 문자열(`“”`)을 반환하네? 그럼 어떻게 docstring에 작성되어 있듯 파라미터를 분석하여 요일을 반환하는 것과 같은 기능을 수행하는 거지?_**

이러한 기능은 분명 Python의 내부 어딘가에 동작을 가능하게 하는 무언가가 있을거예요.

그래서 궁금증을 풀기 위해 계속 나아가 보기로 했어요.

# 2. 갑자기 등장한 Python Interpreter와 CPython

`time.strftime()`의 구현부는 빈 문자열(`“”`)을 반환하도록 되어 있고, 그럼에도 제가 원하는 기능(`datetime` 객체의 요일을 반환)이 정상적으로 작동했어요.

어떤 것이 이러한 동작을 가능하게 할까요? 

결론부터 말씀드리자면, 제가 직접 하단부까지 내려온 코드는 전부 stub이라는 형태예요. 이 것은 실제 런타임에서 동작하는 코드가 아닌 interface인 것이죠. 또한 실제 로직이 수행되는 코드는 사용자가 Python을 다운받은 시점에 Lib/에 포함되어 있는 모듈들이에요. Python/~/Lib/의 경우는 해당 Python을 실행하고 sys.path를 호출하면, Python Interpreter가 모듈을 import할 때 기본으로 참조하는 경로들을 볼 수 있어요. 그 중 제가 앞서 얘기한 Python/~/Lib/ 또한 경로에 포함되어 있죠.
따라서 Runtime 시 Python Interpreter는 import한 모듈을 Lib/ 경로의 것을 사용하도록 되어 있으며, 제가 살펴본 코드는 실제로 사용되지 않는 코드예요.


그 답은 Python Interpreter와 CPython의 사이에서 찾을 수 있어요.

**Python은 표준 라이브러리(datetime, time 등)를 사용자의 코드로 `import`하면 Python Interpreter가 CPython에 구현되어 있는 코드를 참조하도록 되어 있어요.**

`datetime` 모듈의 경우, Python의 표준 라이브러리 중 일부 기능은 순수 Python으로 구현되어 있고, 일부는 C 확장 모듈로 구현되어 `lib-dynload/`에서 찾을 수 있습니다. `_datetime`이라는 C 확장 모듈은 성능이 중요한 날짜 및 시간 연산을 위해 사용됩니다.

어떤 시기에 우리가 사용한 저 코드가 CPython에 구현되어 있는 코드로 교체되어 로직을 수행할 수 있을까요? -> 혹시 import할때 바로 Lib/datetime.py를 사용하는 것이 아닌가? 그렇다면 IDE에서 한 단계씩 내려갈 때 사용했던 저 코드들은 대체 뭐지?

제 생각엔 `import`를 통해 표준 라이브러리 모듈을 호출하게 되면 Python Interpreter는 해당 모듈을 가진 CPython path를 통해 import하는 것 같아요.

![[Pasted image 20240425174511.png]]

따라서, 우리가 여기까지 내려오면서 본 코드는 실제로는 동작하지 않는 코드들이에요. Runtime 시 동작하는 코드는 위에서 보이는 cpython/Lib/datetime.py인 것이죠.
그렇다면 cpython의 코드 외에 다른 코드들은 왜 있는 걸까요? 이런 코드들을 py-stub이라고 해요.

# py-stub이란?
Py-stub은 파이썬에서 사용하는 "stub-file"을 지칭해요. 스텁 파일은 Python 모듈의 공개 인터페이스 구조만을 포함한 파일로, 주로 클래스, 변수, 함수들과 그 타입을 명시하는 데 사용해요.
이러한 스텁 파일을 통해 Mypy와 같은 정적 타입 체커가 C로 컴파일되어 Python 코드로 표현되지 않는 각종 라이브러리의 함수, 클래스 등의 타입을 결정하는 데에 중요한 역할을 해요.
Python에서 스텁 파일은 주로 .pyi 확장자를 사용해요. 또한 문법의 경우 일반 Python 문법을 따르지만, 런타임 로직(변수 초기화, 함수 본문 등)을 포함하지 않고 타입 어노테이션만을 포함하고 있어요. 
이번에 제가 본 파일(time.py의 `strftime()`)은 실제로 로직을 포함하지 않고 있으며, 해당 함수의 시그니처와 기능을 설명하는 주석만을 포함하고 있었어요. 이러한 파일을 스텁 파일이라고 하며, `...`으로만 되어 있는 경우도 있어요.

함수 본문에서는 단순히 빈 문자열을 반환하는 `return`문만을 포함하고 있어요. 이러한 구조는 스텁 파일에서 일반적으로 사용되는 방식이며, 타입 체킹이나 API 문서화 도구에서 해당 함수의 인터페이스를 분석하는 데 유용해요.

스텁 


Python 표준 라이브러리는 CPython에 구현되어 있어요. 따라서 datetime, time 등의 모듈/패키지들은 CPython 소스코드에서 확인할 수 있죠.

자, 이제 datetime 모듈이 CPython에 포함되어 있다는 것을 알았어요.

### 2. CPython에 구현되어 있는 datetime.py

CPython의 여러 폴더 중 datetime.py는 Lib에 포함되어 있어요. (cpython/Lib은 Python code로만 구현되어 있는 모듈/패키지가 있어요)

![CPython에서의 datetime.py 경로](https://cdn-images-1.medium.com/max/1600/1*OtgPkdporcdQH6Zwkiwg9w.png)

![cpython/Lib/datetime.py의 내용](https://cdn-images-1.medium.com/max/1600/1*BmiOuH4KS8NGkqnSvL87XA.png)

datetime.py는 위와 같은 코드로 구성된 모듈이에요. (이곳에도 여전히 로직은 보이지 않네요)

만약 제가 작성한 코드처럼(`from datetime import datetime`) 작성할 경우 위 코드는 다음과 같이 동작해요. (이걸 설명하는게 맞을까? 추상화 단계를 높이려 해보자. 아니면 좀 더 쉽게 애니메이션으로 만들어볼까?)
1. try 문의 `from _datetime import *`을 통해 `_datetime`에 포함된 속성(함수, 클래스, 전역변수 등)들을 로드(ImportError가 발생하면 `_pydatetime`을 `import`하도록 되어있지만 이 글에서는 생략할게요)
2. `__all__`에 

위 코드는 아마 이렇게 동작할거예요.

__all__: 다른 코드에서 이 모듈을 from datetime import *로 참조할 경우 __all__에 명시된 것들만 import할 수 있도록 해요. (만약 모듈/패키지에 __all__이 정의되어 있지 않다면 언더스코어(_)로 시작하지 않는 모든 이름이 import 돼요)

#### CPython-Lib-datetime.py

제가 이번 코드에서 사용한 datetime 모듈이 CPython에 속한 모듈이라는 것을 알았어요. 그렇다면 실제로 Python Runtime 시에 어떻게 CPython의 모듈을 불러와 사용할 수 있을까요?

이 같은 실행과정을 이해하기 위해서는 Python이 컴퓨터에서 어떤 과정으로 실행되어 최종 결과물을 사용자에게 반환하는지 알아보아야 했습니다.

따라서, 본 글의 본문인 ‘Python code가 컴퓨터에서 실행되기까지의 과정’을 이제 시작합니다. (서문이 너무 길었네요. 여기까지 읽어주셨겠죠? 믿어요 :D)

우선 Python의 동작 과정을 간략히 요약하면 다음과 같습니다. (간단하게 만들자. 또는 그림으로 만들기)

1. [User]
2. Python code 작성
3. Terminal이나 IDE(IDE에서 Run을 할 때는 Terminal에서 사용한 python cli를 그대로 사용합니다. 단지 Run 버튼으로 wrapping한 것일 뿐이죠)
4. python cli를 통해 Python Interpreter 프로세스 실행
5. [Interpreter]
6. 코드의 내용(text)를 메모리에 로드
7. 코드의 내용 파싱 및 구문분석을 통해 구문트리(Syntax-tree)를 생성
8. 구문트리를 Byte code로 컴파일 (Python Byte code란?)
9. [PVM]
10. Byte code의 내용을 한 줄씩 읽고 기계어로 번역하여 컴퓨터에서 실행(이 때 내장 데이터 구조, 함수, 모듈이 메모리에 로드됩니다)
11. Byte code의 내용을 한 줄씩 읽는 중 외부(Pypi 등으로 다운로드한 것들) 또는 표준 라이브러리(CPython에 구현되어 있는 것들)를 호출(사용)하는 경우 PVM이 해당 함수를 찾아 실행
12. 사용자에게 결과를 출력
13. [Interpreter]
14. Python Interpreter는 사용된 메모리를 해제하고 프로세스를 종료