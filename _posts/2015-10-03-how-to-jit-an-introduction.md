---
layout: post
title: "JIT은 어떻게 작동하는가? (How to JIT -an introduction) from Eli Bendersky's website "
date: 2015-10-03 00:57:00
categories: "eli's article"
---
우선 이 글은 [Eli Bendersky의 블로그](http://eli.thegreenplace.net/)에서 번역해 온 글임을 알려드립니다.

상당한 의역이 난무하므로 지적사항이 있으면 따로 알려주시면 감사하겠습니다.

[원문](http://eli.thegreenplace.net/2013/11/05/how-to-jit-an-introduction)이 궁금하신 분은 링크를 타고 들어가셔서 읽어보시면 될 것 같습니다.

번역에 대한 허락은 원저작자에게 받았으며, 이 글에 대한 저작권은 원저작자에게 있음을 알려드리며,

번역 된 글을 다른 곳에 배포하거나 수정 할 시에는 저에게 연락을 주시면 감사하겠습니다.

-----------------------------------


libjit에 관한 소개글 [원문](http://eli.thegreenplace.net/2013/10/17/getting-started-with-libjit-part-1/)을 작성할 때, 나는 이미 JIT에 대해 약간의 지식이라도 알고 있는 프로그래머를 대상으로 하였다.
이전에 JIT에 대해 설명하긴 했지만 거의 요약수준이었다. 이번 글의 목적은 JITing에 대한 전반적인 소개를 어떠한 라이브러리에 의존하지 않은 코드 샘플을 활용하여 하는 것이다.

## JIT의 정의 ##
JIT은 단순히 "Just In Time"의 약자이다. 하지만 이 자체는 별 도움이 되지 않는다. - 용어 자체가 뭔가 복잡하고 약간의 프로그래밍이 필요해보인다.
우선 "JIT"을 있는 그대로 정의해보자. 나는 이것을 쉽게 풀어써서

>프로그램이 실행되는 동안, 디스크에 저장된 그 프로그램의 일부가 아닌 실행가능한 코드를 생성하고 실행하면
>그것을 JIT이라고 한다.

"JIT"의 역사적인 용어 사용은 어떻게 된 걸까? 운이 좋게도 Calgary대학의 John Aycock이 쓴 "A Brief Hitory of Just-In-Time"(구글에 검색해보시길 PDF로 다운로드가 가능하다.)
에서  JIT의 기술을 역사적인 관점에서 보고 있다. Aycock의 논문에 따르면 최초로 프로그램 실행시간에 코드 생성과 실행을 언급한 것은 1960년 McCarthy의
LISP 논문이다. 나중에 Thomson의 1968년에 regex 논문에서 이것은 더욱더 분명했다.(regexes는 실시간으로 기계어 코드로 컴파일해서 실행하는 것 이다. )


JIT이란 용어는 Java를 위해 제임스 고슬링이 컴퓨터 용어로 가져왔다. Aycock 는 고슬링이 [제조의 도메인의 용어](https://en.wikipedia.org/wiki/Just-in-Time_Manufacturing) 를 빌려 와 그것을 1990 년대 초부터 사용하기 시작 했다고 언급하고 있다.

여기까지가 역사적인 애기이다. 만약에 더 관심이 있다면 Aycock의 논문을 읽어보길 바란다. 이제 실제의 예에서 JIT의 정의가 의미하고 있는 것을 보도록 하자.

## JIT - 기계어 코드를 생성하고 실행한다 ##

나는 JIT 기술을 쉽게 설명하기 위해서 두 개의 단계로 구분해서 설명한다.

+   1 단계: 런타임 시간에 [기계어 코드](https://en.wikipedia.org/wiki/Machine_code)를 생성한다.
+   2 단계: 기계어 코드를 런타임 시간에 실행한다.

1단계가 아마 JITing의 도전과제의 99퍼센트 일 것이다. 하지만 이것은 미스테리컬한 부분은 아닌 것이, 컴파일러가 실제로 하는 역할 이기 때문이다.
gcc나 clang과 같이 잘 알려진 컴파일러들의 경우 C/C++ 소스코드를 기계어 코드로 변역해준다. 기계어 코드는 출력 스트림으로 emit 될 것이고, 메모리 공간에 저장될 것이다.
(그리고 사실 gcc와 clang/llvm 둘다 JIT 실행을 위해 메모리에 코드를 유지하기 위한 빌딩 블록을 가지고 있다.)
2단계가 내가 이번 글에서 주로 설명하고 싶은 부분이다.

## 동적 생성코드 실행 ##

현대의 운영체제들은 실행시간에 프로그램에게 허용해주는 부분이 까다롭다. [보호모드](https://en.wikipedia.org/wiki/Protected_mode)의 등장으로 os가 다양한 퍼미션으로 가상메모리의 영역에 접근하는 것을 엄격하게 다루기 때문에,
메모리 다루는데에 있어서 복잡했던 과거는 사라졌다. 보통의 코드에서 당신은 새로운 데이터를 동적으로 힙 영역에 할당 할 것이지만, [OS의 허락없이 힙 영역에서 실행 할 수는 없다.](https://en.wikipedia.org/wiki/Executable_space_protection)

이러한 관점에서 기계어 코드는 그저 바이트 스트림의 데이터에 불과하다. 따라서:

```
unsigned char[] code = {0x48, 0x89, 0xf8};
```

하지만 누가 보냐에 따라서 다른 코드이기도 하다. 누군가에게는 아무것도 표현하지 않는 것 일 수도 있고, 누군가에게는 x86-64 바이너리 인코딩으로 보일 것 이다.

```
mov %rdi, %rax
```

기계어 코드를 메모리에 집어넣는 것은 매우 쉽다. 근데 이걸 어떻게 실행시킬 것 인가?

<h2>약간의 코드를 보도록 하자</h2>

이 글에서 포함하고 있는 나머지 코드들은 POSIX 호환 UNIX계열 OS(특히 Linux)에서 작동한다. 다른 운영체제(Windows와 같은)들은 세부적으로 좀 다를 수 있다.
근데 전반적인 내용은 같다. 참고로 모든 현대의 운영체제들은 같은 것을 구현하기 위한 편한 API를 제공한다.

더 말할 것도 없이, 여기에 우리가 메모리 영역에 동적으로 생성하고 실행할 함수가 있다. C코드로 표현된 이 함수는 매우 쉬운 코드이다.

```
long add4(long num) {
  return num + 4;
}
```

우리의 첫번째 작품이다.(Makefile을 포함한 전체 코드는 이 [저장소](https://github.com/eliben/libjit-samples)에서 받아 볼 수 있다.)

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>

// 주어진 크기의 RWX메모리를 할당하고 그것의 포인터를 반환한다.
// 만약 실패한다면, 에러를 출력할 것이고, NULL을 반환한다.

void* allocate_executable_memory(size_t size){
  void *ptr = mmap(0, size,
                   PROT_READ | PROT_WRITE | PROT_EXEC,
                   MAP_PRIVATE | MAP_ANONYMOUS, -1 , 0);
  if (ptr == (void*)-1){
    perror("mmap");
    return NULL;
  }             
  return ptr;    
}

void emit_code_into_memory(unsigned char* m){
  unsigned char code[] = {
    0x48, 0x89, 0xf8,                    // mov %rdu, %rax
    0x48, 0x83, 0xc0, 0x04,              // add $4, %rax
    0xc3                                 // ret
  };
  memcpy(m, code, sizeof(code));
}

const size_t SIZE = 1024;
typedef long (*Jittedfunc)(long);

void run_from_rwx(){
  void *m = allocate_executable(SIZE);
  emit_code_into_memory(m);

  JittedFunc func = m;
  int result = func(2);
  printf("result = %d\n", result);
}
```

  1.  `mmap`을 사용하여 읽고, 쓰고 실행가능한 메모리 영역을 힙에 할당한다.
  2.  기계어 코드로 구현한  `add4`를 메모리 영역에 복사한다.
  3.  함수포인터를 이용하여 메모리 영역으로부터 코드를 실행한다.

3번째 단계는 기계어코드를 포함하고 있는 메모리 영역이 *실행가능* 일 때만 가능하다는 점에 주목하자. 올바른 퍼미션 설정 없이는
실행 시간에 OS로부터 오류가 날 것이다.(segment fault와 같은..) 이러한 현상은 만약에 m을 `malloc`, `free`와 같은 함수로
할당 했을 때 발생한다.(읽고 쓰기만 가능, 실행은 불가능)

## 여담 -힙, malloc 그리고 mmap ##

눈치빠른 독자들은 반쯤 눈치 챘을 것이다. 이전 섹션에서 나는 `mmap`으로 부터 반환받는 메모리를 "힙 영역 메모리"라고 언급한 적이 있다.
정확하게 이야기 하자면. "힙"은  실행시간에 할당한 메모리를 관리하기 위해 `malloc`과 `free`가 사용하기 위해 지정된 메모리이다.
컴파일러가 절대적으로 관리하기 위한 "스택 영역 메모리"와는 정 반대이다.

말은 그렇게 하지만 사실 영 쉬운 내용은 아니다. 전통적으로(아주 오래 전 부터..) `malloc`은 메모리의 원천으로만 사용되어왔다.
('sbrk'이라는 시스템 콜을 통해서) 오늘날 대부분의 malloc 구현은 OS에 따라 세부구현이 많이 다르지만 `mmap`을 많이 사용한다.
그러나 `mmap`은 보통 큰 메모리 영역을 할당 할 때 쓰고, `sbrk`는 작은 크기의 메모리 영역을 할당 할 때 쓴다. 트레이드-오프는
두 함수가 OS로 부터 메모리를 요청 할 때 상대적인 효율성과 관련이 있다.

그러므로 내 생각에는 `mmap`으로부터 받는 메모리가 힙 영역 메모리라는 것은 잘못된 것은 아니다. 그러므로 계속 이대로 설명하겠다.

## 보안에 좀더 신경을 써보자 ##

위에 있는 코드는 조금 보안적으로 취약한 문제점이 보인다. 그 이유는 바로 RWX(Readable, Writeable, eXecutable) 메모리 영역을 할당을 받았기 때문이다.
- 공격과 exploit하기에는 파라다이스 같은 공간이다. 그러므로 이 부분에 대한 좀더 신경을 써보자.
수정된 코드는 다음과 같다.

```
// RW메모리 영역을 할당한다.
// 만약 실패시 에러를 출력하고, NULL을 반환한다.
// malloc과는 다르게 메모리가 페이지 경계부분에 할당된다. 그래야만 mprotect를 호출하기에 적합하다.

void* alloc_writable_memory(size_t size)
{
  void* ptr = mmap(0, size,
                   PROT_READ | PROT_WRITE,
                   MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
  if(ptr == (void*)-1) {
    perror("mmap");
    return NULL;
  }
  return ptr;
}

// 메모리를 RX퍼미션으로 설정한다. page-aligned되어 있어야 한다.
// 성공하면 0을 반환, 실패하면 에러를 출력하고 -1을 반환한다.

int make_memory_executable(void* m, size_t size){
  if(mprotect(m, size, PROT_READ | PROT_EXEC) == -1) {
    perror("mprotect");
    return -1;
  }
  return 0;
}

// RW메모리를 할당하고, 실행 전에 코드를 emit하고 RX메모리로 세팅한다.
void emit_to_rw_run_from_rx(){
  void *m = alloc_writable_memory(SIZE);
  emit_code_into_memory(m);
  make_memory_executable(m, SIZE);

  JittedFunc func = n;
  int result = func(2);
  printf("result =%d\n", result);
}

```

메모리가 처음에 RW메모리로 할당된다는 점(`malloc`이랑 똑같다.)을 빼면 이전 코드 스닙셋과 모든 면에 있어서 동일하다.
이게 우리가 기계어 코드를 집어넣기 위해 작성해야 되는 모든 것들이다. 만약에 코드가 있다면, `mprotect`를 이용하여
메모리 영역을 RW에서 RX로 변환하고, 더 이상 *쓰기 불가능* 상태로 만들어 실행가능하게 만든다. 결국 효과는 동일하다.
쓰기 가능하면서 실행가능한 부분이 우리 프로그램에서 없다는 점에서 보안적인 관점에서 보았을 때 좋다.

## malloc은 어떤가? ##

아무튼  RW 메모리가 `malloc`에서 제공하는 것과 동일하다면 `mmap` 대신에 `malloc`을 쓰는 것은 어떨까? 맞다 그것 또한 가능하다.
하지만 그 방법은 득보다 실이 더 많다. 그 이유는 보호 비트가 가상 메모리 페이지 경계부분에서만 세팅되기 때문이다. 만약에 `malloc`을 쓴다면
우리는 수동으로 페이지 경계에 정렬되도록 설정해줘야 한다. 또한 `mprotect`는 실제로 요청한 것과 다르게 활성화/비활성화를 실패하므로써 우리가
예상하지 못한 영향을 끼칠수가 있다. `mmap`은 페이지 경계부분에만 할당됨으로써 이러한 것들을 사전에 방지한다.
(`mmap`은 페이지 전체에 할당되게 디자인 되어있다.)

## 마무리짓기 ##

이번 글은 우리가 흔히 JIT이라고 부르는 것을 고수준 관점에 시작해서, 기계어 코드가 동적으로 메모리에 들어가서 실행되는 코드 스닙셋을
실제로 작성해보았다.

이 테크닉은 실제 JIT엔진(예를 들어 LLVM이나 libjit)이 메모리 영역으로부터 실행가능한 기계어코드를 emit하고 실행하는지 보여준다.
이제 남은 것은 이 다른 무언가로부터 이 기계어코드로 합치는 "단순한" 작업뿐이다.

LLVM은 완전한 컴파일러이다. 그래서 실제로 C나 C++코드를 실행시간에 기계어 코드로 번역할 수 있다.(LLVM IR을 통해)  그리고 그것을 실행시킨다.
libjit는 훨씬더 저 수준에서 작동한다. - 즉 컴파일러의 벡엔드로 작동한다.
사실 libjit을 소개한 나의 글 [원문](http://eli.thegreenplace.net/2013/10/17/getting-started-with-libjit-part-1/)에서는 복잡한 코드를
어떻게 emit하고 실행시키는지 보여줬다. 그러나 JITing은 훨씬 더 일반적인 컨셉이다. 실행시간에 코드를 emmiting하는 것은 [자료구조](http://blog.christianperone.com/2009/11/a-method-for-jiting-algorithms-and-data-structures-with-llvm/)나 [정규 표현식](http://sljit.sourceforge.net/)
심지어 [프로그래밍 언어의 VM에서 C언어 접근 할 때](http://luajit.org/ext_ffi.html)도 사용이 가능하다.
게다가 내 블로그([원작자 블로그](http://eli.thegreenplace.net/))를 파다보면 [8년 전에 JITing에 대해 언급한 적](http://eli.thegreenplace.net/2005/09/04/cool-hack-creating-custom-subroutines-on-the-fly-in-perl/)도 있다. 그것은 펄 코드가
펄 코드를 실행시간에 생성하는 것이었지만 아이디어는 동일하다.(직렬화 포멧을 XML 표현으로 가져와서)

JITing을 두 단계로 나누어서 설명하는게 중요하다는게 내 생각이고 이유이다. 2단계는 상대적으로 구현하기가 용이하고 잘 정의된 OS API를 사용한다.
1단계의 가능성은 끝이 없다. 그리고 이 부분은 궁극적으로 여러분이 어떤 어플리케이션을 개발하느냐에 따라 달려있다.
