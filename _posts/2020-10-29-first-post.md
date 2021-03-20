---
title: "Pointer magic for efficient dynamic value representations"
date: 2020-10-29 20:31:28 -0400
categories: jekyll update
---
# Pointer magic for efficient dynamic value representations
* nikic의 블로그에 소개된 JavaScript의 실행 속도를 개선할 수 있었던 몇 가지 트릭을 C++을 통해 소개합니다
* 이 방법을 이용하면 C에서도 JavaScript와 같은 방식으로 변수 접근이 가능해지며, 이를 통해 유연하고 효과적인 개발 방식을 적용할 수 있습니다
* https://nikic.github.io/2012/02/02/Pointer-magic-for-efficient-dynamic-value-representations.html#the-trivial-approach-tagged-unions
* 내용은 nikic의 원문에 아주 약간의 내용을 첨부하여 게시하였습니다.

## The trivial approach: Tagged unions
자바 스크립트는 동적으로 입력되는 언어이므로 변수 타입과 해당 변수의 값을 저장하는 일종의 "값" 데이터 구조가 필요합니다. 이러한 접근 방식은 태그가 지정된 공용체를 사용하는 것입니다.
```cpp
#include <stdint.h>

class Value {
public:
    enum TypeTag {
        IntType, DoubleType, StringType, ObjectType
    }

    // constructors, methods, ...
private:
    union {
        int32_t  asInt32;
        double   asDouble;
        String  *asString;
        Object  *asObject;
    } payload;
    TypeTag type;
};
```
보시다시피 매우 간단한 아이디어로 구현이 가능합니다. 현재 값의 타입을 지정하는 타입 태그와 다양한 유형을 저장할 때 사용되는 공용체로 구성이 가능합니다. 위 구조를 이용하여 정수와 객체를 저장하는 예는 아래와 같습니다.
```cpp
inline Value::Value(int32_t number) {
    type = IntType;
    payload = number;
}
inline Value::Value(Object *object) {
    type = ObjectType;
    payload = object;
}
```
이 간단한 접근 방식의 문제점은 모든 개별 값들이 64비트 (8byte) 이하로 저장될 수 있음에도 불구하고 64비트 시스템을 기준으로 위의 Value type의 크기가 128비트 (16byte)로 구성된다는 점( Pointer tagging 참고 )입니다. 특히 이 값의 구조가 배열에 저장되면 오버 헤드를 크게 증가 시킬 것입니다. 배열 외부에서도 이 구조를 단순히 전달하는 것에 64비트 레지스터 두 개가 필요하게 됩니다.


위와 같은 상황에서 이상적으로 페이로드의 크기를 줄이기 위해 Pointer tagging 기술을 살펴보겠습니다.
## An interlude: Pointer tagging
Value 구조를 다시 살펴보겠습니다. 8바이트 크기의 공용체와 1바이트 크기의 정수(TypeTag)로 구성됩니다. 따라서 이론적으로 총 크기는 9바이트입니다. 하지만 실제 메모리에서 구성되는 크기는 16바이트로 구성됩니다. 이는 컴파일러가 메모리에 정렬하기 위해 데이터 구조에 패딩을 추가하기 때문입니다.

이는 성능 상의 이유 때문입니다. CPU는 word size의 chunk로만 메모리에 접근할 수 있습니다. 따라서 데이터가 항상 word의 시작 부분에 있다면 효율적으로 가져올 수 있는 것입니다. word 중간 어딘가에서 시작한다면 CPU는 데이터를 얻기 위해 마스크와 시프트를 적용해야하고 이것은 성능 저하로 연결됩니다.

64비트 시스템에서 word size는 64bit (8byte) 입니다. 따라서 객체에 대한 모든 포인터가 8바이트로 정렬될 것이고 이는 8의 배수로 이루어진다는 것을 의미합니다. 이때 포인터의 값은 ob1000(=8) , ob10000(=16), ob11000(=24), ... , ob11010101001011000(=109144) 와 같이 이루어집니다.

이때 포인터 주소를 가리키는 값의 하위 3비트는 항상 0으로 이루어집니다. 이를 사용하여 0과 7사이의 정수인 "Tag"를 저장할 수 잇습니다.

