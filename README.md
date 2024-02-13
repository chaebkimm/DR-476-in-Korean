This is Korean translation of [DR 476, volatile semantics for lvalues](https://www.open-std.org/jtc1/sc22/wg14/www/docs/summary.htm#dr_476), in [Defect Report Summary for C11](https://www.open-std.org/jtc1/sc22/wg14/www/docs/summary.htm).

## DR 476
제출자: Martin Sebor

제출일: 2015-08-26

소스: [WG14](https://www.open-std.org/jtc1/sc22/wg14/) (국제 C언어 표준화 실무단) 

참고문헌: [N1956](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1956.htm)

주제: 좌측값에서 volatile 키워드의 의미

### 요약
다음 섹션에서는 C에서 volatile 키워드의 의미에 대해 고찰하며, volatile 키워드가 기존 관행을 지지하지 않고 위원회가 제작했을 때의 의도를 반영하지 않음을 보인다. [기술적 수정 제안](#기술적-수정-제안)에서 기존 관행, 제작 의도와 조화를 이루기 위해 C, C++의 규격이 어떻게 수정되어야 할지를 자세히 다룬다. 










### 기술적 수정 제안
