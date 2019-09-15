---
published: true
title: 'MSVC std::exception'
---

Появилась задача - запустить ПО на одной из unix систем. Платформенно зависимый код предусмотрительно окружен `ifdef`, правила сборки настроены. Что может пойти не так?

Запускаю сборку, занимаюсь важными делами, пока все компилируется. Все идет хорошо, но вдруг кровавый compile error портит все настроение: no matching constructor for initialization of `std::exception`!

```cpp
class SomeException: public exception {
  public:
    SomeException(const char * message) : exception(message) {};  
};
```

Казалось бы, где тут можно ошибиться? Перейдем к определению класса [`std::exception`](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/libsupc%2B%2B/exception.h):

```cpp
// Exception Handling support header for -*- C++ -*-

// Copyright (C) 2016-2019 Free Software Foundation, Inc.
//
// This file is part of GCC.
//
// GCC is free software; you can redistribute it and/or modify
// it under the terms of the GNU General Public License as published by
// the Free Software Foundation; either version 3, or (at your option)
// any later version.
//
// GCC is distributed in the hope that it will be useful,
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
// GNU General Public License for more details.
//
// Under Section 7 of GPL version 3, you are granted additional
// permissions described in the GCC Runtime Library Exception, version
// 3.1, as published by the Free Software Foundation.

// You should have received a copy of the GNU General Public License and
// a copy of the GCC Runtime Library Exception along with this program;
// see the files COPYING3 and COPYING.RUNTIME respectively.  If not, see
// <http://www.gnu.org/licenses/>.

/** @file bits/exception.h
 *  This is an internal header file, included by other library headers.
 *  Do not attempt to use it directly.
 */

#ifndef __EXCEPTION_H
#define __EXCEPTION_H 1

#pragma GCC system_header

#pragma GCC visibility push(default)

#include <bits/c++config.h>

extern "C++" {

namespace std
{
  /**
   * @defgroup exceptions Exceptions
   * @ingroup diagnostics
   *
   * Classes and functions for reporting errors via exception classes.
   * @{
   */

  /**
   *  @brief Base class for all library exceptions.
   *
   *  This is the base class for all exceptions thrown by the standard
   *  library, and by certain language expressions.  You are free to derive
   *  your own %exception classes, or use a different hierarchy, or to
   *  throw non-class data (e.g., fundamental types).
   */
  class exception
  {
  public:
    exception() _GLIBCXX_NOTHROW { }
    virtual ~exception() _GLIBCXX_TXN_SAFE_DYN _GLIBCXX_NOTHROW;
#if __cplusplus >= 201103L
    exception(const exception&) = default;
    exception& operator=(const exception&) = default;
    exception(exception&&) = default;
    exception& operator=(exception&&) = default;
#endif

    /** Returns a C-style character string describing the general cause
     *  of the current error.  */
    virtual const char*
    what() const _GLIBCXX_TXN_SAFE_DYN _GLIBCXX_NOTHROW;
  };

} // namespace std

}

#pragma GCC visibility pop

#endif
```

Все довольно просто, но почему в MSVC есть дополнительный конструктор, принимающий `const char * msg`?
Вспоминаю о чудном ~~(ставьте ударение куда хотите)~~ ресурсе [docs.microsoft.com](https://docs.microsoft.com/en-us/cpp/standard-library/exception-class?view=vs-2019):
> The constructors exception(const char* const &message) and exception(const char* const &message, int) are Microsoft extensions to the C++ Standard Library.

Все встало на свои места, но мне стало интересно, насколько код `std::exception` разнится у этих компиляторов. Версию от gcc я уже приводил выше, пришло время показать версию MSVC:

```cpp
//
// vcruntime_exception.h
//
//      Copyright (c) Microsoft Corporation. All rights reserved.
//
// <exception> functionality that is implemented in the VCRuntime.
//
#pragma once

#include <eh.h>

#ifdef _M_CEE_PURE
    #include <vcruntime_new.h>
#endif

#pragma pack(push, _CRT_PACKING)
#if !defined RC_INVOKED && _HAS_EXCEPTIONS

_CRT_BEGIN_C_HEADER

struct __std_exception_data
{
    char const* _What;
    bool        _DoFree;
};

_VCRTIMP void __cdecl __std_exception_copy(
    _In_  __std_exception_data const* _From,
    _Out_ __std_exception_data*       _To
    );

_VCRTIMP void __cdecl __std_exception_destroy(
    _Inout_ __std_exception_data* _Data
    );

_CRT_END_C_HEADER



namespace std {

class exception
{
public:

    exception() throw()
        : _Data()
    {
    }

    explicit exception(char const* const _Message) throw()
        : _Data()
    {
        __std_exception_data _InitData = { _Message, true };
        __std_exception_copy(&_InitData, &_Data);
    }

    exception(char const* const _Message, int) throw()
        : _Data()
    {
        _Data._What = _Message;
    }

    exception(exception const& _Other) throw()
        : _Data()
    {
        __std_exception_copy(&_Other._Data, &_Data);
    }

    exception& operator=(exception const& _Other) throw()
    {
        if (this == &_Other)
        {
            return *this;
        }

        __std_exception_destroy(&_Data);
        __std_exception_copy(&_Other._Data, &_Data);
        return *this;
    }

    virtual ~exception() throw()
    {
        __std_exception_destroy(&_Data);
    }

    virtual char const* what() const
    {
        return _Data._What ? _Data._What : "Unknown exception";
    }

private:

    __std_exception_data _Data;
};
...
```

Из-за наличия объекта `__std_exception_data` MSVC пришлось отказаться от дефолтных реализаций некоторых функций, но в целом видно, что разница не велика.
