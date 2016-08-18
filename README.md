to::_string w/ C++
====
__미완__<br>
<br>
https://github.com/pjc0247/to.cpp/blob/master/to.h

컴파일 타임 타입 가지고 놀기
----
C++에는 기본적으로 `타입`이라는 타입이 없고, __RTTI__의 기능은 굉장히 제한적입니다. 하지만 컴파일 타임에는 해당 타입의 여러가지 정보를 가져오거나, 두개의 타입이 같은지 비교하는것도 가능합니다.<br>
이미 __std__네임스페이스 아래에는 수많은 타입 유틸리티들이 포함되어 있으며, `to.cpp` 프로젝트에서는 아래와같은 함수들을 사용하였습니다. (이름이 직관적이기 때문에 별도의 설명은 달지 않도록 하겠습니다.)
* std::is_same
* std::is_pointer

`is_initializer_list`는 기본 제공되지 않기 때문에 아래와 같이 직접 작성하였습니다.
```cpp
template<typename T>
struct is_initializer_list {
    enum { value = false };
};
template<typename T>
struct is_initializer_list<std::initializer_list<T>> {
    enum { value = true };
};
```

SFINAE
----
(__SFINAE__를 위키에 검색하면 거창한 이름이 나오지만, 그런건 별로 중요하지 않고.)<br>
C++에서 어떤 클래스의 메소드 목록을 가져올 수는 없지만, 특정한 이름의 메소드가 존재하는지는 검사할 수 있습니다. 다행히도 `to.cpp`에서는 모든 메소드를 다 들고올 필요는 없고, 해당 클래스에 `to_string()` 메소드가 존재하는지만 검사하면 되기 때문에 별 문제가 되지 않았습니다. 특정 이름의 메소드의 존재 유무를 검사하는것은 SFINAE 기법의 가장 대표적인 쓰임새입니다.

std::enable_if
----
`std::enable_if`는 __SFINAE__기법을 이용한 유틸리티 함수입니다. 

SFINAE++
----
__SFINAE__로 `to_string` 메소드가 있는지 없는지를 검사하는것을 조금 넘어서, 해당 메소드의 리턴 타입이 `std::string`인지도 검사하면 더 좋겠다는 생각을 하였습니다.<br>
아래의 코드는 `decltype`을 이용하여 해당 메소드의 리턴타입을 가져옵니다. 실제로 메소드를 실행시킬것은 아니기 때문에 `(T*)nullptr->`와 같은 꼼수를 이용하면 문법에 어긋나지 않으면서도 실존하는 `this` 없이 메소드를 호출하는 코드를 작성할 수 있습니다.
```cpp
template <typename T, bool A>
struct is_ret_string {
    enum { value = false };
};
template <typename T>
struct is_ret_string<T, true> {
    enum { value = std::is_same<std::string, decltype(((T*)nullptr)->to_string()) >::value };
};
```

constexpr
----
매크로 `CREATE_TO_STRING(...)` 를 이용하면 클래스에 `to_string` 메소드가 자동으로 생성됩니다. <br>
중략.....<br>
`##__VA_ARGS__`를 사용하면 유저가 넘겨준 멤버 목록을 스트링화할 수 잇습니다.<br>
중략2....<br>
해당 스트링을 다시 `,`를 기준으로 잘라내고 싶은데, `##`로 만들어낸 문자열은 컴파일 타임 문자열 리터럴이기 때문에 런타임에 잘라져야 할 필요가 전혀 없습니다. 
```cpp
template <size_t I, size_t OFFSET>
struct tokenizer {
  constexpr static int find(int target, const char a[]) {
    return
      a[I] == ',' ?
        target == 0 ?
          _make_16(I + 1, OFFSET - I) :
          tokenizer<I - 1, I - 1>::find(target - 1, a)
          :
        tokenizer<I - 1, OFFSET>::find(target, a);
  }
};
```

HI+LO byte
----
(적어도 VS2015에서는) __constexpr__이 붙은 함수에는 지역변수를 선언할 수 없다는 제약이 있습니다. `slice`라는 함수를 만들어서 __n__번째 토큰의 시작위치와 길이를 얻어오고 싶은데, 함수가 2개의 값(시작/길이)을 반환하는것은 원래 불가능하고, 그렇다고 `call by ref`를 이용하여 `slice(OUT int &start, OUT int &len)` 처럼 구성하자니 `slice`를 호출하는쪽이 `constexpr`이 될 수 없다는 제약이 생겨버립니다.<br>
중략 ~~~~<br>
8비트 정수(char) 두개를 이어붙여 16비트 정수(short)를 만든 뒤 __short__ 한개를 리턴하는 방법으로 두개의 변수를 반환할 수 있습니다.
```cpp
constexpr unsigned short _make_16(unsigned char a, unsigned char b) {
    return ((unsigned short)((a & 0xff) | ((b & 0xff) << 8)));
}
constexpr unsigned char _lo_8(unsigned short a) {
    return ((unsigned char)(a & 0xff));
}
constexpr unsigned char _hi_8(unsigned short a) {
    return ((unsigned char)(a >> 8) & 0xff);
}
```

tuple unpack
----
`CREATE_TO_STRING(a, b, c, d)` 기능 중, __KEY__ 에 해당하는 `a/b/c/d`를 분리하는것은 위에서 설명한 __constexpr__과 `##`문법으로 해결할 수 있었습니다. 하지만 __VALUE__에 해당하는 부분은 어떻게 처리해야 할까요? __KEY__ 에서는 타입을 걱정할 필요 없이, 통채로 string으로 취급
