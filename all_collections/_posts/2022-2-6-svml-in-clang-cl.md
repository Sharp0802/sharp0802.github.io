---
layout: post
title: "Clang-cl에서 SVML 사용하기"
date: 2022-2-6
categories: [ "C++" "SIMD" ]
---

## 문제

SVML은 Short Vector Math Library라는 이름에 걸맞게, 정말 라이브러리에 불과하다

다르게 말하면, 대다수의 컴파일러에서는 SVML을 기본적으로 지원하지 않는다

MSVC는 SVML을 기본 제공해주고 있지만, Clang-cl을 사용할 경우 SVML을 부가적인 작업을 해주어야 사용할 수 있다

## 해결

[Neumann-A/llvm_svml_intrinsic_generator](https://github.com/Neumann-A/llvm_svml_intrinsic_generator) 레포지토리가 제공해준다

소스코드를 다운로드 받고, generated_code 폴더 내부의 avx_svml_intrin.h와 avx512_svml_intrin.h를 사용하도록 하자

레포지토리는 아래와 같은 헤더를 제공해주는데, 보이는것과 같이 명령어를 직접 어셈블리어로 삽입한다

```cpp
__SVML_INTRIN_PROLOG __m128 __DEFAULT_SVML_FN_ATTRS128
    _mm_sin_ps(__m128 param0) 
{ 
    register __m128 reg0 asm("xmm0") = param0;
    asm( 
         "call __vdecl_sinf4 \t\n" 
        : "=v" (reg0)
        : "0" (reg0)
        : "%ymm1", "%ymm2", "%ymm3", "%ymm4", "%ymm5", "%rax", "%rcx", "%rdx", "%r8", "%r9", "%r10", "%r11"
        );
    return reg0;
}
```