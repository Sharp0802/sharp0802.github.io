---
LCID: ko-kr
Layout: default
Title: --nostdlib과 --nostartfiles에 대하여
Timestamps:
- 2023-02-23
- 2023-05-01
Topic: [ "C", "C++", "Optimization", "Depolyment", "Size" ]
---

`--nostdlib`, `--nostartfiles`...

사실, gcc나 clang에 익숙하신 분들도 특정 분야를 제외하곤 이 플래그에 대해서는 생소하실 것이라고 생각한다.

그도 그럴 것이 이 플래그들은 펌웨어나 드라이버를 제작할때나 쓰이는 플래그들로,
평소엔 볼일이 없다는 것이 사실이다.

그럼에도, `--nostdlib`, `--nostartfiles`를 통해 몇가지 이점을 얻을 수 있다.

-   작은 코드 크기
-   가벼운 런타임
-   불필요한 종속성 제거

모두 가벼운 App을 개발하는데에 필요한 조건들이다.

본인 또한 작은 배포 크기에 집착하다 이 플래그를 알게되었고,
이 플래그들을 써오며 골치 아픈 일들이 꽤나 있었기에,
사용법이나 기타 팁들을 공유하고자 한다.

## `--nostdlib`, `--nostartfiles`의 동작?

### 1. `-nostdlib`

먼저, `-nostdlib`은 이름에서도 쉽게 알수 있듯,
표준 라이브러리들을 전부 기본 링크 대상에서 제외하겠다는 플래그다.

`msvcrt`, `ucrt`, `libgcc` 등등 모든 기본 종속성이 제거된다.
따라서 단순히 `-nostdlib` 플래그만 적용시키고 프로젝트를 컴파일하면,
수많은 오류를 뿜어낼 것이다.

#### \_"Undefined symbol: `___chkstk_ms`"\_ 에 대하여

> <picture> 
> <img alt="Warning" src="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/dark-theme/tip.svg">
> </picture><br>
> MinGW에선 <code>___chkstk_ms</code>입니다만,
> Darwin에선 <code>___chkstk_darwin</code>이고,
> 어떤 Linux 배포판에선 그냥 <code>__chkstk</code>이기도 합니다.
> 플랫폼 마다 이름이 조금씩 다르니 유의해 주십시오.

`___chkstk_ms`...
본격적으로 `--nostdlib`을 적용하고,
사용한 모든 함수의 문서를 뒤져가며 필요한 종속성을 추가한 당신에게,
코드에 사용된 적도 없고, 듣도 보도 못한 함수가 정의되지 않았다며 당신을 반길 것이다.

gcc나 clang은 stack-overflow를 방어하기위해 함수 호출시에 같이 호출될 함수를 끼워넣는데, 이것이 바로 `___chkstk_ms`다.
구체적인 원리는 알 필요 없고, 그런게 있다고만 알아두면 된다.

문제는 이것이 `libgcc.a`에 정의되어있다는 것이고, `--nostdlib`는 이 종속성을 제거한다.

간단히 `-lgcc` 플래그를 쓰거나,
`-fno-stack-check` 플래그로 아예 스택 보호를 없애버릴 수도 있다.

가끔 링크 오류나는 김에 `___chkstk_ms`를 아래와 같이 정의해버리는 경우가 있는데,
당연히 작동 안하고, 문서화도 안돼있는지라,
MinGW 레포지토리에서 소스 긁어와서 실험적으로 사용할게 아니라면,
순순히 `-lgcc`나 `-fno-stack-check`를 쓰는 편이 편하다.

```asm
.text
.global ___chkstk_ms
___chkstk_ms:
    ret
```

### 2. `--nostartfiles`

`--nostartfiles`는 크게 두가지 기능을 가지는데,

1. 기본 CRT 종속성을 제거한다.
2. 표준 혹은 사실상 표준으로 자리잡은 모든 초기화 과정을 무시한다.

C언어를 사용중이거나, 2번에 대해 충분히 숙지하고 있다면, 진행해도 좋다.

> <picture> 
> <img alt="Warning" src="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/dark-theme/warning.svg">
> </picture><br>
> 이하 본문은 Windows에서의 시나리오만 다룹니다.
> 타 플랫폼을 이용하시는 분들은 해당 플랫폼 혹은 컴파일러의 문서를 참조해 주십시오.

표준은 아니지만, C컴파일러에 의해 Windows의 Entry Function은 총 4종류다.
사용하는 프로그램이 Console이냐 아니냐에 따라서 `WinMain` 계열과 `main` 계열로 갈리고,
매크로 `WPRFLAG`가 정의되었냐 아니냐에 따라서 `w` 접두사의 유무가 결정된다.

표로 나타내면 다음과 같다.

|         |        `WCS`         |       `MBCS`        |
| ------: | :------------------: | :-----------------: |
| Windows | `wWinMainCRTStartup` | `WinMainCRTStartup` |
| Console |  `wmainCRTStartup`   |  `mainCRTStartup`   |

-   `WPRFLAG`가 정의되었다면 `WCS` 환경,
    정의되지 않았다면 `MBCS` 환경이다.
-   `_WINMAIN_`이 정의되었다면 Windows app,
    정의되지 않았다면 Console app이다.

