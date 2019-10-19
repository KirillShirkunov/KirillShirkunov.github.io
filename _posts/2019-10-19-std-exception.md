---
published: true
title: 'А не собрать ли нам пример с std::exception?'
---

Задумывались ли вы когда-либо, как выглядит один и тот же код, собранный разными компиляторами?

После статьи [/MSVC-exception](/MSVC-exception) у меня в голове зародилась мысль, насколько разный код мы получим для `msvc` и `gcc`. И вот, пришло время это узнать!

Для этого эксперимента отлично подходит ресурс [godbolt.org](https://godbolt.org/). Он позволяет переключаться между разными версиями компилятора и анализировать изменения в собранном коде.

Рассмотрим тривиальный пример:

```cpp
#include "exception"

int main() {
    throw std::exception();
}
```

## `gcc`

Выберем компилятор `gcc 9.2`, подождем результата компиляции несколько секунд и вуаля, перед нами дизассемблированный код нашего примера:

```asm
std::exception::exception() [base object constructor]:
        push    rbp
        mov     rbp, rsp
        mov     QWORD PTR [rbp-8], rdi
        mov     edx, OFFSET FLAT:_ZTVSt9exception+16
        mov     rax, QWORD PTR [rbp-8]
        mov     QWORD PTR [rax], rdx
        nop
        pop     rbp
        ret
main:
        push    rbp
        mov     rbp, rsp
        push    rbx
        sub     rsp, 8
        mov     edi, 8
        call    __cxa_allocate_exception
        mov     rbx, rax
        mov     rdi, rbx
        call    std::exception::exception() [complete object constructor]
        mov     edx, OFFSET FLAT:_ZNSt9exceptionD1Ev
        mov     esi, OFFSET FLAT:_ZTISt9exception
        mov     rdi, rbx
        call    __cxa_throw

```

Добавим оптимизацию `-O3` (от уровня отптимизации результат в данном случае не меняется):

```asm
main:
        sub     rsp, 8
        mov     edi, 8
        call    __cxa_allocate_exception
        mov     rdi, rax
        mov     QWORD PTR [rax], OFFSET FLAT:_ZTVSt9exception+16
        mov     edx, OFFSET FLAT:_ZNSt9exceptionD1Ev
        mov     esi, OFFSET FLAT:_ZTISt9exception
        call    __cxa_throw
```

Выглядит просто замечательно.

## `msvc`

Переключаемся на `msvc 19.22 x64`:

```asm
std::exception::`RTTI Base Class Descriptor at (0,-1,0,64)' DD imagerel std::exception `RTTI Type Descriptor' ; std::exception::`RTTI Base Class Descriptor at (0,-1,0,64)'
        DD      00H
        DD      00H
        DD      0ffffffffH
        DD      00H
        DD      040H
        DD      imagerel std::exception::`RTTI Class Hierarchy Descriptor'
std::exception::`RTTI Base Class Array' DD imagerel std::exception::`RTTI Base Class Descriptor at (0,-1,0,64)' ; std::exception::`RTTI Base Class Array'
        ORG $+3
std::exception::`RTTI Class Hierarchy Descriptor' DD 00H                            ; std::exception::`RTTI Class Hierarchy Descriptor'
        DD      00H
        DD      01H
        DD      imagerel std::exception::`RTTI Base Class Array'
const std::exception::`RTTI Complete Object Locator' DD 01H                      ; std::exception::`RTTI Complete Object Locator'
        DD      00H
        DD      00H
        DD      imagerel std::exception `RTTI Type Descriptor'
        DD      imagerel std::exception::`RTTI Class Hierarchy Descriptor'
        DD      imagerel const std::exception::`RTTI Complete Object Locator'
_CT??_R0?AVexception@std@@@8??0exception@std@@QEAA@AEBV01@@Z24 DD 00H
        DD      imagerel std::exception `RTTI Type Descriptor'
        DD      00H
        DD      0ffffffffH
        ORG $+4
        DD      018H
        DD      imagerel std::exception::exception(std::exception const &)
_CTA1 ?? std::AVexception DD 01H
        DD      imagerel _CT??_R0?AVexception@std@@@8??0exception@std@@QEAA@AEBV01@@Z24
_TI1 ?? std::AVexception DD 00H
        DD      imagerel virtual std::exception::~exception(void)
        DD      00H
        DD      imagerel _CTA1 ?? std::AVexception
`string' DB 'Unknown exception', 00H ; `string'
const std::exception::`vftable' DQ FLAT:const std::exception::`RTTI Complete Object Locator'  ; std::exception::`vftable'
        DQ      FLAT:virtual void * std::exception::`vector deleting destructor'(unsigned int)
        DQ      FLAT:virtual char const * std::exception::what(void)const 

$T1 = 32
main    PROC
$LN3:
        sub     rsp, 72                             ; 00000048H
        lea     rcx, QWORD PTR $T1[rsp]
        call    std::exception::exception(void)             ; std::exception::exception
        lea     rdx, OFFSET FLAT:_TI1 ?? std::AVexception
        lea     rcx, QWORD PTR $T1[rsp]
        call    _CxxThrowException
$LN2@main:
        add     rsp, 72                             ; 00000048H
        ret     0
main    ENDP

this$ = 16
std::exception::exception(void) PROC                      ; std::exception::exception, COMDAT
$LN3:
        mov     QWORD PTR [rsp+8], rcx
        push    rdi
        mov     rax, QWORD PTR this$[rsp]
        lea     rcx, OFFSET FLAT:const std::exception::`vftable'
        mov     QWORD PTR [rax], rcx
        mov     rax, QWORD PTR this$[rsp]
        add     rax, 8
        mov     rdi, rax
        xor     eax, eax
        mov     ecx, 16
        rep stosb
        mov     rax, QWORD PTR this$[rsp]
        pop     rdi
        ret     0
std::exception::exception(void) ENDP                      ; std::exception::exception

this$ = 48
_Other$ = 56
std::exception::exception(std::exception const &) PROC             ; std::exception::exception, COMDAT
$LN3:
        mov     QWORD PTR [rsp+16], rdx
        mov     QWORD PTR [rsp+8], rcx
        push    rdi
        sub     rsp, 32                             ; 00000020H
        mov     rax, QWORD PTR this$[rsp]
        lea     rcx, OFFSET FLAT:const std::exception::`vftable'
        mov     QWORD PTR [rax], rcx
        mov     rax, QWORD PTR this$[rsp]
        add     rax, 8
        mov     rdi, rax
        xor     eax, eax
        mov     ecx, 16
        rep stosb
        mov     rax, QWORD PTR this$[rsp]
        add     rax, 8
        mov     rcx, QWORD PTR _Other$[rsp]
        add     rcx, 8
        mov     rdx, rax
        call    __std_exception_copy
        mov     rax, QWORD PTR this$[rsp]
        add     rsp, 32                             ; 00000020H
        pop     rdi
        ret     0
std::exception::exception(std::exception const &) ENDP             ; std::exception::exception

this$ = 48
virtual std::exception::~exception(void) PROC                      ; std::exception::~exception, COMDAT
$LN3:
        mov     QWORD PTR [rsp+8], rcx
        sub     rsp, 40                             ; 00000028H
        mov     rax, QWORD PTR this$[rsp]
        lea     rcx, OFFSET FLAT:const std::exception::`vftable'
        mov     QWORD PTR [rax], rcx
        mov     rax, QWORD PTR this$[rsp]
        add     rax, 8
        mov     rcx, rax
        call    __std_exception_destroy
        add     rsp, 40                             ; 00000028H
        ret     0
virtual std::exception::~exception(void) ENDP                      ; std::exception::~exception

tv69 = 0
this$ = 32
virtual char const * std::exception::what(void)const  PROC                    ; std::exception::what, COMDAT
$LN5:
        mov     QWORD PTR [rsp+8], rcx
        sub     rsp, 24
        mov     rax, QWORD PTR this$[rsp]
        cmp     QWORD PTR [rax+8], 0
        je      SHORT $LN3@what
        mov     rax, QWORD PTR this$[rsp]
        mov     rax, QWORD PTR [rax+8]
        mov     QWORD PTR tv69[rsp], rax
        jmp     SHORT $LN4@what
$LN3@what:
        lea     rax, OFFSET FLAT:`string'
        mov     QWORD PTR tv69[rsp], rax
$LN4@what:
        mov     rax, QWORD PTR tv69[rsp]
        add     rsp, 24
        ret     0
virtual char const * std::exception::what(void)const  ENDP                    ; std::exception::what

this$ = 48
__flags$ = 56
virtual void * std::exception::`scalar deleting destructor'(unsigned int) PROC           ; std::exception::`scalar deleting destructor', COMDAT
$LN4:
        mov     DWORD PTR [rsp+16], edx
        mov     QWORD PTR [rsp+8], rcx
        sub     rsp, 40                             ; 00000028H
        mov     rcx, QWORD PTR this$[rsp]
        call    virtual std::exception::~exception(void)             ; std::exception::~exception
        mov     eax, DWORD PTR __flags$[rsp]
        and     eax, 1
        test    eax, eax
        je      SHORT $LN2@scalar
        mov     edx, 24
        mov     rcx, QWORD PTR this$[rsp]
        call    void operator delete(void *,unsigned __int64)               ; operator delete
$LN2@scalar:
        mov     rax, QWORD PTR this$[rsp]
        add     rsp, 40                             ; 00000028H
        ret     0
virtual void * std::exception::`scalar deleting destructor'(unsigned int) ENDP           ; std::exception::`scalar deleting destructor'
```

Что за простыня?! Попробуем оптимизировать с опцией `/O2`:

```asm
std::exception::`RTTI Base Class Descriptor at (0,-1,0,64)' DD imagerel std::exception `RTTI Type Descriptor' ; std::exception::`RTTI Base Class Descriptor at (0,-1,0,64)'
        DD      00H
        DD      00H
        DD      0ffffffffH
        DD      00H
        DD      040H
        DD      imagerel std::exception::`RTTI Class Hierarchy Descriptor'
std::exception::`RTTI Base Class Array' DD imagerel std::exception::`RTTI Base Class Descriptor at (0,-1,0,64)' ; std::exception::`RTTI Base Class Array'
        ORG $+3
std::exception::`RTTI Class Hierarchy Descriptor' DD 00H                            ; std::exception::`RTTI Class Hierarchy Descriptor'
        DD      00H
        DD      01H
        DD      imagerel std::exception::`RTTI Base Class Array'
const std::exception::`RTTI Complete Object Locator' DD 01H                      ; std::exception::`RTTI Complete Object Locator'
        DD      00H
        DD      00H
        DD      imagerel std::exception `RTTI Type Descriptor'
        DD      imagerel std::exception::`RTTI Class Hierarchy Descriptor'
        DD      imagerel const std::exception::`RTTI Complete Object Locator'
_CT??_R0?AVexception@std@@@8??0exception@std@@QEAA@AEBV01@@Z24 DD 00H
        DD      imagerel std::exception `RTTI Type Descriptor'
        DD      00H
        DD      0ffffffffH
        ORG $+4
        DD      018H
        DD      imagerel std::exception::exception(std::exception const &)
_CTA1 ?? std::AVexception DD 01H
        DD      imagerel _CT??_R0?AVexception@std@@@8??0exception@std@@QEAA@AEBV01@@Z24
_TI1 ?? std::AVexception DD 00H
        DD      imagerel virtual std::exception::~exception(void)
        DD      00H
        DD      imagerel _CTA1 ?? std::AVexception
`string' DB 'Unknown exception', 00H ; `string'
const std::exception::`vftable' DQ FLAT:const std::exception::`RTTI Complete Object Locator'  ; std::exception::`vftable'
        DQ      FLAT:virtual void * std::exception::`vector deleting destructor'(unsigned int)
        DQ      FLAT:virtual char const * std::exception::what(void)const 

$T1 = 32
main    PROC                                            ; COMDAT
$LN4:
        sub     rsp, 72                             ; 00000048H
        lea     rcx, QWORD PTR $T1[rsp]
        call    std::exception::exception(void)             ; std::exception::exception
        lea     rdx, OFFSET FLAT:_TI1 ?? std::AVexception
        lea     rcx, QWORD PTR $T1[rsp]
        call    _CxxThrowException
        int     3
$LN3@main:
main    ENDP

this$ = 8
std::exception::exception(void) PROC                      ; std::exception::exception, COMDAT
        lea     rax, OFFSET FLAT:const std::exception::`vftable'
        xorps   xmm0, xmm0
        mov     QWORD PTR [rcx], rax
        mov     rax, rcx
        movups  XMMWORD PTR [rcx+8], xmm0
        ret     0
std::exception::exception(void) ENDP                      ; std::exception::exception

this$ = 48
_Other$ = 56
std::exception::exception(std::exception const &) PROC             ; std::exception::exception, COMDAT
$LN5:
        push    rbx
        sub     rsp, 32                             ; 00000020H
        mov     rbx, rcx
        mov     rax, rdx
        lea     rcx, OFFSET FLAT:const std::exception::`vftable'
        xorps   xmm0, xmm0
        lea     rdx, QWORD PTR [rbx+8]
        mov     QWORD PTR [rbx], rcx
        lea     rcx, QWORD PTR [rax+8]
        movups  XMMWORD PTR [rdx], xmm0
        call    __std_exception_copy
        mov     rax, rbx
        add     rsp, 32                             ; 00000020H
        pop     rbx
        ret     0
std::exception::exception(std::exception const &) ENDP             ; std::exception::exception

this$ = 48
__flags$ = 56
virtual void * std::exception::`scalar deleting destructor'(unsigned int) PROC           ; std::exception::`scalar deleting destructor', COMDAT
$LN9:
        mov     QWORD PTR [rsp+8], rbx
        push    rdi
        sub     rsp, 32                             ; 00000020H
        lea     rax, OFFSET FLAT:const std::exception::`vftable'
        mov     rdi, rcx
        mov     QWORD PTR [rcx], rax
        mov     ebx, edx
        add     rcx, 8
        call    __std_exception_destroy
        test    bl, 1
        je      SHORT $LN6@scalar
        mov     edx, 24
        mov     rcx, rdi
        call    void operator delete(void *,unsigned __int64)               ; operator delete
$LN6@scalar:
        mov     rbx, QWORD PTR [rsp+48]
        mov     rax, rdi
        add     rsp, 32                             ; 00000020H
        pop     rdi
        ret     0
virtual void * std::exception::`scalar deleting destructor'(unsigned int) ENDP           ; std::exception::`scalar deleting destructor'

this$ = 8
virtual std::exception::~exception(void) PROC                      ; std::exception::~exception, COMDAT
        lea     rax, OFFSET FLAT:const std::exception::`vftable'
        mov     QWORD PTR [rcx], rax
        add     rcx, 8
        jmp     __std_exception_destroy
virtual std::exception::~exception(void) ENDP                      ; std::exception::~exception

this$ = 8
virtual char const * std::exception::what(void)const  PROC                    ; std::exception::what, COMDAT
        mov     rdx, QWORD PTR [rcx+8]
        lea     rax, OFFSET FLAT:`string'
        test    rdx, rdx
        cmovne  rax, rdx
        ret     0
virtual char const * std::exception::what(void)const  ENDP                    ; std::exception::what
```

Это же просто ужасно, столько команд ради такого скромного функционала...

## `clang`

Для полноты картины добавим `x86-x64 clang 9.0`:

```asm
main:                                   # @main
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     dword ptr [rbp - 4], 0
        mov     edi, 8
        call    __cxa_allocate_exception
        mov     rcx, rax
        mov     rdi, rcx
        mov     qword ptr [rbp - 16], rax # 8-byte Spill
        call    std::exception::exception() [base object constructor]
        movabs  rax, offset typeinfo for std::exception
        movabs  rcx, offset std::exception::~exception() [complete object destructor]
        mov     rdi, qword ptr [rbp - 16] # 8-byte Reload
        mov     rsi, rax
        mov     rdx, rcx
        call    __cxa_throw
std::exception::exception() [base object constructor]:                    # @std::exception::exception() [base object constructor]
        push    rbp
        mov     rbp, rsp
        movabs  rax, offset vtable for std::exception
        add     rax, 16
        mov     qword ptr [rbp - 8], rdi
        mov     rcx, qword ptr [rbp - 8]
        mov     qword ptr [rcx], rax
        pop     rbp
        ret
```

Добавим оптимизацию `-O2`:

```asm
main:                                   # @main
        push    rax
        mov     edi, 8
        call    __cxa_allocate_exception
        mov     qword ptr [rax], offset vtable for std::exception+16
        mov     esi, offset typeinfo for std::exception
        mov     edx, offset std::exception::~exception() [complete object destructor]
        mov     rdi, rax
        call    __cxa_throw
```

Очень приятный результат, особенно после `msvc`.
