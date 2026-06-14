---
title: "[예제로 배우는 C++ STL] 01강 C++20의 새로운 기능"
excerpt: "예제로 배우는 C++ STL 01강"
date: 2026-06-14 09:00:00 +0900
categories:
  - 공부 기록
tags:
  - C++
  - Modern C++
  - C++20
  - C++23
series: "예제로 배우는 C++ STL"
toc: true
toc_label: "목차"
toc_sticky: true
---

## 새로운 format 라이브러리로 텍스트 서식화하기

### 기존 방법 텍스트 서식화

#### printf

C에서 파생되어 성능과 유연성이 좋다. 하지만 타입 안정성이 낮다.

- C의 가변인자 모델을 사용하여 인자를 포맷터에 넘긴다. 이때 인자의 타입이 서식 지정자와 일치하지 않을 경우 심각한 문제가 생긴다. (왜 생기지????)

#### iostream

타입 안정성은 제공하지만 가독성과 런타임 성능이 저하되는 단점이 있다. 문법과 구현이 복잡하다.

### format 헤더

```cpp
template<typename... Args>
string format(string_view fmt, const Args&... args);

string who{ "everyone" };
int ival { 42 };
double pi{ std::numbers::pi };

// 중복 플레이스홀더가 가능
format("Hello, {}\n", who);       // Hello, everyone!
format("Hello {} {}", ival, who); // Hello 42 everyone
```

#### 출력

```cpp
// iostream
cout << format("Hello, {}", who) << "\n";
// cstdio
puts(format("Hello, {}", who).c_str());
```

추후 C++23부터 `print` 함수 제공.

#### format으로 printf 만들기

```cpp
#include <format>
#include <string_view>
#include <cstdio>

template<typename... Args>
void print(const string_view fmt_str, Args&&... args)
{
    auto fmt_args{ make_format_args(args...) };
    string outstr{ vformat(fmt_str, fmt_args) };
    fputs(outstr.c_str(), stdout);
}
```

- 문자열 서식을 위한 `string_view` 객체
- 파라미터 팩 → `make_format_args()` → 타입이 소거된 값(type-erased)을 포함하는 객체
- 타입이 소거된 값을 포함하는 객체 → `vformat` 함수 → 적절한 문자열 → `fputs`을 통해 출력
- C++23에서는 `using std::print;`로 대체하면 된다

#### 사용자 정의 타입 커스텀

```cpp
struct Frac
{
    long n;
    long d;
};

template<>
struct formatter<Frac>
{
    template<typename ParseContext>
    constexpr auto parse(ParseContext& ctx)
    {
        return ctx.begin();
    }

    template<typename FormatContext>
    auto format(const Frac& f, FormatContext& ctx)
    {
        return format_to(ctx.out(), "{0:d}/{1:d}", f.n, f.d);
    }
};

int main()
{
    Frac f{ 5, 3 };
    print("Frac: {}\n", f);
}

// Frac: 5/3
```

format이 print와 iostream의 단점을 해결했다.

## constexpr로 컴파일 타임에 벡터와 문자열 사용하기

constexpr를 통해서 컴파일 타임에 평가될 수 있어 효율성이 향상된다. (왜?)
`string`과 `vector` 객체는 constexpr 컨텍스트에서 사용할 수 있다.

```cpp
constexpr auto use_string()
{
    string str{ "string" };
    return str.size();
}

constexpr auto use_vector()
{
    vector<int> vec{ 1, 2, 3, 4, 5 };
    return accumulate(begin(vec), end(vec), 0);
}
// accumulate 알고리즘 결과는 컴파일 타임과 constexpr 컨텍스트에서 사용 가능
```

constexpr은 변수나 함수를 컴파일 타임에 평가할 수 있도록 한다.

- 평가할 수 있다는 것은 메모리도 컴파일 타임에 해제되어야 한다는 의미다.
- `vector`를 반환하는 constexpr은 컴파일에서는 문제가 없지만 런타임에서는 오류가 난다. 이는 컴파일 타임에 이미 할당 및 해제가 일어났기 때문이다.
- `size()` 같은 constexpr로 한정된 메서드는 런타임에서 사용할 수 있다.

## 서로 다른 타입의 정수 안전하게 비교하기

