This is Korean translation of [DR 476, volatile semantics for lvalues](https://www.open-std.org/jtc1/sc22/wg14/www/docs/summary.htm#dr_476), in [Defect Report Summary for C11](https://www.open-std.org/jtc1/sc22/wg14/www/docs/summary.htm).

## DR 476
제출자: Martin Sebor

제출일: 2015-08-26

소스: [WG14](https://www.open-std.org/jtc1/sc22/wg14/) (국제 C언어 표준화 실무단) 

참고문헌: [N1956](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1956.htm)

주제: 좌측값에서 volatile 키워드의 의미

### 요약
다음 섹션에서는 C에서 volatile 키워드의 의미에 대해 고찰하며, volatile 키워드가 기존 관행을 지지하지 않고 위원회가 제작했을 때의 의도를 반영하지 않음을 보인다. [기술적 수정 제안](#기술적-수정-제안)에서 기존 관행, 제작 의도와 조화를 이루기 위해 C, C++의 규격이 어떻게 수정되어야 할지를 자세히 다룬다. 

### volatile의 동기
volatile 키워드를 C에 도입하게된 동기를 제공한 용례는 다음과 같은 초기 UNIX 소스에서 복사해온 코드 일부의 변종이다 [[1]](#f1) :

```c
#define KL 0177560

struct { char lobyte, hibyte; };
struct { int ks, kb, ps, pb; };

getchar() {
  register rc;
  ...
  while (KL->ks.lobyte >= 0);
  rc = KL->kb & 0177;
  ...
  return rc;
}
```
getchar() 함수의 while문에서 원하는 효과는 KL 매크로(PDP11 컴퓨터에서 KBD_STAT I/O 레지스터가 매핑된 메모리 주소)에 정의된 메모리 주소에 매핑된 키보드 상태 레지스터의 가장 큰 자릿수의 비트가 0이 아닌 값이 될 때까지 반복하고 - 키보드의 키가 눌림을 의미 - 입력된 키에 해당하는 7개 비트에서 얻은 문자를 반환하는 것이다. 함수가 기대되는대로 행동하기 위해서는, 컴파일러가 만든 명령어에서 반복문의 각 반복 때마다 I/O 레지스터에서 값을 읽어와야 한다. 특히 컴파일러는 CPU 레지스터 값을 한번만 읽어와서 캐시에 저장해놓고 값을 읽어오는 것을 캐시에 저장된 값을 쓰는 것으로 대체하지 말아야 한다. 

한편, 메모리 주소가 메모리에 매핑된 레지스터에 해당하는 특별한 경우가 아닐 때에는 메모리에서 값을 한번 읽어서 CPU 레지스터에 저장한 후에는 다시 메모리에서 값을 읽는 것보다, CPU 레지스터에 캐시된 값을 쓰는 것이 더 효율적이다. 

문제는 어떤 종류의 표기 (K&R C에서는 없었음) 없이는 컴파일러의 입장에서 두 케이스를 구분할 방법이 없다는 것이다. <i>The C Programming Language, 2판, Kernighan, Ritchie 저</i>에서 발췌한 다음 문단은 표준 C에서 이 문제를 해결하기 위해 도입된 volatile 키워드를 설명한다. 

> volatile의 목적은 구현시 발생할 수 있는 최적화를 강제로 억제하는 것이다. 예를 들어, 메모리에 매핑된 입출력이 있는 기계의 경우, 장치의 레지스터를 가리키는 포인터는 volatile에 대한 포인터로 선언될 수 있으며, 이를 통해 컴파일러가 반복적으로 포인터의 값을 읽어오는 것을 제거하는 것을 방지할 수 있다. 

그렇다면 volatile 키워드를 사용해서 다음과 같이 코드 내 반복문을 수정해서 쓸 수 있어야 한다. 
```c
  while (*(volatile int*)&KL->ks.lobyte >= 0);
```
또는 동일하게:
```c
  volatile int *lobyte = &KL->ks.lobyte;
  while (*lobyte >= 0);
```
와 같이 쓰고, 컴파일러가 키보드 상태 레지스터의 값을 캐시에 저장해 놓고 쓰는 것을 방지하여, 레지스터가의 값이 각 반복에서 한번씩 읽어지는 것을 보장할 수 있어야 한다.

수정된 두 형태의 반복문간의 차이에는 역사적인 내용이 있다: 초기 C 컴파일러들은 (volatile 키워드 없이) 첫번째 패턴을 인식한다고 알려져왔는데, 레지스터에 접근하는데 쓰이는 주소는 상수였고, 레지스터 접근에 대한 원치 않는 최적화는 방지되었다 [[11]](#f11). 하지만, 주소가 저장되어 있는 포인터 변수를 통해서 접근을 했을 경우에는 같은 기능을 하지 못했으며, 특히 변수의 사용이 마지막 대입과 다를 때 더 그랬다. volatile 키워드는 두 반복문 형태 모두 기대한 대로 동작하게 하는 것을 의도했다. 

반복문으로 보인 용례는 관용적이되어 최신 소프트웨어에서 입출력 레지스터 값을 읽는 것을 넘어서 널리 쓰이게 되었다.

대표적인 예로, 리눅스 커널에서는 스핀락과 같은 동기화 구성요소를 구현할 때나 성능 카운터를 쓸 때 volatile에 의존한다. 동기화 구성요소에서 쓰는 변수는 대부분 특별히 규정되지 않은 (volatile이 아닌) 기본형 타입으로 일반 메모리에 할당된다. 직렬 코드에서, 최대의 효율을 위해, 각 변수를 읽고 쓰는 동안에는 일반적인 다른 변수에서와 동일하게 컴파일러 최적화가 지정한대로 값이 CPU 레지스터에 캐시되어 있다. 코드의 잘 정의된 어떤 지점에서 변수가 여러 CPU에 의해서 동시에 접근될 수 있으면, 캐싱이 반드시 방지되어야 하고, 변수는 반드시 volatile을 써서 접근되어야 한다. 이를 위해서, 커널은 READ_ONCE와 WRITE_ONCE라는 두 매크로를 정의해서 동기화 구성요소를 구현한다. 각 매크로는 컴파일러 최적화를 방지하기 위해 인자의 주소를 volatile T*로 형변환하고, volatile로 규정된 T (T는 기본형 타입 중 하나) type의 lvalue를 통해 변수에 접근한다. 다른 구성요소는 메모리 동기화와 가시성을 보장하지만 본 문서의 주제에서 벗어난다. [[3]](#f3)에 설명되어 있다. 

비슷한 예는 C11, C++11에서 새로 도입된 원자성 타입과 연산에 의존하지 않는 C11, C++11 이전 코드들이나 다른 시스템, 임베디드 프로그램에서 찾아볼 수 있다. 프로그램 책[[4]](#f4)과 인터넷 문헌[[5](#f5),[6](#f6),[7](#f7),[8](#f8)]에서 종종 인용된다. 

### volatile의 문제점
volatile의 필요성과 광범위한 업무에서 이에 기대되는 효과에 의존하고 있음에도 불구하고 C 표준에서 이 유명한 단어를 위한 필수적인 보증이 없다는 것은 놀라운 일이다. C 표준의 텍스트에서, volatile의 의미가 적용되는 대상을 명시적으로 규정된 객체에 한정하고 있다. &sect;5.1.2.3, 프로그램 실행, 2쪽에서 인용하면:

> volatile 객체에 접근하고, 객체를 수정하고, ... 모두 부수효과로 실행 환경의 상태의 변화이다. 

그리고 6쪽에서:

>volatile 객체에 대한 접근은 추상 기계에 대한 규칙에 따라 엄격하게 정해진다. 

여기서 특히 유의할 점은 텍스트에서 언급되는 volatile 객체는 값을 저장하고 있는 저장소의 일부 영역으로 정의되어있다는 것이다. 객체는 이를 가리키고 접근하는 식과는 다르다. 이런 식을 좌측값이라고 불리며, 가리키는 객체의 이름일 수도 있고, 꼭 그렇지 않아도 된다. 하지만, 상기 문단의 단어들이 좌측값을 언급하지 않기 때문에 특별한 volatile 의미가 좌측값 접근에 적용되지 않는다. 그 결과로, `*(volatile int*)&KL->ks.lobyte` 식이 객체가 아니고 volatile int 타입의 좌측값으로 정해지지않은 타입의 객체를 가리킨다는 이유로 (KL 포인터는 C 관점에서 객체를 가리키지 않는다), volatile의 효과가 적용되지 않는다. 결과적으로, 그리고, &sect;6.8.5, 반복문, 6쪽에 의해

> 조건식이 상수식이 아닌 반복문은, ... volatile 객체에 접근하지 않으면 ... 가정하기를 ... 종료한다.

while문의 조건식이 계산될 때 특별한 volatile 효과를 적용받지 않고, C 컴파일러가 키보드 상태 레지스터의 값을 한번만 읽을 수 있게 허가하고, 값이 0이라고 하더라도 함수에서 반환하게 된다. (어떤 알려진 컴파일러도 이 허가를 남용하는 것이 관찰되지는 않았다.) 이는 당연히 getchar 함수가 예측할 수 없는 행동을 하게 만든다. 

volatile에 대한 C 규격의 문제가 널리 알려져 있지는 않아도, 새로운 것은 아니다. 과거에 지적된 적이 있으며, 예를 들어 <i> The trouble with volatile </i> [[9]](#f9)에서, Jonathan Corbet은 리눅스 커널의 저자 및 관리자인 Linus Torvalds를 인용해서 말한다:

> 또한, 더 중요하게, "volatile"은 전체 시스템 중에서 잘못된 축에 속한다. C에서 volatile인 대상은 "데이터"이지만 이것은 정신 나간 일이다. 데이터가 volatile인 것이 아니다 - 접근이 volatile이다. 따라서 "특정 접근을 조심하라"고 말하는 것은 말이 되지만, "특정 데이터에 대한 모든 접근을 어떤 랜덤한 방법으로 하라"고 말하는 것은 아니다. 


### C++에서의 Volatile
이 문제는 C 표준의 고유한 문제다. C와는 다르게 C++ 표준의 텍스트는 volatile object라는 표현을 피하고 대신 volatile 일반좌측값이라고 부른다. (일반좌측값은 C에서의 좌측값 개념을 C++에서 일반화한 것이다.) 상기된&sect;5.1.2.3 프로그램 실행, C11의 2쪽과 &sect;1.9 프로그램 실행 12쪽에 해당하는 C++ 텍스트를 보면:

> volatile 일반좌측값이 가리키는 객체에 접근하는 것, 객체를 수정하는 것, ... 은 모두 부속 효과로, 실행 환경 속성의변화다. 

이 차이점을 고의로 또는 우연히 C++ 규격이 C에서 벗어난 것이라고 기록하고 싶은 마음이 들 수 있다. 하지만 C++ 표준은 &sect;7.1.6.1 cv지정자의 유익한 메모에서 확실히 하고 있다:
> 일반적으로, volatile의 의미는 C++에서와 C에서 같을 것을 의도한다. 

이 메모는 최신 2014년 C++에서도 보이고 시간을 거슬러 올라 1998년 표준의 첫번째 개정안에서도 보인다. 

### 의도된 의미
C 표준의 단어들이 실제 관행을 반영하고 있지 않다는 증거 외에도, C++ 표준에 있는 유용한 메모 너머에 단어들이 기존에 제작될 당시의 위원회의 원래 의도도 반영하지 않고 있을 가능성이 높다는 것을 보이는 기록도 있다. 

C99 Rationale [[10]](#f10)에 따르면 &sect;6.7.3에서 volatile을 도입했을 때 위원회의 의도가 volatile이 아닌 객체에 volatile 지정 좌측값으로 접근했을 때 적용되는 것이고, 명시적으로 volatile이라고 지정된 접근할때뿐이 아니라고 확실히 하고 있다.

> C89 위원회는 C에 두 지정자 const와 volatile를 추가했다; ... 개별적으로, 결합되어서도 두 지정자는 컴파일러가 좌측값으로 객체에 접근할 때 할 수 있고 해야하는 가정을 지정한다.
> 
> ... volatile과 restrict는 위원회의 발명이며, 둘 다 const의 구문 모델을 따른다.

(메모: const의 구문 모델은 좌측값을 통한 접근에 불변성을 부여하며, 이는 접근되는 객체가 const로 지정되어 선언되었는지 아닌지와 관계없다.)

같은 섹션에서 추가적으로 밝힌다:

> 필요하다면 volatile이 아닌 객체에 volatile 의미를 써서 접근할 수 있고, 그 테크닉은 객체의 주소를 적합한 지정자가 적용된 타입으로 형변환한 후에 역참조하는 것이다. 

### 기술적 수정 제안
제안된 기술적 수정은 volatile 규격을 실제 관행, 원래 의도, C++ 규격과 맞춘다. 

&sect; 5.1.2.3, 프로그램 실행, 2쪽에서:

> ***volatile로 지정된 좌측값을 이용한 객체*** ~~volatile 객체~~에 대한 접근, 파일의 수정, 이런 연산을 수행하는 함수의 호출 ...

&sect; 5.1.2.3, 프로그램 실행, 4쪽에서:
> 실제 구현은 식 일부의 값이 쓰이지 않고 필요한 부속 효과가 없다고 판단할 경우 식 일부를 계산하지 않아도 된다 (함수 호출이나 ***volatile로 지정된 좌측값을 이용한 객체*** ~~volatile 객체~~에 대한 접근을 포함한다).

&sect; 5.1.2.3, 프로그램 실행, 6쪽, 첫번째 항목에서:
> ***volatile로 지정된 좌측값을 이용한 객체*** ~~volatile 객체~~에 대한 접근은 추상 기계에 대한 규칙을 엄격하게 따른다. 

&sect; 6.7.3, 타입 지정자, 7쪽에서:
> 무엇이 ***volatile로 지정된 좌측값을 이용한 객체*** ~~volatile 객체~~에 대한 접근으로 간주되는지는 구현에서 정의한다.

&sect; 6.8.5, 반복문, 6쪽에서:
> 반복문 중 조건식이 상수식이 아니거나,<sup>156)</sup> 입출력 연산을 하지 않거나, ***volatile로 지정된 좌측값을 이용한 객체*** ~~volatile 객체~~에 대한 접근을 하지 않거나, ... 구현에 의해 종료된다고 가정될 수 있다. 

&sect; J.3.10, 지정자, 1쪽에서:
> ***volatile로 지정된 좌측값을 이용한 객체*** ~~volatile 객체~~에 대한 접근으로 간주되는 것(6.7.3).

&sect; L.2.1, 1쪽에서:
> 구역 이탈은 프로그램 실행시, 주어진 계산 상태에서, 본 표준에 의해 허가된 구역을 벗어난 하나 이상의 바이트의 변경 (또는, ***volatile로 지정된 좌측값*** ~~volatile 객체~~에서 바이트 값을 가져옴)을 하는 접근(시도)이다.


### 참고문헌

<a name="f1">1</a>. [/usr/src/stand/pdp11/iload/console.c](http://minnie.tuhs.org/cgi-bin/utree.pl?file=SysIII/usr/src/stand/pdp11/iload/console.c), AT&T UNIX System III, 1982년

<a name="f2">2</a>. The C Programming Language, Second Edition, Brian W. Kernighan, Dennis M. Ritchie

<a name="f3">3</a>. [N4444](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4444.html) ISO/IEC SC22/WG21 문서: Linux-Kernel Memory Model, Paul E. McKenney

<a name="f4">4</a>. [8.4. Const and volatile](https://publications.gbdirect.co.uk//c_book/chapter8/const_and_volatile.html), The C Book, Second Edition, Mike Banahan and Declan Brady, GBdirect

<a name="f5">5</a>. [Introduction to the volatile keyword](https://www.embedded.com/introduction-to-the-volatile-keyword/), embedded.com의 글, Nigel Jones, 2001년 7월 2일

<a name="f6">6</a>. [Why does volatile exist?](https://stackoverflow.com/questions/72552/why-does-volatile-exist), stackoverflow.com의 글, 2008년 9월 16일

<a name="f7">7</a>. [Why is volatile needed in c?](https://stackoverflow.com/questions/246127/why-is-volatile-needed-in-c), stackoverflow.com의 글, 2008년 10월 29일

<a name="f8">8</a>. [volatile (computer programming)](https://en.wikipedia.org/wiki/Volatile_(computer_programming)),
위키피디아의 글

<a name="f9">9</a>. [The trouble with volatile](https://lwn.net/Articles/233479/),
LWN의 글, Jonathan Corbet, 2007년 5월 9일

<a name="f10">10</a>. [국제 표준의 학술적 근거 -- 프로그래밍 언어 -- C](https://www.open-std.org/jtc1/sc22/wg14/www/C99RationaleV5.10.pdf), 개정안 5.10, 2003년 4월

<a name="f11">11</a>. [A question on volatile accesses]() - comp.std.c 문의에 대한 Doug Gwyn의 응답, 1990년 11월