C/C++에서 Entry point가 `main`이나 `WinMain`인줄 알고있던 사람도 있을 것이다.

엄밀히 따지자면, `mainCRTStartup`또는 그 친구들이 기본 환경을 초기화하고 `main`을 호출하는 것이다.
이는 기본 CRT 종속성에 정의되어 있었으나,
`--nostartfiles`로 없애버렸기 때문에,
원래 `mainCRTStartup`이 하던 일을 일부분만(나머지 일부분은 쓸모없다) 다시해줘야 한다.

원래 `mainCRTStartup`이 해주는 일은 다음과 같다(MSVC 기준).

1. 윈도우 버전 알아내서 전역 변수(`_osver`, `_winminor`, `_winmajor`, `_winver`)에 저장
2. 힙 초기화(`_heap_init`, 내부적으로는 `HeapCreate`를 호출한다)
3. 멀티스레드 초기화(`_mtinit`, `_MT` 매크로가 정의된 경우)
4. Command-line 값, 환경변수 값 알아내서 전역 변수(`_*cmdln`, `_*envptr`, `_*argv`, `_*envp` 등)에 저장
5. `*WinMain` 혹은 `*main` 호출
6. exit code 반환

1, 2, 3번 항목은 웬만하면 쓰일 일이 없어서 해줄 필요가 없다.
애초에 이것들을 직접 써볼만한 실력을 가지고 있다면 이 글을 읽고 있지 않을 것이다.

우리가 직접 해주어야 하는 것은

1. Command-line 값 알아오기
2. exit code 반환

둘 뿐이다.

간단하게 아래와 같이 짜 볼 수 있다.

```c
#include <windows.h>

int mainCRTStartup(void)
{
    int argc = 0;
    LPWSTR* argv = CommandLineToArgvW(GetCommandLineW(), &argc);

    // do stuff...

    LocalFree(argv);
    return 0;
}
```

본래 mainCRTStartup은 try-catch문(기본적으로 SEH)을 통해
표준 Entry point(`main`)를 실행하는 도중 예외가 발생하면 프레임 포인터를 정리하고
오류코드를 반환한 다음 뻗어버리지만,
위처럼 구현하면 stack-overflow라도 생겼을때,
아무것도 정리가 안된채로 segmentation fault 뿜고 뻗는다.

그래서 디버그할때 힘들다.

이때, 스택프레임에 잡히지 않는 대부분의 버그는 stack-overflow에서 생긴다.
배열을 정적할당에서 동적할당으로 전부 바꾸면 대부분의 런타임 에러가 사라진다.

gdb나 lldb에서 스택프레임에 달랑 `___chkstk_ms`하나만 찍혀있고 그 이전은
stack corruption?이라는 메시지와 함께 나오지 않는다면 확실하다.

## 제거되는 종속성 목록

-   MinGW

|              | `-nostartfiles` | `-nodefaultlibs` | `-nostdlib` |
| -----------: | :-------------: | :--------------: | :---------: |
|     `crt2.o` |     trimmed     |                  |   trimmed   |
| `crtbegin.o` |     trimmed     |                  |   trimmed   |
|   `crtend.o` |     trimmed     |                  |   trimmed   |
| `-ladvapi32` |                 |     trimmed      |   trimmed   |
|      `-lgcc` |                 |     trimmed      |   trimmed   |
|   `-lgcc_eh` |                 |     trimmed      |   trimmed   |
|    `-liconv` |                 |     trimmed      |   trimmed   |
| `-lkernel32` |                 |     trimmed      |   trimmed   |
| `-lmoldname` |                 |     trimmed      |   trimmed   |
|  `-lmingw32` |                 |     trimmed      |   trimmed   |
|  `-lmingwex` |                 |     trimmed      |   trimmed   |
|   `-lmsvcrt` |                 |     trimmed      |   trimmed   |
|  `-lpthread` |                 |     trimmed      |   trimmed   |
|  `-lshell32` |                 |     trimmed      |   trimmed   |
|   `-luser32` |                 |     trimmed      |   trimmed   |

-   Linux

|                    | `-nostartfiles` | `-nodefaultlibs` | `-nostdlib` |
| -----------------: | :-------------: | :--------------: | :---------: |
| `/usr/lib/scrt1.o` |     trimmed     |                  |   trimmed   |
|  `/usr/lib/crt1.o` |     trimmed     |                  |   trimmed   |
|  `/usr/lib/crtn.o` |     trimmed     |                  |   trimmed   |
|      `crtbeginS.o` |     trimmed     |                  |   trimmed   |
|        `crtendS.o` |     trimmed     |                  |   trimmed   |
|            `-lgcc` |                 |     trimmed      |   trimmed   |
|          `-lgcc_s` |                 |     trimmed      |   trimmed   |
|              `-lc` |                 |     trimmed      |   trimmed   |

제거되는 종속성에 관한 자세한 내용은 [링크](https://renenyffenegger.ch/notes/development/languages/C-C-plus-plus/GCC/options/no/compare-nostartfiles-nodefaultlibs-nolibc-nostdlib)를 참조하자.

<br/>
<br/>
<br/>
<br/>
<br/>

오류 지적은 언제나 환영중이다.