```cpp
int x{ -3 };
unsigned y{ 7 };
```

위 두 수 비교 시 부호가 있는 타입을 부호 없는 타입으로 변환시킨다.

- (자세한 것 추가)

그래서 안전하게 정수를 비교하려면 `utility`를 사용해야 한다.

### cmp_less 구현

`make_unsigned_t`를 통해 부호 있는 타입을 안전하게 부호 없는 타입으로 변환하고, 이것을 통해서 비교를 한다.

## 삼중 비교를 위해 우주선 연산자 `<=>` 사용하기

같으면 0, 좌변이 우변보다 크면 음수, 좌변이 우변보다 작으면 양수를 반환한다.
연산자 `<=>`은 표현식 재작성을 사용한다. (이게 뭐지?)

## `<version>` 헤더를 사용하여 기능 시험 매크로 쉽게 찾기

새로운 기능이 추가되면 이를 확인할 수 있는 매크로.

## 컨셉과 제약조건을 통해 더 안전한 템플릿 만들기

템플릿에서 예측하지 못한 타입을 넣었을 때 생기는 문제 때문에 생긴 기능이다.

```cpp
template<typename T>
requires Numeric<T>
T arg42(const T& arg)
{
    return arg + 42;
}
```

- `concepts` 헤더에 정의되어 있다.
- 제약 조건은 컨셉이나 타입 특성을 사용하여 타입의 성격을 평가한다.
- `<type_traits>` 헤더에 있는 미리 정의된 특성들을 사용한다.

#### requires 키워드 사용

```cpp
template<typename T>
requires Numeric<T>
T arg42(const T& arg) {
    return arg + 42;
}
```

#### 템플릿 선언에 컨셉 적용

```cpp
template<Numeric T>
T arg42(const T& arg) {
    return arg + 42;
}
```

#### 함수 시그니처에 requires 키워드 적용

```cpp
template<typename T>
T arg42(const T& arg) requires Numeric<T> {
    return arg + 42;
}
```

#### 축약된 함수 템플릿을 위한 인자 목록에 컨셉 적용

```cpp
auto arg42(Numeric auto& arg) {
    return arg + 42;
}
```

제약조건의 생성에 사용되는 표현식: 결합, 분리, 원자적.
컨셉이나 제약조건은 `&&`, `||`을 사용하여 결합·분리를 할 수 있다.

제약 조건의 **결합**은 두 제약조건과 `&&` 연산자로 이루어진다.

```cpp
template<typename T>
concept Integral_s = Integral<T> && is_signed<T>::value;
```

제약 조건 **분리**는 두 제약조건과 `||` 연산자로 이루어진다.

```cpp
template<typename T>
concept Numeric = Integral<T> || floating_point<T>;
```

**원자적** 제약 조건은 bool 타입을 반환하는 표현식으로, 분해될 수 없는 제약조건을 말한다.

```cpp
template<typename T>
concept is_gt_byte = sizeof(T) > 1;

// 논리적 !(Not) 연산자는 사용 가능하다
template<typename T>
concept is_byte = !is_gt_byte<T>;
```

## 모듈을 사용하여 템플릿 라이브러리의 재컴파일 피하기

헤더 파일은 텍스트 치환 매크로나 번역 단위 사이의 외부 심볼을 링킹하는 목적으로 사용됐다.
하지만 헤더 파일이 늘어날수록 헤더 충돌의 가능성도 커진다. 그래서 모듈로 해결한다.

```cpp
#ifndef BW_MATH
#define BW_MATH
namespace bw {
    template<typename T>
    T add(T lhs, T rhs) {
        return lhs + rhs;
    }
}
#endif // BW_MATH
```

위처럼 하면 모든 헤더에서 추가가 가능하고, 컴파일러에서는 이 충돌을 확인할 수 없다.

```cpp
template<typename T>
T add(T lhs, T rhs) {
    return lhs + rhs;
}
```

위 코드는 템플릿이기 때문에 `add` 함수를 사용하면 별도의 특수화를 생성한다.
그럼 호출될 때마다 파싱하고 특수화가 된다. 이는 대형 템플릿 클래스와 함수가 추가되면 확장성 문제가 생긴다.
모듈은 이 문제를 해결한다.

