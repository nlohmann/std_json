| Document Number | ???                                       |
|-----------------|-------------------------------------------|
| Date            | ???                                       |
| Project         | Programming Language C++, Library Working Group |
| Reply-to        | Niels Lohmann <<mail@nlohmann.me>><br>Mario Konrad <<mario.konrad@gmx.net>> |

# JSON parser and generator library

<a name="introduction"></a>
## Introduction

This paper presents a proposal for *JavaScript Object Notation* [RFC7159] parsing and generation.
It proposes a library extension.


<a name="toc"></a>
## Table of Contents

- [Motivation](#motivation)
- [Example](#example)
- [Scope](#scope)
- [Terminology](#terminology)
- [Technical Specification](#tech-spec)
  - [Header `<json>` synopsis](#header-synopsis)
  - [Class Template `basic_json`](#class-basic_json)
    - [Enumeration `value_t`](#class-basic_json-enum-value_t)
    - [Construction](#class-basic_json-construction)
    - [Destruction](#class-basic_json-destruction)
    - [Modifying Operators](#class-basic_json-modifying-operators)
    - [Non-Modifying Operators](#class-basic_json-non-modifying-operators)
    - [Query Member Functions](#class-basic_json-query-member-functions)
    - [Element Access](#class-basic_json-element-access)
    - [Container Access](#class-basic_json-container-access)
    - [Container Operations](#class-basic_json-container-operations)
    - [Serialization / Deserialization](#class-basic_json-serialization)
  - [Nested Class `basic_json::json_pointer`](#class-json_pointer)
  - [Free Functions `to_json`](#func-to_json)
  - [Free Functions `from_json`](#func-from_json)
  - [User Defined Literals](#func-user-defined-literals)
  - [Free Factory Functions](#func-factory)
  - [Template Function Specialization `swap`](#func-swap)
  - [Template Specialization `hash`](#func-hash)
- [Acknowledgements](#acknowledgements)
- [References](#references)


<a name="motivation"></a>
## Motivation

Data represented in JSON format is very widely used for serialization of data. Prominent uses are the numerous frameworks used by websites for asynchronous communication, as well as configuration data.

The format itself is human readable and very lightweight.

There are numerous libraries written in C and C++, usable by C++, but none in the standard.

The goal should be a library extension that fits well into the standard library, customizable and with an user friendly interface.


<a name="example"></a>
### Example: representation of data in JSON and handling in C++

JSON data:

```js
{
    "pi": 3.141,
    "flag": true,
    "name": "Niels Lohmann",
    "nothing": null,
    "answer": {
        "everything": 42
    },
    "list": [0, 1, 2],
    "object": {
        "currency": "USD",
        "value": 42.99
    }
}
```

Access in C++:

```cpp
json data;

// floating point numbers
data["pi"] = 3.141;

// booleans
data["flag"] = true;

// strings
data["name"] = "Niels Lohmann";
data["name"] = std::string{"Niels Lohmann"};

// null object
data["nothing"] = nullptr;

// object inside another object
data["answer"]["everything"] = 42;

// array of objects
data["list"] = { 0, 1, 2 };

// object, defined as list of pairs
data["object"] = {{ "currency", "USD" }, { "value", 42.99 }};
```

or a more direct representation in C++:

```cpp
json j = {
    { "pi", 3.141 },
    { "flag", true },
    { "name", "Niels Lohmann" },
    { "nothing", nullptr },
    { "answer",
        { "everything", 42 }
    },
    { "list", { 0, 1, 2 }},
    { "object", {
        { "currency", "USD" },
        { "value", 42.99 }
    }}
};
```


<a name="scope"></a>
## Scope

This extension is a pure library extension. It does not require changes to the standard components. The extension can be implemented in C++11, C++14, and C++17.


<a name="terminology"></a>
## Terminology

The terminology used in this paper is intended to be consistent with terms used in the JSON world [RFC7159].

The term *value* [RFC7159 / chapter 3] refers to an entity that represents information and can be one of the following:

- object
- array
- number
- string
- boolean
- null

Values for *boolean* can be

- true
- false

The term *object* [RFC7159 / chapter 4] refers to an structured entity, consisting of zero or more name/value pairs. The term *name* refers to the key that identifies an entity. This is always of type *string*.

The term *array* [RFC7159 / chapter 5] refers to an structured entity, consisting of zero or more values of any type.

The term *number* [RFC7159 / chapter 6] represents a numerical value and is one of the following types:

- integral signed
- integral unsigned
- floating point

Speical cases like *inf* and *NaN* are not permitted.


<a name="tech-spec"></a>
## Technical Specification

**TODO**


<a name="header-synopsis"></a>
### Header `<json>` synopsis

```cpp
namespace std {
inline namespace json_v1 {

    // generic base type basic_json
    template < /*omitted*/ > class basic_json;

    // function templates `to_json` for various types, customization points
    // ...

    // function templates `from_json` for various types, customization points
    // ...

    // factory functions
    template < /*omitted*/ > basic_json make_json( /*omitted*/ );

    // default json object class
    using json = basic_json<>;

    // user defined literals
    json               operator "" _json(const char *, std::size_t);
    json::json_pointer operator "" _json_pointer(const char *, std::size_t);
}

    // swap
    template <> void swap(json_v1::json &, json_v1::json &);

    // hash
    template <> struct hash<json_v1::json>
    {
        std::size_t operator()(const json_v1::basic_json &) const;
    };
}
```


<a name="class-basic_json"></a>
### Class Template `basic_json`

```cpp
template <
    template<typename U, typename V, typename... Args> class ObjectType = std::map,
    template<typename U, typename... Args> class ArrayType = std::vector,
    class StringType = std::string,
    class BooleanType = bool,
    class NumberIntegerType = std::int64_t,
    class NumberUnsignedType = std::uint64_t,
    class NumberFloatType = double,
    template<typename U> class AllocatorType = std::allocator
    >
class basic_json
{
public:
    // content type aliases

    using name_type = std::string;
    using string_type = StringType;
    using boolean_type = BooleanType;
    using integral_signed_type = NumberIntegerType;
    using integral_unsigned_type = NumberUnsignedType;
    using floating_point_type = NumberFloatType;

    using object_type = ObjectType<StringType,
        basic_json,
        std::less<StringType>,
        AllocatorType<std::pair<const StringType, basic_json>>>;

    using array_type = ArrayType<basic_json, AllocatorType<basic_json>>;

    // common type aliases

    using value_type = basic_json;
    using reference = value_type&;
    using const_reference = const value_type&;
    using difference_type = std::ptrdiff_t;
    using size_type = std::size_t;

    using allocator_type = AllocatorType<basic_json>;
    using pointer = typename std::allocator_traits<allocator_type>::pointer;
    using const_pointer = typename std::allocator_traits<allocator_type>::const_pointer;

    // nested classes and types

    class const_iterator          = /*implementation defined*/ ;
    class iterator                = /*implementation defined*/ ;
    class const_reverse_iterator  = /*implementation defined*/ ;
    class reverse_iterator        = /*implementation defined*/ ;

    class json_pointer;

    enum class value_t /*omitted*/;

    // constructors ...

    // destructor ...

    // modifying operators ...

    // non-modifying operators ...

    // query member functions ...

    // element access ...

    // container access ...

    // container operations ...

    // serialization ...
};
```


<a name="class-basic_json-enum-value_t"></a>
#### Enumeration `value_t`

```cpp
enum class value_t : /*unspecified*/
{
    null                     = /*unspecified*/ ,
    object                   = /*unspecified*/ ,
    array                    = /*unspecified*/ ,
    string                   = /*unspecified*/ ,
    boolean                  = /*unspecified*/ ,
    number_integral_signed   = /*unspecified*/ ,
    number_integral_unsigned = /*unspecified*/ ,
    number_floating_point    = /*unspecified*/ ,
    discarded                = /*unspecified*/
};
```


<a name="class-basic_json-construction"></a>
#### Construction

```cpp
basic_json() noexcept;
```

*Effect:* Default constructor.

*Postcondition:* The type of the JSON value is `value_t::null`.

*Remarks:* Is trivially default constructible.

*Throws:* Nothing.

```cpp
explicit basic_json(value_t) noexcept;
```

*Effect:* Constructs an empty JSON value with the specified type. The contained data
will be initialized, according to the following table:

  Type                       | Value
  -------------------------- | ----------
  `value_t::null`            | -
  `value_t::object`          | Default constructed container of type `ObjectType`
  `value_t::array`           | Default constructed container of type `ArrayType`
  `value_t::string`          | Default constructed string of type `StringType`
  `value_t::boolean`         | `false`
  `value_t::number_integer`  | `0`
  `value_t::number_unsigned` | `0`
  `value_t::number_float`    | `0.0`
  `value_t::discarded`       | -

*Throws:* Nothing.

*Complexity:* Constant.

```cpp
basic_json(std::nullptr) noexcept;
```

*Effect:* Constructs an empty JSON value of type `value_t::null`.

*Remarks:* Has the same effect as the default constructor.

*Throws:* Nothing.

```cpp
template <typename CompatibleType, /*omitted*/ >
basic_json(CompatibleType &&) noexcept( /*omitted*/ );
```

*Effect:* Constructs a JSON value from the specified data. This constructor utilizes
the `to_json` functions, which act as a customization point. The specified argument
is forwarded.

*Preconditions:*
- In order to be compatible, a function `to_json` must exist for the specific type.
  See chapter [Free Functions `to_json`](#func-to_json).
- Not allowed for `CompatibleType` are:
  - `basic_json`, to avoid hijacking copy/move constructors.
  - Nested types of `basic_json`.

*Postcondition:* If not exception was thrown, the constructed JSON value has a valid
type, e.g. any of `value_t`.

```cpp
basic_json(size_type count, const basic_json & value);
```

*Effect:* Constructs a JSON value with `count` number of elements of the specified
JSON value. The resulting type of the current object is `value_t::array`.

*Postcondition:*
- The JSON value is of type `value_t::array` and contains `count` copies
  of the specified value.
- If the specified `count` is `0`, the resulting JSON value is of type `value_t::array`,
  with an empty container.

*Throws:* `std::bad_alloc` if the container to hold the copies could not be constructed.

*Complexity:* Linear in number of elements.

```cpp
template <typename InputIterator, /*SFINAE omitted*/ >
basic_json(InputIterator first, InputIterator last);
```

*Effect:* Constructs a JSON value from another JSON value with copying data in the specified
range `[first,last)`.

*Precondition:*
- `InputIterator` must be one of the following types:
  - `basic_json::iterator`
  - `basic_json::const_iterator`
- Both specified iterators must be valid, i.e. originate from the same JSON value.
- If the iterators originate from a primitive JSON value, `first` must be the
  result of `origin.begin()` and `last` must be the result of `origin.end()`.

*Throws:*
- `std::domain_error` if the iterators are not compatible according to the preconditions.
- `std::domain_error` if the iterators originate from a JSON value of type `value_t::null`.
- `std::out_of_range` if the iterators originate from a primitive JSON value and do
  not meet the precondition for `first` and `last`.
- `std::bad_alloc` if the allocation of memory for the types `value_t::object`, `value_t::array`
  or `value_t::string` failes.

*Complexity:*
- For structured types (`value_t::object`, `value_t::array`): linear in number of number
  of elements in the range `[first,last)`.
- For primitive types: constant.

```cpp
basic_json(const basic_json &);
```

*Effect:* Copy constructor.

*Throws:* `std::bad_alloc` if the allocation for object, array and string type failed.
Will never be thrown if the specified JSON value is primitive.

```cpp
basic_json(basic_json &&) noexcept;
```

*Effect:* Move constructor.

*Postcondition:* The moved-from JSON value will be left as type `value_t::null`.

*Throws:* Nothing.

*Complexity:* Constant.

```cpp
basic_json(std::initializer_list<basic_json> init,
           bool automatic_type_deduction = true,
           value_t type_override = value_t::array);
```

*Effect:* Constructs a JSON value initalized with the specified parameters.
The type of the JSON value will be `value_t::array` or `value_t::object`, depending on the
specified of the parameters, according to the following rules:

1. If the initializer list is empty, an empty JSON value of type `value_t::object` is created.
2. If the initializer list consists of pairs whose first element is a string, a JSON value
   of type `value_t::object` is created, containing elements constructed from the pairs,
   with their first value (string) as key and the second value as data.
3. In all other cases, a JSON value of type `value_t::array` is created, containing copies
   of the elements in the initializer list.

The rules aim to create the best fit between a C++ initializer list and
JSON values. The rationale is as follows:

- The empty initializer list is written as `{}`, which is exactly an empty JSON object.
- C++ has no way of describing mapped types other than to list a list of pairs. As JSON
  requires that keys must be of type string, rule 2 is the weakest constraint one can pose
  on initializer lists to interpret them as an object.
- In all other cases, the initializer list could not be interpreted as JSON object type,
  so interpreting it as JSON array type is safe.

Using the parameters, it is possible to control the construction of the JSON value manually.
The parameter `automatic_type_deduction` controls weather or not the resulting type is enforced.
The value of the parameter `automatic_type_deduction` controls the following behaviour:
- `true`: the constructor uses the rules above to determine the type for the JSON
  value (`value_t::object` or `value_t::array).
- `false`: the constructor tries to use the type defined by `type_override`, which must be
  either `value_t::object` or `value_t::array`. If it is `value_t::object`, rule 2 must
  be valid or an exception is thrown.

*Throws:*
- `std::domain_error`: if the constructor is forced to construct a JSON value of
  type `value_t::object`, but is unable to because the initializer list does not contain
  pairs with the first element (key) a string.
- `std::bad_alloc`: if the underlying container can not be constructed, containing copies
  of the elements in the initializer list.

*Complexity:* The same complexity to create the underlying data structure defined
by `ArrayType` or `ObjectType`, and linear time in size of the initializer list. 

```cpp
static basic_json array(std::initializer_list<basic_json> = std::initializer_list<basic_json>{});
```

*Effect:* Factory method. Creates and returns a JSON value of type `value_t::array` containing
the elements specified by the initializer list, containing JSON values.

*Remarks:* This member function is needed to disambiguate the creation of JSON values of
type `value_t::array` and `value_t::object` for empty initializer lists and initializer lists
containing pairs.

*Complexity:* The same complexity to create the underlying data structure defined by `ArrayType`
and linear time in size of the initializer list. 

```cpp
static basic_json object(std::initializer_list<basic_json> = std::initializer_list<basic_json>{});
```

*Effect:* Factory method. Creates and returns a JSON value of type `value_t::object` containing
the elements specified by the initializer list, containing JSON values.

The initializer list must contain pairs, the type of the first element must be a string.

*Remarks:* This member function is needed to disambiguate the creation of JSON values of
type `value_t::array` and `value_t::object` for empty initializer lists and initializer lists
containing pairs.

*Throws:*
- `std::domain_error` if the a key of the key-value pairs is not a string.
  it is not possible to create all pairs.

*Complexity:* The same complexity to create the underlying data structure defined by `ObjectType`
and linear time in size of the initializer list. 


<a name="class-basic_json-destruction"></a>
#### Destruction

```cpp
~basic_json();
```

*Effect:* Destroys the JSON value and frees all allocated memory.


<a name="class-basic_json-modifying-operators"></a>
#### Modifying Operators

```cpp
basic_json & operator=(const basic_json &) noexcept( /*omitted*/ );
```

*Effect:* Copy assignment operator.

*Complexity:* Linear.

```cpp
basic_json & operator=(basic_json &&) noexcept( /*omitted*/ );
```

*Effect:* Move assignment operator.

*Complexity:* Linear.


<a name="class-basic_json-non-modifying-operators"></a>
#### Non-Modifying Operators

```cpp
bool operator==(const_reference) noexcept;
```

*Effect:* Compares two JSON values for equality according to the following rules:
- Two JSON values are equal if (1) they are from the same type and (2)
  their stored values are equal.
- Integer numbers are automatically converted before comparison.
- Floating-point numbers are automatically converted before
  comparison. Floating-point numbers are compared indirectly: two
  floating-point numbers `f1` and `f2` are considered equal if neither
  `f1 > f2` nor `f2 > f1` holds.
- Two JSON null values are equal.
- The types `ObjectType` and `ArrayType` must provide their own
  comparison `operator==` which will be used to compare stored values.

*Throws:* Nothing.

*Complexity:* Type dependent:

  Value type                 | Complexity
  -------------------------- | ----------
  `value_t::null`            | constant
  `value_t::object`          | depending on comparison of equality of `ObjectType`
  `value_t::array`           | depending on comparison of equality of `ArrayType`
  `value_t::string`          | depending on comparison of equality of `StringType`
  `value_t::boolean`         | constant
  `value_t::number_integer`  | constant
  `value_t::number_unsigned` | constant
  `value_t::number_float`    | constant
  `value_t::discarded`       | constant


```cpp
template <typename ScalarType, /*SFINAE omitted*/ >
friend bool operator==(const_reference, const Scalartype) noexcept;

template <typename ScalarType, /*SFINAE omitted*/ >
friend bool operator==(const Scalartype, const_reference) noexcept;
```

*Effect:* Compare scalar types to the JSON value by converting the scalar
value into a JSON value, followed by a comparison of equality.

*Remarks:* Same properties as `bool operator==(const_reference) noexcept;`

```cpp
bool operator!=(const_reference) noexcept;
```

*Effect:* Compares two JSON values for inequality by calculating the equivalent
of `not (lhs == rhs)`.

*Throws:* Nothing.

*Complexity:* Equal to `operator==(const_reference)`.

```cpp
template <typename ScalarType, /*SFINAE omitted*/ >
friend bool operator!=(const_reference, const Scalartype) noexcept;

template <typename ScalarType, /*SFINAE omitted*/ >
friend bool operator!=(const Scalartype, const_reference) noexcept;
```

*Effect:* Compare scalar types to the JSON value by converting the scalar
value into a JSON value, followed by a comparison of inequality.

*Remarks:* Same properties as `bool operator!=(const_reference) noexcept;`


<a name="class-basic_json-query-member-functions"></a>
#### Query Member Functions

```cpp
constexpr value_t type() const noexcept;
```

*Effect:* Returns the type of the JSON value as a value of the `value_t` enumeration.

*Throws:* Nothing.

*Complexity:* Constant.

```cpp
constexpr operator value_t () const noexcept;
```

*Effect:* Implicit type cast of the type of the JSON value to a value of the `value_t` enumeration.

*Throws:* Nothing.

*Complexity:* Constant.

```cpp
constexpr bool is_primitive() const noexcept;
```

*Effect:* This function returns `true` if the JSON type is primitive, which is one of the following types:
  - `value_t::string`
  - `value_t::number_integral_signed`
  - `value_t::number_integral_unsigned`
  - `value_t::number_floating_point`
  - `value_t::boolean`
  - `value_t::null`

If not one of those types, it returns `false`.

*Throws:* Nothing.

*Complexity:* Constant.

```cpp
constexpr bool is_structured() const noexcept;
```

*Effect:* This function returns `true` if the JSON type is structured, which is one of the following types:
  - `value_t::object`
  - `value_t::array`

If not one of those types, it returns `false`.

*Throws:* Nothing.

*Complexity:* Constant.

```cpp
constexpr bool is_number() const noexcept;
```

*Effect:* This function returns `true` if the JSON type is a number, which is one of the following types:
  - `value_t::number_integral_signed`
  - `value_t::number_integral_unsigned`
  - `value_t::number_floating_point`

If not one of those types, it returns `false`.

*Throws:* Nothing.

*Complexity:* Constant.

```cpp
constexpr bool is_null()              const noexcept;
constexpr bool is_boolean()           const noexcept;
constexpr bool is_string()            const noexcept;
constexpr bool is_integral_signed()   const noexcept;
constexpr bool is_integral_unsigned() const noexcept;
constexpr bool is_floating_point()    const noexcept;
constexpr bool is_object()            const noexcept;
constexpr bool is_array()             const noexcept;
constexpr bool is_discarded()         const noexcept;
```

*Effect:* This function returns `true` if the JSON type is of the specific one, `false` otherwise.

*Throws:* Nothing.

*Complexity:* Constant.

```cpp
bool empty() const noexcept;
```

*Effect:* Returns `true` if a JSON value has no elements, `false` otherwise.
The return value depends on the different types and is defined as follows:

  Value type                           | return value
  ------------------------------------ | -------------
  `value_t::null`                      | `true`
  `value_t::boolean`                   | `false`
  `value_t::string`                    | `false`
  `value_t::number_integral_signed`    | `false`
  `value_t::number_integral_unsigned`  | `false`
  `value_t::number_floating_point`     | `false`
  `value_t::object`                    | result of function `object_type::empty()`
  `value_t::array`                     | result of function `array_type::empty()`
  `value_t::discarded`                 | `false`

*Remarks:* This function does not return whether a string stored as JSON value
is empty - it returns whether the JSON container itself is empty which is
`false` in the case of a string.

*Remarks:* This function helps `basic_json` satisfying the [Container](http://en.cppreference.com/w/cpp/concept/Container)
requirements:
  - The complexity is constant.
  - Has the semantics of `begin() == end()`.

*Throws:* Nothing.

*Complexity:* Constant, as long as `array_type` and `object_type` satisfy the Container concept;
that is, their `empty()` functions have constant complexity.


```cpp
size_type size() const noexcept;
```
*Effect:* Returns the number of elements in a JSON value.
The return value depends on the different types and is defined as follows:

  Value type                           | return value
  ------------------------------------ | -------------
  `value_t::null`                      | `0`
  `value_t::boolean`                   | `1`
  `value_t::string`                    | `1`
  `value_t::number_integral_signed`    | `1`
  `value_t::number_integral_unsigned`  | `1`
  `value_t::number_floating_point`     | `1`
  `value_t::object`                    | result of function `object_type::size()`
  `value_t::array`                     | result of function `array_type::size()`
  `value_t::discarded`                 | `0`

*Remarks:* This function does not return the length of a string stored as JSON
value - it returns the number of elements in the JSON value which is `1` in the case of a string.

*Remarks:* This function helps `basic_json` satisfying the [Container](http://en.cppreference.com/w/cpp/concept/Container)
requirements:
  - The complexity is constant.
  - Has the semantics of `std::distance(begin(), end())`.

*Throws:* Nothing.

*Complexity:* Constant, as long as `array_type` and `object_type` satisfy the Container concept;
that is, their size() functions have constant complexity.


```cpp
size_type max_size() const noexcept;
```

*Effect:* Returns the maximum number of elements a JSON value is able to hold due to
system or library implementation limitations.
The return value depends on the different types and is defined as follows:

  Value type                           | return value
  ------------------------------------ | -------------
  `value_t::null`                      | `0` (same as `size()`)
  `value_t::boolean`                   | `1` (same as `size()`)
  `value_t::string`                    | `1` (same as `size()`)
  `value_t::number_integral_signed`    | `1` (same as `size()`)
  `value_t::number_integral_unsigned`  | `1` (same as `size()`)
  `value_t::number_floating_point`     | `1` (same as `size()`)
  `value_t::object`                    | result of function `object_type::max_size()`
  `value_t::array`                     | result of function `array_type::max_size()`
  `value_t::discarded`                 | `0` (same as `size()`)

*Remarks:* This function helps `basic_json` satisfying the [Container](http://en.cppreference.com/w/cpp/concept/Container)
requirements:
  - The complexity is constant.
  - Has the semantics of returning `b.size()` where `b` is the largest possible JSON value.

*Throws:* Nothing.

*Complexity:* Constant, as long as `array_type` and `object_type` satisfy the Container concept;
that is, their `max_size()` functions have constant complexity.

```cpp
size_type count(typename object_type::key_type key) const;
```

*Effect:* Returns the number of occurrences of a key in a JSON object.
This method always returns `0` when executed on a JSON type that is not of
type `value_t::object`.

*Remarks:* If `ObjectType` is the default `std::map` type, the return value will
always be `0` (`key` was not found) or `1` (`key` was found).

*Throws:* Nothing.

*Complexity:* Depends on the underlying type of `ObjectType` for lookups.
`basic_json` adds constant complexity.

```cpp
iterator       find(typename object_type::key_type key);
const_iterator find(typename object_type::key_type key) const;
```

*Effect:* Finds an element in a JSON object with key equivalent to `key`.
If the element is not found or the JSON value is not an object, the result
of `end()` is returned.
This method always returns the result of `end()` when executed on a JSON type
that is not of type `value_t::object`.

*Remarks:* There is a const and a non-const overload of this function.

*Throws:* Nothing.

*Complexity:* Depends on the underlying type of `ObjectType` for lookups.
`basic_json` adds constant complexity.


<a name="class-basic_json-element-access"></a>
#### Element Access

```cpp
const_reference operator[](size_type) const;
      reference operator[](size_type);
```

*Effect:* Returns a reference to an element (a JSON value itself) of a JSON value of
type `value_t::array` at the specified index. Overloads for const and non-const versions.

The non-const overload will, in case the specified index is greater or equal to `size()`,
extend the array to a size capable of holding the specified index. All other values
in the new array between the previous size and the new size will be filled with JSON values
of type `value_t::null`. The reference to the newly created JSON value which is returned is
a default constructed JSON value.

The non-const overload also will convert the JSON value into a type `value_t::array`, if
it was of type `value_t::null`.

*Precondition:* The JSON value must be of type `value_t::array` or `value_t::null`.

*Postcondition:* The non-const overload invalidates all iterators, pointers and references.

*Remarks:*
- No synchronization.
- const overload: No index checks are being performed. Access to indices other than `0..size()-1`
  are undefined behaviour.

*Throws:* `std::domain_error` if the type of the JSON value is diffrent than `value_t::array`
or `value_t::null`.

*Complexity:*
- const overload: The complexity of a random element access of the underlying data structure
defined by `ArrayType`. Constant overhead by `basic_json`.
- non-const overload: If the specified index is in the range `0..size()-1`, it is the same
complexity as the const overload. If not, it is the same complexity as for the underlying
data structure defined by `ArrayType` to create a container of sufficient size.
Constant overhead by `basic_json`.

```cpp
const_reference operator[](const typename object_type::key_type &) const;
      reference operator[](const typename object_type::key_type &);
```

*Effects:* Returns a reference to an element (a JSON value itself) of a JSON value of
type `value_t::object` for the specified key. Overloads for const and non-const versions.

The non-const overload will, in case the specified key is not present in the JSON value,
default construct a JSON value for the key and return the reference to it.

The non-const overload also will convert the JSON value into a type `value_t::object`, if
it was of type `value_t::null`.

*Precondition:* The JSON value must be of type `value_t::object` or `value_t::null`.

*Postcondition:* The non-const overload invalidates all iterators, pointers and references.

*Remarks:*
- No synchronization.
- const overload: No checks are being performed. Specifying a non-existent key is undefined
  behaviour.

*Throws:*
- const overlaod: `std::domain_error` if the type of the JSON value is diffrent than `value_t::object`.
- non-const overload:  `std::domain_error` if the type of the JSON value is diffrent than
  `value_t::object` or `value_t::null`.

*Complexity:*
- const overload: The complexity of an element access of the underlying data structure
  defined by `ObjectType`. Constant overhead by `basic_json`.
- non-const overload: If the specified key exists, it is the same complexity as the const overload.
  If not, it is the same complexity as for the underlying data structure defined by `ObjectType` to
  create a container and the element for the corresponding key. Constant overhead by `basic_json`.

```cpp
template <typename T> const_reference operator[](T *) const;
template <typename T>       reference operator[](T *);

template <typename T, std::size_t N> const_reference operator[](T * (&key)[N]) const;
template <typename T, std::size_t N>       reference operator[](T * (&key)[N]);
```

**TODO**

```cpp
const_reference operator[](const json_pointer &) const;
      reference operator[](const json_pointer &);
```

*Effect:* Returns a reference to a JSON value specified by the JSON pointer.
Overloads for const and non-const versions.

The non-const overload will insert the requested JSON value, in particular:
- If the JSON pointer points to an object key which does not exist, the JSON value
  is default constructed.
  If the JSON value on which the member function is called upon is of type `value_t::null`,
  it will be converted to `value_t::object`.
- If the JSON pointer points to array index which does not exist, the array will
  be expanded to a be able to hold the specified index. All newly created JSON values
  in the array are constructed and of type `value_t::null`. The JSON value at the
  specified index is default constructed.
  If the JSON value on which the member function is called upon is of type `value_t::null`,
  it will be converted to `value_t::array`.
- The JSON pointer special value `-` is treated as array index past the last existing,
  i.e. appending to the existing array.
  If the JSON value on which the member function is called upon is of type `value_t::null`,
  it will be converted to `value_t::array`.

*Postcondition:* The non-const overload invalidates all iterators, pointers and references.

*Remarks:*
- No bounds checking is performed.
- No synchronization.

*Throws:*
- `std::out_of_range` is thrown if the JSON pointer can not be resolved.
- `std::domain_error` is thrown in case of an array index of `0`.
- `std::domain_error` is thrown in the const overload with a JSON pointer special
  value of `-`.
- `std::invalid_argument` is thrown in case of a non-numerical array index.

*Complexity:*
- const overload: The complexity of an element access of the underlying data structure
  defined by `ObjectType` or `ArrayType`, constant for primitive types.
  Constant overhead by `basic_json`.
- non-const overload: If the specified key exists, it is the same complexity as the const overload.
  If not, it is the same complexity as for the underlying data structure defined by `ObjectType`
  or `ArrayType` to create a container and the element for the corresponding key or index.
  Constant for primitive types. Constant overhead by `basic_json`.

```cpp
const_reference at(size_type) const;
      reference at(size_type);
```

*Effect:* Returns a reference to the element specified by an array index.
Overloads for const and non-const versions.

*Precondition:* The JSON value which this member function is called upon must be of
type `value_t::array`.

*Remarks:*
- Performs bound checks.
- No synchronization.

*Throws:*
- `std::domain_error` if the JSON value is not of type `value_t::array`.
- `std::out_of_range` if the specified index is not within the range `0..size()-1`.

*Complexity:* The same as for a random access of the underlying data structure defined
by `ArrayType`. Constant overhead by `basic_json`.

```cpp
const_reference at(const typename object_type::key_type &) const;
      reference at(const typename object_type::key_type &);
```

*Effect:* Returns a reference to the element for the specified key.
Overloads for const and non-const versions.

*Precondition:* The JSON value which this member function is called upon must be of
type `value_t::object`.

*Remarks:*
- Performs bound checks.
- No synchronization.

*Throws:*
- `std::domain_error` if the JSON value is not of type `value_t::object`.
- `std::out_of_range` if the specified key does not exist, equivalent to `find(key)==end()`.

*Complexity:* The same as for finding an element with a specified key of the underlying data
structure defined by `ObjectType`. Constant overhead by `basic_json`.

```cpp
const_reference at(const json_pointer &) const;
      reference at(const json_pointer &);
```

*Effect:* Returns a reference to the element specified by the JSON pointer.
Overloads for const and non-const versions.

*Remarks:*
- Performs bound checks.
- No synchronization.

*Throws:*
- `std::out_of_range` if the JSON pointer can not be resolved.
- `std::domain_error` if an array index begins with `0`.
- `std::domain_error` if the JSON pointer holds the special value `-`.
- `std::invalid_argument` is thrown in case of a non-numerical array index.

*Complexity:* The complexity of an element access of the underlying data structure
defined by `ObjectType` or `ArrayType`, constant for primitive types.
Constant overhead by `basic_json`.

```cpp
template <class ValueType, /*SFINAE omitted*/ >
ValueType value(const typename object_type::key_type &, ValueType default_value) const;

string_type value(const typename object_type::key_type &, const char * default_value) const;

template <class ValueType, /*SFINAE omitted*/ >
ValueType value(const json_pointer &, ValueType default_value) const;

string_type value(const json_pointer &, const char * default_value) const;
```

*Effect:* Returns a copy of the data of an JSON value specified by a key or a default value,
if the key does not exist within the called JSON value of type `value_t::object`.

The JSON value to be found must be convertible into `ValueType`.

There is an overload specialized for `const char *` returning a `StringType`.

There is also an overload pair which takes a JSON path instead of an object key. If the
JSON pointer can not be resolved, the default value is returned.

*Remarks:* No synchronization.

*Throws:* `std::domain_error` if the JSON value which the member function is called upon is not
  of type `value_t::object`.

*Complexity:* The same complexity af for the underlying data structure defined by `ObjectType`
to find an element. Constant overhead by `basic_json`.


<a name="class-basic_json-container-access"></a>
#### Container Access

```cpp
const_iterator begin()  const noexcept;
const_iterator cbegin() const noexcept;
      iterator begin()        noexcept;
```

*Effect:* Returns an iterator to the first element, overloads for const and non-const versions.

*Throws:* Nothing.

*Complexity:* Constant.

```cpp
const_iterator end()  const noexcept;
const_iterator cend() const noexcept;
      iterator end()        noexcept;
```

*Effect:* Returns an iterator to one past the last element, overloads for const and
non-const versions.

*Throws:* Nothing.

*Complexity:* Constant.

```cpp
reverse_const_iterator rbegin() const noexcept;
reverse_iterator       rbegin()       noexcept;
```

*Effect:* Returns an iterator to the reverse-beginning, overloads for const and non-const versions.

*Throws:* Nothing.

*Complexity:* Constant.

```cpp
reverse_const_iterator rend() const noexcept;
reverse_iterator       rend()       noexcept;
```

*Effect:* Returns an iterator to the reverse-end, overloads for const and non-const versions.

*Throws:* Nothing.

*Complexity:* Constant.

```cpp
const_reference front() const noexcept;
      reference front()       noexcept;
```

*Effect:* Returns a reference to the first element of the JSON value, overloads for
const and non-const references. Depending on the type of the JSON value:
- `value_t::object` and `value_t::array`: the function returns a reference to the first
   element in the underlying container.
- primitive type (except `value_t::null`): the function returns a reference to the value.

*Throws:* `std::domain_error` if the function was called on a JSON value with a type
of `value_t::null`.

*Remarks:* Calling this function on a structured JSON value (types `value_t::object` and
`value_t::array`) with empty underlying containers, the behaviour is *undefined*.

*Complexity:*
- For primitive JSON value types (except type `value_t::null`): Constant.
- For JSON value of type `value_t::object`, the complexity to access the first element of
  the underlying container defined by `ObjectType`.
- For JSON value of type `value_t::array`, the complexity to access the first element of
  the underlying container defined by `ArrayType`.

```cpp
const_reference back() const noexcept;
      reference back()       noexcept;
```

*Effect:* Returns a reference to the last element of the JSON value, overloads for
const and non-const references. Depending on the type of the JSON value:
- `value_t::object` and `value_t::array`: the function returns a reference to the last
  element in the underlying container.
- primitive type (except `value_t::null`): the function returns a referene to the value.

*Throws:* `std::domain_error` if the function was called on a JSON vaule with a type
of `value_t::null`.

*Remarks:* Calling this function on a structured JSON value (types `value_t::object` and
`value_t::array`) with empty underlying containers, the behaviour is *undefined*.

*Complexity:*
- For primitive JSON value types (except type `value_t::null`): Constant.
- For JSON value of type `value_t::object`, the complexity to access the last element of
  the underlying container defined by `ObjectType`.
- For JSON value of type `value_t::array`, the complexity to access the last element of
  the underlying container defined by `ArrayType`.


<a name="class-basic_json-container-operations"></a>
#### Container Operations

```cpp
void clear() noexcept; // clears container
```

*Requires:* Nothing.

*Effect:* Clears the JSON value.

*Postcondition:* The contained value is destructed if necessary. All sub-values (in case
of types `value_t::object` and `value_t::array`) are destructed. The JSON value is of
type `value_t::null`. The JSON value is in the same state as if constructed
with `basic_json(std::nullptr)` or the default constructor.

*Remarks:* No synchronization.

*Throws:* Nothing.

```cpp
void swap(reference) noexcept;
```

*Effect:* Exchanges the contents of the JSON value with those of the specified one.
Does not invoke any move, copy, or swap operations on individual elements.
All iterators and references remain valid. The past-the-end iterator is invalidated.

*Remarks:* No synchronization.

*Requires:* Nothing.

*Throws:* Nothing.

*Comlexity:* Constant.

```cpp
void swap(array_type &);
```

*Effect:* Exchanges the contents of a JSON array with those of the specified one.
Does not invoke any move, copy, or swap operations on individual elements.
All iterators and references remain valid. The past-the-end iterator is invalidated.

*Remarks:* No synchronization.

*Throws:* `std::domain_error` if JSON value is not of type `value_t::array`.
Example: `"cannot use swap() with string"`.

*Comlexity:* Constant.

```cpp
void swap(object_type &);
```

*Effect:* Exchanges the contents of a JSON object with those of the specified one.
Does not invoke any move, copy, or swap operations on individual elements.
All iterators and references remain valid. The past-the-end iterator is invalidated.

*Remarks:* No synchronization.

*Throws:* `std::domain_error` if JSON value is not of type `value_t::object`.
Example: `"cannot use swap() with string"`.

*Comlexity:* Constant.

```cpp
void swap(string_type &);
```

*Effect:* Exchanges the contents of a JSON string with those of the specified one.
Does not invoke any move, copy, or swap operations on individual elements.
All iterators and references remain valid. The past-the-end iterator is invalidated.

*Remarks:* No synchronization.

*Throws:* `std::domain_error` when JSON value is not of type `value_t::string`
Example: `"cannot use swap() with boolean"`.

*Comlexity:* Constant.

```cpp
void push_back(basic_json &&);
void push_back(const basic_json &);
void push_back(const typename object_type::value_type &);
```

*Requires:* The JSON value which the data is appended to must be of type
`value_t::array` or `value_t::null`.

*Effect:* Appends data to the JSON value. If the type was `value_t::null`, an
empty JSON value of type `value_t::array` is created and the specified data appended.
The appended data is stored in form of a JSON value and is owned by the JSON value
it was appended to.

*Remarks:* No synchronization.

*Throws:*
  - If the JSON value type was wrong: `std::domain_error`.
  - The underlying data structure which holds the values also may throw an exception.
    Which one depends on the underlying data structure, which is defined by the template
    parameter `ArrayType`.

*Complexity:* The operation relies on its underlying type for handling
arrays, which is defined by the template parameter `ArrayType`. The operation
`basic_json::push_back` does not introduce a higher complexity than amortized *O(1)*.

```cpp
void push_back(std::initializer_list<basic_json>);
```

*Effect:* This function allows to use `push_back` with an initializer list. In case
  1. the current JSON value is of type `value_t::object`, and
  2. the initializer list contains only two elements, and
  3. the first element of the initializer list is a string,
the initializer list is converted into an object element (JSON value of type `value_t::object`)
and added using `void push_back(const typename object_type::value_type&)`. Otherwise,
the initializer list is converted to a JSON value and added using `void push_back(basic_json&&)`.

*Throws:*
  - If the JSON value type was wrong: `std::domain_error`.
  - The underlying data structure which holds the values also may throw an exception.
    Which one depends on the underlying data structure, which is defined by the template
    parameter `ArrayType`.

*Complexity:* Linear in the size of the initializer list.

*Remarks:* This function is required to resolve an ambiguous overload error,
because pairs like `{"key", "value"}` can be both interpreted as `object_type::value_type`
or `std::initializer_list<basic_json>`.

```cpp
reference operator+=(basic_json &&);
reference operator+=(const basic_json &);
reference operator+=(const typename object_type::value_type &);
```

*Remarks:* The same requirements, effects, exceptions and complexity as
`void push_back(basic_json &&);`. Except, it returns the added JSON value
as reference.

```cpp
reference operator+=(std::initializer_list<basic_json>);
```

*Remarks:* The same requirements, effects, exceptions and complexity as
`void push_back(std::initializer_list<basic_json>);`. Except, it returns the
added JSON value as reference.

```cpp
template<class... Args> reference emplace_back(Args && ...);
```

*Effect:* Appends a new JSON value from the passed parameters to the JSON value.
If the function is called on a JSON value of type `value_t::null`, an empty
JSON value of type `value_t::array` is created before appending the newly created
value from the arguments. A reference to the newly created object is returned.

*Requires:* A `basic_json` object must be construtible from the template argument types.

*Throws:* `std::domain_error` when called on a JSON value of type other than
`value_t::array` or `value_t::null`.

*Remarks:* No synchronization.

*Complexity:* Amortized constant plus the complexity of the append operation of
the underlying data structure, which is defined by the template parameter `ArrayType`.

```cpp
template<class... Args> std::pair<iterator, bool> emplace(Args && ...);
```

*Effect:* Inserts a new JSON value into a JSON value of type `value_t::object`,
constructed in-place with the given arguments, if there is no element with the key in the
container. If the function is called on a JSON value of type `value_t::null`, an empty
JSON value of type `value_t::object` is created before appending the newly created value.

If a JSON value with the same key already exists, its content will be replaced with
the newly created JSON value from the specified parameters. If an insertion of the newly
created JSON value is not possible, the underlying container remains unchanged.

The function returns a pair, containing:
  - *first:* an iterator to the inserted JSON value or the already existing JSON value if
    no insertion happened because of an existing key. If the insertion failed, the result
    of `end()` will be returned.
  - *second:* a `bool` denoting wheather the insertion took place.

*Requires:* A `basic_json` object must be construtible from the template argument types.

*Throws:* `std::domain_error` when called on a JSON value of type other than
`value_t::object` or `value_t::null`.

*Remarks:* No synchronization.

*Complexity:* Amortized constant plus the complexity of the insert operation of
the underlying data structure, which is defined by the template parameter `ObjectType`.

```cpp
iterator insert(const_iterator, const basic_json &);
iterator insert(const_iterator, basic_json &&);
```

*Effect:* Inserts a JSON value before the position specified by the iterator.
The function returns an iterator pointing to the newly inserted JSON value.

*Throws:*
- `std::domain_error` if the JSON value which the member function is called upon,
  is not of type `value_t::array`.
- `std::invalid_argument` if the specified iterator is not an iterator of `*this`.

*Postcondition:* If the insertion was not possible, the JSON value which the member
function was called upon, remains in the same state as before the function call.

*Complexity:* Constant plus the complexity of the insert operation of the underlying
data structure, defined by the type `ArrayType`.

```cpp
iterator insert(const_iterator, size_type, const basic_json &);
```

*Effect:* Inserts a number of copies of the specified JSON value before the given position,
specified by the iterator. The function returns an iterator pointing to the first newly
inserted JSON value. If the number of elements to insert is `0`, the function inserts no
elements and returns the position specified by the iterator.

*Throws:*
- `std::domain_error` if the JSON value which the member function is called upon,
  is not of type `value_t::array`.
- `std::invalid_argument` if the specified iterator is not an iterator of `*this`.

*Postcondition:* If the insertion was not possible, the JSON value which the member
function was called upon, remains in the same state as before the function call.
Either all new JSON values can be inserted or none.

*Complexity:* Linear in number of copies to insert, plus the complexity of the insert
operation of the underlying data structure, defined by the type `ArrayType`.

```cpp
iterator insert(const_iterator pos, const_iterator first, const_iterator last);
```

*Effect:* Inserts JSON values from the specified range `[first, last)` before the
position given by the iterator. The function returns an iterator pointing to the first newly
inserted JSON value. If the range is empty (`first == last`), the function inserts no
elements and returns the position specified by the iterator.

*Remarks:* Insertion into itself is forbidden.

*Remarks:* No synchronization.

*Throws:*
- `std::domain_error` if the JSON value which the member function is called upon,
  is not of type `value_t::array`.
- `std::invalid_argument` if the specified iterator is not an iterator of `*this`.
- `std::invalid_argument` if the specified iterators `first` and `last` do not belong
  to the same JSON value.
- `std::invalid_argument` if the specified iterators `first` or `last` are iterators
  of the container for which `insert` is being called.

*Precondition:* The JSON value which the elements are inserted into must be of
type `value_t::array`.

*Postcondition:* If the insertion was not possible, the JSON value which the member
function was called upon, remains in the same state as before the function call.
Either all new JSON values can be inserted or none.

*Complexity:* Linear in number of copies to insert, i.e. `std::distance(first, last)`,
plus the complexity of the insert operation of the underlying data structure, defined by
the type `ArrayType`.

```cpp
iterator insert(const_iterator, std::initializer_list<basic_json>);
```

*Effect:* Inserts JSON values from the initializer list before the position
specified by the iterator. The function returns an iterator pointing to the first
newly inserted JSON value. If the initializer list is empty, the function inserts
no elements and returns the specified iterator.

*Remarks:* No synchronization.

*Precondition:* The JSON value which the elements are inserted into must be of
type `value_t::array`.

*Postcondition:* If the insertion was not possible, the JSON value which the member
function was called upon, remains in the same state as before the function call.
Either all new JSON values can be inserted or none.

*Throws:*
- `std::domain_error` if the JSON value which the member function is called upon,
  is not of type `value_t::array`.
- `std::invalid_argument` if the specified iterator is not an iterator of `*this`.

*Complexity:* Linear in size of the initializer list, plus the complexity of the
insert operation of the underlying data structure, defined by the type `ArrayType`.

```cpp
size_type erase(const typename object_type::key_type &);
```

*Effect:* Removes elements from a JSON value of type `value_t::object`.
The function returns the number of removed elements, which is zero or a positive
number.

*Postcondition:* References and iterators to the erased elements are invalidated.
Other references and iterators are not affected.

*Throws:* `std::domain_error` if the type of the JSON value was other than `value_t::object`.

*Remarks:* No synchronization.

*Complexity:* The same complexity to find and erase all occurrences of the specified key
of the underlying data structure.

```cpp
void erase(const size_type);
```

*Effect:* Removes an element at the specified index (in the range `[0, size()-1]`)
from a JSON value of type `value_t::array`.

*Postcondition:* References and iterators are invalidated.

*Throws:*
- `std::domain_error` if the JSON value which the member function is called upon,
  is not of type `value_t::array`.
- `std::out_of_range` if the specified index is not in the range `[0, size()-1]`.

*Remarks:* No synchronization.

*Complexity:* The same complexity to erase the element at the specified index in
the underlying data structure.

```cpp
template<class IteratorType, /*SFINAE omitted*/ > IteratorType erase(IteratorType pos);
```

*Effects:* Erases the element from the JSON value at the position specified by the iterator.
The member function returns an iterator pointing to the element after the erased element.
Depending on the type of the JSON value:
- `value_t::object` / `value_t::array`: the element at the specified position will
  be erased.
- Primitive types (except `value_t::null`): the resulting JSON value will be of
  type `value_t::null`, its content will be erased.

*Precondition:* The iterator type `IteratorType` must be of `iterator` or `const_iterator`.

*Postcondition:* References and iterators are invalidated.

*Throws:*
- `std::domain_error` if the JSON value which the member function is called upon,
  is of type `value_t::null`.
- `std::invalid_argument` if the specified iterator does not belong to the current
  JSON value.
- `std::out_of_range` if the specified position is not in the range `[begin(), end())`.

*Remarks:* No synchronization.

*Complexity:* Depends on the type of the JSON value:
- `value_t::object`: The complexity of erasure of an element of the underlying
  data structure, defined by `ObjectType`. Constant overhead by `basic_json`.
- `value_t::array`: The complexity of erasure of an element of the underlying
  data structure, defined by `ArrayType`. Constant overhead by `basic_json`.
- `value_t::string`: Complexity of the destruction of the `StringType`. Constant
  overhead by `basic_json`.
- others: Constant.

```cpp
template<class IteratorType, /*SFINAE omitted*/ > IteratorType erase(IteratorType first, IteratorType last);
```

*Effect:* Erases all elments from the JSON value specified by the range `[first, last)`.
The function returns an iterator pointing to the element after the last erased. Erasing an
empty range is a no-op. Depending on the type of the JSON value:
- `value_t::object` / `value_t::array`: The elements in the specified range are
  erased from the JSON value.
- Primitive types (except `value_t::null`): the resulting JSON value will be of
  type `value_t::null`, its content will be erased.

*Precondition:* The iterator type `IteratorType` must be of `iterator` or `const_iterator`.

*Postcondition:* References and iterators are invalidated.

*Throws:*
- `std::domain_error` if the JSON value which the member function is called upon,
  is of type `value_t::null`.
- `std::invalid_argument` if of of the specified iterators does not belong to the current
  JSON value.
- `std::out_of_range` if the specified position is not in the range `[begin(), end())`.

*Remarks:* No synchronization.

*Complexity:* Depends on the type of the JSON value:
- `value_t::object`: The complexity of erasure of the range of elements of the underlying
  data structure, defined by `ObjectType`. Constant overhead by `basic_json`.
- `value_t::array`: The complexity of erasure of the range of elements of the underlying
  data structure, defined by `ArrayType`. Constant overhead by `basic_json`.
- `value_t::string`: Complexity of the destruction of the `StringType`. Constant
  overhead by `basic_json`.
- others: Constant.


<a name="class-basic_json-serialization"></a>
#### Serialization / Deserialization

```cpp
// content to string
string_type str(int indent) const;

// output stream
friend std::ostream & operator<<(std::ostream & const basic_json &);

// input stream
friend std::isteram & operator>>(std::istream & basic_json &);
```


<a name="class-json_pointer"></a>
### Nested Class `basic_json::json_pointer`

```cpp
class json_pointer
{
    // allow basic_json to access private members
    friend class basic_json;

public:
    explicit json_pointer(const std::string & s = "");

    // return a string representation of the JSON pointer
    std::string to_string() const noexcept;
    operator std::string() const;

private:
    // remove and return last reference pointer
    std::string pop_back();

    // return whether pointer points to the root document
    bool is_root() const;

    json_pointer top() const;

    // create and return a reference to the pointed to value
    // complexity: Linear in the number of reference tokens.
    reference get_and_create(reference j) const;

    // return a reference to the pointed to value
    // complexity: Linear in the length of the JSON pointer.
    reference get_unchecked(pointer ptr) const;
    reference get_checked(pointer ptr) const;

    // return a const reference to the pointed to value
    const_reference get_unchecked(const_pointer ptr) const;
    const_reference get_checked(const_pointer ptr) const;

    // split the string input to reference tokens
    static std::vector<std::string> split(const std::string & reference_string);

private:
    // replace all occurrences of a substring by another string
    static void replace_substring(
        std::string & s,
        const std::string & f,
        const std::string & t);

    // escape tilde and slash
    static std::string escape(std::string s);

    // unescape tilde and slash
    static void unescape(std::string& s);

    static void flatten(
        const std::string & reference_string,
        const basic_json & value,
        basic_json & result);

    static basic_json unflatten(const basic_json & value);

private:
    friend bool operator==(
        json_pointer const & lhs,
        json_pointer const & rhs) noexcept;

    friend bool operator!=(
        json_pointer const & lhs,
        json_pointer const & rhs) noexcept;

private:
    // the reference tokens
    std::vector<std::string> reference_tokens {};
};
```


<a name="func-to_json"></a>
### Free Functions `to_json`

```cpp
template <typename BasicJsonType, typename T, /*SFINAE omitted*/>
void to_json(BasicJsonType &, T) noexcept;                                               // (1)

template <typename BasicJsonType, typename UnscopedEnumType, /*SFINAE omitted*/>
void to_json(BasicJsonType &, UnscopedEnumType) noexcept;                                // (2)

template <typename BasicJsonType, typename CompatibleStringType, /*SFINAE omitted*/>
void to_json(BasicJsonType &, const CompatibleStringType &);                             // (3)

template <typename BasicJsonType, typename CompatibleArrayType, /*SFINAE omitted*/>
void to_json(BasicJsonType &, const CompatibleArrayType &);                              // (4)

template <typename BasicJsonType, typename CompatibleObjectType, /*SFINAE omitted*/>
void to_json(BasicJsonType &, const CompatibleObjectType &);                             // (5)
```

*Effect:* Converts the specified argument to a JSON value of type `BasicJsonType`.

*Precondition:* Specified value types must be convertible to a JSON value. Depending on the
overload, the specified value must implicitly convertible, according to the following table:

Overload | Implicitly convertible into             | Resulting JSON value type
-------- | --------------------------------------- | -----------------------------------
(1)      | `BasicJsonType::boolean_type`           | `value_t::boolean`
         | `BasicJsonType::integral_signed_type`   | `value_t::number_integral_signed`
         | `BasicJsonType::integral_unsigned_type` | `value_t::number_integral_unsigned`
         | `BasicJsonType::floating_point_type`    | `value_t::number_floating_point`
         |                                         | `value_t::null`
(2)      | `BasicJsonType::integral_signed_type`   | `value_t::number_integral_signed`
(3)      | `BasicJsonType::string_type`            | `value_t::string`
(4)      | `BasicJsonType::array_type`             | `value_t::array`
(5)      | `BasicJsonType::object_type`            | `value_t::object`

*Postcondition:*
- The output parameter of type `BasicJsonType` is a valid JSON value of appropriate
  type `value_t` (see table *Precondition*).
- If the value is a floating point number and either `inf` or `NAN`, the resulting
  JSON value is of type `value_t::null`.

*Remarks:* This function is a customization point. Custom implementations of `to_json` may
be provided by the user to map custom data to JSON values.

*Throws:*
- Overloads (1) and (2): Nothing.
- Overloads (3), (4), (5): `std::bad_alloc` if the memory allocation to hold the copy of the specified
  data fails.

*Complexity:*
- Overloads (1) and (2): Constant.
- Overload (3): The same complexity to copy the the string, defined by the type `StringType`.
- Overloads (4), (5): The same complexity as the underlying data structures, defined by
  either `ArrayType` or `ObjectType` to create the container and linear in number of elements to copy.


<a name="func-from_json"></a>
### Free Functions `from_json`

```cpp
template <typename BasicJsonType>
void from_json(const BasicJsonType &, typename BasicJsonType::boolean_type &);           // (1)

template <typename BasicJsonType>
void from_json(const BasicJsonType &, typename BasicJsonType::integral_signed_type &);   // (2)

template <typename BasicJsonType>
void from_json(const BasicJsonType &, typename BasicJsonType::integral_unsigned_type &); // (3)

template <typename BasicJsonType>
void from_json(const BasicJsonType &, typename BasicJsonType::floating_point_type &);    // (4)

template <typename BasicJsonType, typename UnscopedEnumType, /*SFNINAE omitted*/>
void from_json(const BasicJsonType &, UnscopedEnumType &);                               // (5)

template <typename BasicJsonType>
void from_json(const BasicJsonType &, typename BasicJsonType::string_type &);            // (6)

template <typename BasicJsonType>
void from_json(const BasicJsonType &, typename BasicJsonType::array_type &);             // (7)

template <typename BasicJsonType, typename CompatibleArrayType, /*SFINAE omitted*/>
void from_json(const BasicJsonType &, CompatibleArrayType &);                            // (8)

template <typename BasicJsonType, typename CompatibleObjectType, /*SFINAE omitted*/>
void from_json(const BasicJsonType &, CompatibleObjectType &);                           // (9)

template <typename BasicJsonType, typename ArithmeticType, /*SFINAE omitted*/>
void from_json(const BasicJsonType &, ArithmeticType &);                                 // (10)

template <typename BasicJsonType, typename T, typename Allocator>
void from_json(const BasicJsonType &, std::forward_list<T, Allocator> &);                // (11)
```

*Effect:* Converts a JSON value of type `BasicJsonType` into a specified data type.

*Precondition:*
- all: The specified JSON value must not be of type `value_t::null`.
- (11): The specified forward list must contain the same data type as the JSON value.

*Postcondition:* The specified data type holds a copy of the contents of the given
JSON value.

*Remarks:* This function is a customization point. Custom implementations of `from_json` may
be provided by the user to map custom data to JSON values.

*Throws:*
- `std::domain_error` if the JSON value is not of the appropriate type `value_t`, according
  to the specified value type.
- (11): `std::domain_error` if the JSON value is of type `value_t::null` or the data
  type contained within the JSON value does not match the `value_type` of the specified
  forward list.

*Complexity:*
- Overloads (1), (2), (3), (4), (5), (10): Constant.
- Overload (6): The same complexity as to copy a string defined by the type `StringType`.
- Overload (7), (8): The same complexity as to copy data into the specified container,
  defined by `ArrayType`.
- Overload (9): The same complexity as to copy data into the specified container,
  defined by `ObjectType`.
- Overload (11): The same complexity as to copy data into the specified list.


<a name="func-factory"></a>
### Free Factory Functions

**TODO**


<a name="func-user-defined-literals"></a>
### User Defined Literals

**TODO**


<a name="func-swap"></a>
### Template Function Specialization `swap`

**TODO**


<a name="func-hash"></a>
### Template Specialization `hash`

**TODO**


<a name="acknowledgements"></a>
## Acknowledgements

**TODO**


<a name="references"></a>
## References

- [RFC7159]	JavaScript Object Notation, <https://tools.ietf.org/html/rfc7159>

- [RFC6901]	JavaScript Object Notation Pointer, <https://tools.ietf.org/html/rfc6901>

- [RFC6902]	JavaScript Object Notation Patch, <https://tools.ietf.org/html/rfc6902>

- [nlohmann-json]	Example Implementation, <https://github.com/nlohmann/json>