수행하는 방법에 대한 샘플 구현은 다음과 같습니다.
```cpp
#include <cassert>
#include <stdint.h>

template <typename T, int alignedTo>
class TaggedPointer {
private:
    static_assert(
        alignedTo != 0 && ((alignedTo & (alignedTo - 1)) == 0),
        "Alignment parameter must be power of two"
    );

    // for 8 byte alignment tagMask = alignedTo - 1 = 8 - 1 = 7 = 0b111
    // i.e. the lowest three bits are set, which is where the tag is stored
    static const intptr_t tagMask = alignedTo - 1;

    // pointerMask is the exact contrary: 0b...11111000
    // i.e. all bits apart from the three lowest are set, which is where the pointer is stored
    static const intptr_t pointerMask = ~tagMask;

    // save us some reinterpret_casts with a union
    union {
        T *asPointer;
        intptr_t asBits;
    };

public:
    inline TaggedPointer(T *pointer = 0, int tag = 0) {
        set(pointer, tag);
    }

    inline void set(T *pointer, int tag = 0) {
        // make sure that the pointer really is aligned
        assert((reinterpret_cast<intptr_t>(pointer) & tagMask) == 0);
        // make sure that the tag isn't too large
        assert((tag & pointerMask) == 0);

        asPointer = pointer;
        asBits |= tag;
    }

    inline T *getPointer() const {
        return reinterpret_cast<T *>(asBits & pointerMask);
    }
    inline int getTag() const {
        return asBits & tagMask;
    }
};
```
보시다시피 코드가 상당히 간단한 편이고, 아래와 같이 사용할 수 있습니다.
```cpp
double number = 17.0;
TaggedPointer<double, 8> taggedPointer(&number, 5);
taggedPointer.getPointer(); // == &number
taggedPointer.getTag(); // == 5
```
nikic의 독자로써 첨언하자면,  TaggedPointer 생성자는 생성할 포인터의 타입 명과 위에서 언급해왔듯이 8바이트로 정렬하기 위해 8을 매개변수로 전달하고 있는 것이다. 

위 구조를 통해 정렬된 포인터에 약간의 정보를 추가하여 많은 상황에 적용할 수 있게된다. ( **Sample implementation** 참고)

## Storing integers in the pointer
JavaScript에는 정수 유형 자체가 없으며 IEEE 754 double만 있습니다. 하지만 대부분의 사람들은 그보다 작은 정수로 작업을 합니다( ex : loop , array index 등 ).

이러한 경우 16바이트 구조를 가지는 Value 구조는 비효율적입니다 ( 데이터 4바이트, Tag 1바이트, 11바이트 패딩 상태)


이에 대한 해결책으로는 위에 소개된 TaggingPointer의 구성 방식을 이용해 "포인터의 마지막 비트가 1이면 실제로는 포인터가 아닌 정수입니다" 로 나타내는 것입니다.

예로써 TaggingPointer의 구성을 아래와 같이 합니다. ( 하위 3비트 중 1비트는 0으로, 2비티는 태깅을 위해 사용 )
```
pppppppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppTT0
^- actual pointer                                     two tag bits -^ ^- last bit 0
```
그리고 정수를 표현할 때는 아래와 같이 구성합니다. (하위 1비티는 1로 고정, 나머지는 정수 값으로 채워주는 것 )
```
iiiiiiii|iiiiiiii|iiiiiiii|iiiiiiii|iiiiiiii|iiiiiiii|iiiiiiii|iiiiiii1
^- actual integer                                                     ^- last bit 1
```
그리고 정수 값을 얻을 때는 1비트 shift를 이용하면 됩니다 `(integer >> 1)` 