```cpp
// 모듈 예시
export module bw_math;
export template<typename T>
T add(T lhs, T rhs) {
    return lhs + rhs;
}

/// 사용법
import bw_math;
import std;

int main() {
    double f = add(1.23, 4.56);
    int i = add(7, 42);
    std::string s = add<std::string>("one ", "two");
    std::cout <<
        "double: " << f << "\n" <<
        "int: " << i << "\n" <<
        "string: " << s << "\n";
}
```

`import` 선언은 전처리기 명령어로 사용되면 `#include`처럼 사용할 수 있고, 링킹 단계에서 모듈에 있는 심볼 테이블을 임포트한다.

```cpp
export module bw_math;      // 모듈 자체 선언
export template<typename T> // 모듈 소비자 선언
T add(T lhs, T rhs) {
    return lhs + rhs;
}

// module; 전역 모듈 조각 도입 선언
// 전처리기 지시문만이 전역 모듈 조각에 있어야 한다
module;
#define SOME_MACRO 42
#include <stdlib.h>
export module bw_math;

export int a{ 7 }; // 모듈 소비자에 공개
int b{ 42 };       // 비공개
```

#### 블록 단위 export 가능

```cpp
// 블록 단위로 소비자에게 공개
export {
    int a() { return 7; };
    int b() { return 42; };
}
```

#### namespace export 가능

```cpp
// namespace의 내용 전체 공개
export namespace bw {
    template<typename T>
    T add(T lhs, T rhs) {
        return lhs + rhs;
    }
}
```

#### namespace에 있는 개별 심볼 단위 export도 가능

```cpp
namespace bw {
    export template<typename T>
    T add(T lhs, T rhs) {
        return lhs + rhs;
    }
}
```

#### import 선언으로 소비자는 특정 모듈 import

```cpp
import bw_math;

int main() {
    double f = add(1.23, 4.56);
    int i = add(7, 42);
    std::string s = add<std::string>("one ", "two");
}
```

#### 모듈을 임포트하여 소비자에게 전달하는 익스포트도 가능

```cpp
export module bw_math;
export import std;
// export 키워드는 반드시 import 앞에 와야 한다.
```

#### 전체 사용 예시

```cpp
import bw_math;
using std::cout, std::string, std::format;

int main() {
    double f = add(1.23, 4.56);
    int i = add(7, 42);
    string s = add<string>("one ", "two");
    cout <<
        format("double {} \n", f) <<
        format("int {} \n", i) <<
        format("string {} \n", s);
}
```

## 레인지를 사용하여 컨테이너에 뷰 생성하기

컨테이너 필터링을 하는 새로운 패러다임이다.

- 뷰는 다른 하위 레인지를 변환하는 레인지
- 뷰 어댑터는 레인지를 받아서 뷰 객체를 반환하는 객체

#### 뷰

```cpp
const vector<int> nums{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
auto result = ranges::take_view(nums, 5);
auto result = nums | views::take(5);
for (auto v : result) cout << v << " ";
```

#### 뷰 어댑터

- 뷰 어댑터는 반복 가능하기 때문에, 이것 역시 레인지다.

```cpp
const vector<int> nums{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
auto result = nums | views::take(5) | views::reverse;
```

- `filter()` 뷰는 술어(predicate) 함수를 사용한다.

```cpp
auto result = nums | views::filter([](int i){ return 0 == i % 2; });
```

- `transform()` 뷰는 변형 함수를 사용한다.

```cpp
auto result = nums | views::transform([](int i){ return i * i; });
```

레인지 라이브러리는 몇 개의 레인지 팩토리를 포함한다.

```cpp
auto rnums = views::iota(1, 10);
```

레인지가 되기 위해서는 객체는 최소한 두 개의 반복자 `begin()`, `end()`를 가져야 한다.
여기서 `end()`는 중단점을 결정하는 데 사용되는 센티널이다.
`stack()`과 `queue()`의 경우 `begin()`과 `end()` 반복자가 없다.

## 추가 공부

- C의 가변인자 모델은 무슨 한계가 있길래 이런 문제가 생기지?
- iostream이 런타임 성능이 안 좋은 이유
- `string_view` 객체는 뭐지?
