---
layout: post
title: "x86에서 스택의 맨 위는 어디일까? "
date: 2015-10-06 00:57:00
categories: "Assembly"
---
우선 이 글은 [Eli Bendersky의 블로그](http://eli.thegreenplace.net/)에서 번역해 온 글임을 알려드립니다.

상당한 의역이 난무하므로 지적사항이 있으면 따로 알려주시면 감사하겠습니다.

[원문](http://eli.thegreenplace.net/2011/02/04/where-the-top-of-the-stack-is-on-x86)이 궁금하신 분은 링크를 타고 들어가셔서 읽어보시면 될 것 같습니다.

번역에 대한 허락은 원저작자에게 받았으며, 이 글에 대한 저작권은 원저작자에게 있음을 알려드리며,

번역 된 글을 다른 곳에 배포하거나 수정 할 시에는 저에게 연락을 주시면 감사하겠습니다.

-----------------------------------

최근에 몇몇 프로그래머들이 x86구조에서 스택이 자라는 방향과 "스택의 맨 윗 지점"과 "스택의 바닥"의 의미를 혼동스러워 하는것을 알게 되었다. 그리고 그것은 사람들이 스택과 x86에서 실제로 스택이 하는 행동에 대한 잘못된 이해에서 오게 된 것이다.

이번 글은 약간의 다이어그램을 통해 이러한 혼동을 풀어주려고 한다.

## 스택의 정의 ##

기본으로 돌아가보자. 새로 배우는 학생들에게는 스택을 쌓인 접시에 비유하곤 한다. 당신은 접시를 접시 더미에 올릴 거나 접시 더미에서 빼낸다. 스택의 맨 위는 당신이 새로운 접시를 올릴 자리이거나 빼올 자리이다.

<center>![plates](/images/plates.png)</center>
## 하드웨어 스택 ##

컴퓨터에서 스택은 보통 메모리의 구역으로 특별하게 다루어진다. 추상적인 느낌에서 적용해보자면, 여러분은 스택의 맨 윗 부분에 데이터를 밀어넣고 맨 윗부분에서 꺼내온다. 다만 이것은 메모리 상에서 스택의 맨 윗 부분이 어디인지는 따지지 않는다는 것은 명심하라.

## x86에서의 스택 ##

혼돈의 원천를 사실에 비추어 볼 때, 인텔의 x86 구조는 "아래로 향한다". 어딘지는 모르겠지만 임의의 주소지부터 더 낮은 주소지로 아래로 자란다.
보통은 이렇게 생겨먹었다:

<center>![stack1](/images/stack1.png)</center>

그러므로 우리가 "스택의 맨 위"를 x86에서 말할 때는 사실은 스택이 차지하고 있는 공간의 메모리 주소 중 가장 낮은 곳을 말하는 것이다.
이게 몇몇 사람들에게는 부자연스러울 수 있다. 위 에 있는 그림을 계속보다 보면 익숙해질 것 이다.

이제 x86 어셈블리 프로그래밍에서 일반적으로 작동하는 방식에 대해서 한번 보도록 하자.

## 스택 포인터에 데이터를 Push하고 Pop하기 ##

x86구조에서 스택과 함께 작동하는 특별하 레지스터가 예약되어 있다. 바로 ESP(Extended Stack Pointer)이다. ESP는 그 의미처럼 항상 스택의 상단부에 있다.

<center>![stack2](/images/stack2.png)</center>

이 그림에서 `0x9080ABCC`는 스택의 최상단 부이다. "foo"의 일부와 ESP는 `0x9080ABCC` 메모리 주소 지점을 가지고 있다. 즉 가리키고 있다는 말이다.

새로운 데이터를 스택에 push하기 위해서는 `push` 명령어를 사용하면 된다. `push`가 처음 하는 것은  `esp`를 4만큼 감소 시키는 것이다. 그리고 나서 `esp`가 가리키는 지점에 저장한다. 즉:

```
push eax
```

이 코드는 다음 코드와 동등하다:

```
sub esp, 4
mov [esp], eax
```

이전 그림에 이어서 진행하자면 스택에 `push`가 된 후, `eax`는 `0xDEADBEEF`의 값을 가지고 있다.

<center> ![stack3](/images/stack3.png) </center>

비슷하게  `pop` 명령어는 스택의 맨 상단부의 값을 없애고 스택 포인터를 증가시킨다.
```
pop eax
```

이 코드와 동일하다.

```
mov eax, [esp]
add esp, 4
```

그럼 다시, 이전 그림에서(`push`를 한 직후) 계속 이어가서, `pop eax`는 다음과 같이 행동한다.

<center>![stack4](/images/stack4.png)</center>

그리고 `0xDEADBEEF`는 `eax`에 쓰여질 것이다. 우리가 따로 덮어 쓰지 않는 한 `0xDEADBEEF`는 계속 `0x9080ABC8`지점에 계속 머물러있다는 점 또한 알아둬야 한다.

## 스택 프레임과 호출 규약##

C에서 생성된 어셈블리 코드를 보면, 우리는 흥미로운 패턴을 많이 볼 수 있다. 스택을 이용해서 매개변수가 함수 안으로 전달되고 지역 변수가 스택 안에 할당되는 것이다.

조그만한 C 프로그램을 하나 보여주겠다.

```
int foobar(int a, int b, int c)
{
  int xx = a + 2;
  int yy = b + 3;
  int zz = c + 4;
  int sum = xx + yy + zz;

  return xx * yy * zz + sum;
}

int main()
{
  return foobar(77, 88, 99);
}

```
`foobar`에 전달되는 매개변수와 함수의 지역변수는 `foobar`이 호출된 스택 안에 저장된다. 스택안의 데이터 집합을 *프레임* 이라고 부른다. `return` 문이 실행되기 전에 `foobar`의 스택 프레임은 다음과 같다.

<center>![stackframe1](/images/stackframe1.png)</center>

초록색 영역은 호출 규약에 따라 스택에 집어넣어진 부분이고, 파란 영역은 `foobar` 함수 그 자체이다. `gcc`로 컴파일해서 어셈블리를 다음과 같이 하면 된다.

```
gcc -masm=intel -S z.c -o z.s
```
다음 어셈블리 코드는 `foobar`를 변환한 것이다. 이해하기 쉽도록 주석을 많이 달아놨다.

```
_foobar:
    ; ebp는 호출 이전에 보존 되어야 한다.
    ; 함수가 변경되기 때문에 저장되어야 한다.
    push ebp

    ; 이제부터 ebp는 현재 스택의 프레임을 가르킨다.
    ; 함수의 프레임
    mob ebp, esp

    ; 스택 내 함수의 지역 변수를 위해 미리 공간을 확보한다.
    ;
    sub esp, 16

    ; eax <-- a, eax += 2, xx에 eax를 저장
    mov eax, DWORD PTR [ebp+8]
    add eax, 2
    mov DWORD PTR [ebp-4], eax

    ; eax <-- b, eax +=3, yy에 eax를 저장
    mov eax, DWORD PTR[ ebp+12]
    add eax, 3
    mov DWORD PTR [ebp-8], eax

    ; eax <-- c, eax +=4, zz에 eax를 저장
    mov eax, DWORD PTR [ebp+16]
    add eax, 4
    mov DWORD PTR [ebp-12], eax

    ; xx + yy + zz를 계산하고 sum에 저장한다.
    mov eax, DWORD PTR [ebp-8]
    mov edx, DWORD PTR [ebp-4]
    lea eax, [edx+eax]
    add eax, DWORD PTR [ebp-12]
    mov DWORD PTR[ebp-16], eax

    ; eax에 최종값을 계산하고
    ; 반환 할 때까지 유지한다.
    mov   eax, DWORD PTR [ebp-4]
    imul  eax, DWORD PTR [ebp-8]
    imul  eax, DWORD PTR [ebp-12]
    add   eax, DWORD PTR [ebp-16]

    ; leave명령어는 아래의 명령어와 동일하다
    ; mov esp, ebp
    ; pop ebp
    ;
    ; ebp를 복구하고 할당된 지역을 해제한다
    ;
    levae
    ret
```

`esp`를 움직이면서 함수를 실행하기 때문에 `ebp`(다른 아키택쳐에서는 프레임 포인터라고도 알려져 있는 베이스 포인터이다.) 함수의 매개변수와 지역변수를 찾기 쉽게 해주는 상대적인 지표로 사용된다. 지역변수가 `ebp` 하단에 있을 때여 매개변수는 스택 내의 `ebp` 상단에 있다.(그러므로 접근할 때에는 양수의 오프셋을 가진다.)