위 구조에 대한 예제는 아래와 같습니다.
```cpp
#include <cassert>
#include <stdint.h>

template <typename T, int alignedTo>
class TaggedPointerOrInt {
private:
    static_assert(
        alignedTo != 0 && ((alignedTo & (alignedTo - 1)) == 0),
        "Alignment parameter must be power of two"
    );
    static_assert(
        alignedTo > 1,
        "Pointer must be at least 2-byte aligned in order to store an int"
    );

    // for 8 byte alignment tagMask = alignedTo - 1 = 8 - 1 = 7 = 0b111
    // i.e. the lowest three bits are set, which is where the tag is stored
    static const intptr_t tagMask = alignedTo - 1;

    // pointerMask is the exact contrary: 0b...11111000
    // i.e. all bits apart from the three lowest are set, which is where the pointer is stored
    static const intptr_t pointerMask = ~tagMask;

    // save us some reinterpret_casts with a union
    union {
        T *asPointer;
        intptr_t asBits;
    };

public:
    inline TaggedPointerOrInt(T *pointer = 0, int tag = 0) {
        setPointer(pointer, tag);
    }
    inline TaggedPointerOrInt(intptr_t number) {
        setInt(number);
    }

    inline void setPointer(T *pointer, int tag = 0) {
        // make sure that the pointer really is aligned
        assert((reinterpret_cast<intptr_t>(pointer) & tagMask) == 0);
        // make sure that the tag isn't too large
        assert(((tag << 1) & pointerMask) == 0);

        // last bit isn't part of tag anymore, but just zero, thus the << 1
        asPointer = pointer;
        asBits |= tag << 1;
    }
    inline void setInt(intptr_t number) {
        // make sure that when we << 1 there will be no data loss
        // i.e. make sure that it's a 31 bit / 63 bit integer
        assert(((number << 1) >> 1) == number);

        // shift the number to the left and set lowest bit to 1
        asBits = (number << 1) | 1;
    }

    inline T *getPointer() const {
        assert(isPointer());

        return reinterpret_cast<T *>(asBits & pointerMask);
    }
    inline int getTag() const {
        assert(isPointer());

        return (asBits & tagMask) >> 1;
    }
    inline intptr_t getInt() const {
        assert(isInt());

        return asBits >> 1;
    }

    inline bool isPointer() const {
        return (asBits & 1) == 0;
    }
    inline bool isInt() const {
        return (asBits & 1) == 1;
    }
};
```
아래는 예제를 이용했을 때에 대한 사용 방법 입니다.
```cpp
// either a pointer to a double (with a two bit tag) or a 63 bit integer
double number = 17.0;
TaggedPointerOrInt<double, 8> taggedPointerOrInt(&number, 3);
taggedPointerOrInt.isPointer(); // == true;
taggedPointerOrInt.getPointer(); // == &number
taggedPointerOrInt.getTag(); // == 3

taggedPointerOrInt.setInt(123456789);
taggedPointerOrInt.isInt(); // == true
taggedPointerOrInt.getInt(); // == 123456789
```
위 예제를 이용하면 정수는 포인터에 직접 저장되고 double, 문자열 및 객체는 식별 태그를 이용하여 포인터에 저장됩니다. ( o = 0b00 = object, 0b01 = string, 0b10 = double). 또한 boolean은 int와 유사하게 사용할 수 있습니다.

```
00000000|00000000|00000000|00000000|00000000|00000000|00000000|0000b110
                                                actual bool value -^  ^- last bit 0 (as it isn't an int)
                       tag = 3 = 0b11 to identify that it's a bool -^^
```
위 방법들을 통해 처음 소개한 Value 구조의 크기가 64비트로 절반 줄어들게 되었습니다. 그러나, 이에 대한 비용(단점)도 있습니다. Doubles는 포인터로 저장되어야 하며, 이는 힙에 할당 되어야 함을 의미합니다.

만약 Double에 대한 힙 할당도 피할 수 있다면 더 좋지 않을까요?

## Pointers in the NaN-Space
### Recap: How do doubles look like?
JS의 Double은부동 소수점 산술에 대해 IEEE 754 표준을 따릅니다. C++에서 double은 어떤 표준도 따르도록 지정되지 않았지만 IEEE 754도 사용한다고 가정합니다. ( 32bit 시스템의 gcc 2.95.2 버전에서 발생했던 상황 같은 경우는 제외 ref [5] [**Bug 323**](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=323) )

IEEE 754 double은 내부적으로 아래와 같은 비트 시퀀스를 사용하여 표현됩니다
```
seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
 ^- exponent ^- mantissa
^- sign bit
```
소수 결과 값은 기본적으로  `(-1) ^ s * m * 2 ^ e`입니다.
하지만 우리에게 중요한 것은 double이 가질 수 있는 몇 가지의 특별한 값들입니다.
```
Positive zero (0.0):
seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
00000000|00000000|00000000|00000000|00000000|00000000|00000000|00000000
^- sign bit 0 (= +), all other bits also 0

Negative zero (-0.0):
seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
10000000|00000000|00000000|00000000|00000000|00000000|00000000|00000000
^- sign bit 1 (= -), all other 0
```
IEEE는 +0 및 -0 두 개의 0을 정의합니다. 둘 다 지수와 가수가 0이 되었으며 `부호 비트의 값`으로 구별됩니다. 

0의 흥미로운 동작 중하나는 비교에서 동일한 것으로 간주된다는 것입니다 `( 0.0 == -0.0 )`. 숫자가 -0인지 확인하는 유일한 방법은 부호 비트가 설정되어있는지 확인하는 것입니다.

```cpp
inline bool isNegativeZero(double number) {
    return number == 0. && *reinterpret_cast<int64_t *>(&number) != 0;
}

Positive infinity (+INF):
seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
01111111|11110000|00000000|00000000|00000000|00000000|00000000|00000000
 ^- exponent bits all 1                          mantissa bits all 0 -^
^- sign bit 0 (= +)

Negative infinity (-INF):
seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
11111111|11110000|00000000|00000000|00000000|00000000|00000000|00000000
 ^- exponent bits all 1                          mantissa bits all 0 -^
^- sign bit 1 (= -)
```
Infinity에는 모든 지수 비트는 1, 가수 비트는 0으로 설정되어있으며 부호 비트로 구분되는 +INF, -INF가 있습니다.
```
Signaling NaN:
seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
s1111111|11110ppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp
             ^- first mantissa bit 0    everything else is "payload" -^
 ^- exponent bits all 1                 and mustn't be all-zero (as it
^- any sign bit                         would be INF then)

Quiet NaN:
seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
s1111111|11111ppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp
             ^- first mantissa bit 1    everything else is "payload" -^
 ^- exponent bits all 1
^- any sign bit
```
NaN은 숫자가 아닌 값을 나타냅니다 `( ex : 0.0 / 0.0 = NaN ) `. 또한 NaN과 NaN을 비교할 경우 항상 false라는 흥미로운 점도 포함합니다. `( NaN == NaN is flase ) `

NaN에는 모든 지수 비트가 설정되어 있지만 ( 부호비트는 NaN과 관련이 없으므로 제외 ), 가수 부분은 0이 아닌 값으로 구성되어 있습니다.

NaN에는 Signaling NaN과 Quiet NaN 두가지 유형이 존재합니다. 지수 다음의 첫번째 비트로 구분되며 0이면 Signaling, 1이면 Quiet으로 구분됩니다.
Signaling NaN은 이론상 작동 시 유효하지 않은 연산 예외 ( EM_INVALID ) 를 발생시켜야하는 반면, Quiet NaN은 그대로 두어야 합니다.
하지만 실제로 이 기능은 많이 사용되지 않는 듯 합니다. 적어도 MSVC에서는 기본적으로 비활성화 되어있으며, 컴파일러 옵션 + __controlfp() 호출로 활성화해야합니다.

흥미로운 점은 NaN이 가수에 51비트 페이로드를 추가로 인코딩한다는 것입니다. 이 페이로드는 원래 오류 정보를 포함하도록 설계되었습니다.

우리는 이 NaN 페이로드를 활용하여 정수 및 포인터와 같은 다른 것들을 넣어 사용할 것입니다. 

## 64 bit is a lie
64비트 아키텍처에서 포인터의 크기는 분명 64비트입니다.  64비트는 `2^64 = 1.84467441 * 10 ^ 19 `  로, 주소 지정 가능한 메모리 바이트는 16 EiB 또는 17179869184 기가 바이트입니다.

하지만 보통 우리가 사용하는 메모리의 크기는 커봐야 64Gb를 넘기지 않을 겁니다. 일부 고급 서버에서는 4TiB의 메모리를 장착할 수 있겠지만 이 또한 43비트 주소 공간으로 충분히 소화 가능합니다 ( 2^42 = 4TiB).

x86-64 아키텍처는 포인터의 하위 48비트만 사용합니다. 또한 비트 63에서 48은 비트 47의 복사본이어야 합니다. 이 패턴을 따르는 포인터를 표준이라고 합니다.

이로 인해 아래와 같이 이상하게 보이는 주소 공간이 생깁니다.
```
0xFFFFFFFF.FFFFFFFF
         ...        <- Canonical upper "half"
0xFFFF8000.00000000
         ...
         ...        <- Noncanonical
         ...
0x00007FFF.FFFFFFFF
         ...        <- Canonical lower "half"
0x00000000.00000000
```

운영 체제는 일반적으로 아래쪽 절반의 포인터만 응용 프로그램에 할당하고 위쪽 절반은 자체적으로 예약해둡니다. Solaris는 일부 상황(mmap)에서 상위 절반의 주소를 할당합니다. 그러나 이 경우를 무시하고 0x00000000.00000000에서 0x00007FFF.FFFFFFFF 까지의 하반부만 사용된다고 가정합니다 ( 이것 만으로도 140조 이상, 4K 해상도에 대해 N^2 연산이 가능한 수준 )

## Sample implementation

```cpp
#include <stdint.h>
#include <cassert>

inline bool isNegativeZero(double number) {
    return number == 0 && *reinterpret_cast<int64_t *>(&number) != 0;
}

class Value {
private:
    union {
        double asDouble;
        uint64_t asBits;
    };

    static const uint64_t MaxDouble = 0xfff8000000000000;
    static const uint64_t Int32Tag  = 0xfff9000000000000;
    static const uint64_t PtrTag    = 0xfffa000000000000;
public:
    inline Value(const double number) {
        int32_t asInt32 = static_cast<int32_t>(number);

        // if the double can be losslessly stored as an int32 do so
        // (int32 doesn't have -0, so check for that too)
        if (number == asInt32 && !isNegativeZero(number)) {
            *this = Value(asInt32);
            return;
        }

        asDouble = number;
    }

    inline Value(const int32_t number) {
        asBits = number | Int32Tag;
    }

    inline Value(void *pointer) {
        // ensure that the pointer really is only 48 bit
        assert((reinterpret_cast<uint64_t>(pointer) & PtrTag) == 0);

        asBits = reinterpret_cast<uint64_t>(pointer) | PtrTag;
    }

    inline bool isDouble() const {
        return asBits < MaxDouble;
    }
    inline bool isInt32() const {
        return (asBits & Int32Tag) == Int32Tag;
    }
    inline bool isPointer() const {
        return (asBits & PtrTag) == PtrTag;
    }

    inline double getDouble() const {
        assert(isDouble());

        return asDouble;
    }
    inline int32_t getInt32() const {
        assert(isInt32());

        return static_cast<int32_t>(asBits & ~Int32Tag);
    }
    inline void *getPointer() const {
        assert(isPointer());

        return reinterpret_cast<void *>(asBits & ~PtrTag);
    }
};
```

다음 바이너리는 MaxDouble, Int32Tag, PtrTag에 대한 값입니다
 ```
Maximum double (qNaN with sign bit set without payload):
                    seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
0xfff8000000000000: 11111111|11111000|00000000|00000000|00000000|00000000|00000000|00000000

32 bit integer:
                    seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
0xfff9000000000000: 11111111|11111001|00000000|00000000|iiiiiiii|iiiiiiii|iiiiiiii|iiiiiiii
                                                        ^- integer value

Pointer:
                    seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
0xfffa000000000000: 11111111|11111010|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp
                                      ^- 48 bit pointer value
```

## A few more considerations
위의 코드는 추가 작업없이 double에 직접 액세스 할 수 있도록하고 비트 마스크를 사용하여 포인터에 접근합니다 (이것은 Mozilla가 수행하는 작업 방법입니다 ). 대체 구현으로는 여러 방법이 있으며, 포인터에 직접 액세스 할 수 있도록하고 오프셋을 사용하여 두 배로 만들 수 있습니다 ( 즉 0x0001000000000000, double 삽입에 추가하고 검색 할 때 빼는 것 ). 이는 Webkit이 하는 방법입니다.

32비트 머신에서는 64비트 값의 하위 32비트에 직접 접근할 수 있습니다. 따라서 포인터와 정수 모두 어떤 경우에도 직접 접근이 가능합니다. 

반면에 정수 방식이 내장 된 태그 포인터(첫 번째 예)는 32비트 시스템에서 32비트 값만 필요하지만 NaN 방식은 32비트 및 64 비트 환경 모두에서 64비트 값을 필요로 합니다.  따라서 32비트에서는 word * 2 크기로 전달해야합니다. ( 이것이 Mozilla가 이 접근 방식을 fatval이라고 부르는 이유입니다 ).

그래도 전체적인 효율성 측면에서 그만한 가치가 있습니다. int와 double(그리고 boolean과 다른 모든 **작은 값**도 마찬가지입니다)은 힙에 액세스하지 않고도 값에 직접 저장할 수 있습니다.

## Refer
-   [1] [value representation in javascript implementations](http://wingolog.org/archives/2011/05/18/value-representation-in-javascript-implementations)  (article similar to this one, looking more at the general picture than at the concrete implementation)
-   [2] [Mozilla’s New Javascript Value Representation](http://evilpie.github.com/sayrer-fatval-backup/cache.aspx.htm)  (you’ll have to scroll down a bit)
-   [3] WebKit’s implementation:  [declaration](http://trac.webkit.org/browser/trunk/Source/JavaScriptCore/runtime/JSValue.h#L250);  [inline methods](http://trac.webkit.org/browser/trunk/Source/JavaScriptCore/runtime/JSValueInlineMethods.h);  [methods](http://trac.webkit.org/browser/trunk/Source/JavaScriptCore/runtime/JSValue.cpp)
-   [4] [Mozilla’s implementation](http://dxr.lanedo.com/mozilla-central/js/src/jsval.h.html)
- [5] [**Bug 323**](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=323)  -  optimized code gives strange floating point results
