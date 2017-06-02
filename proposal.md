| Document Number | ???                                       |
|-----------------|-------------------------------------------|
| Date            | ???                                       |
| Project         | Programming Language C++, Library Working Group |
| Reply-to        | Niels Lohmann <<mail@nlohmann.me>><br>Mario Konrad <<mario.konrad@gmx.net>> |

# JSON parser and generator library

## Introduction

This paper presents a proposal for *JavaScript Object Notation* [RFC7159] parsing and generation. It proposes a library extension.

## Motivation

Data represented in JSON format is very widely used for serialization of data. Prominent uses are the numerous frameworks used by websites for asynchronous communication, as well as configuration data.

The format itself is human readable and very lightweight.

There are numerous libraries written in C and C++, usable by C++, but none in the standard.

The goal should be a library extension that fits well into the standard library, customizable and with an user friendly interface.

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

## Scope

This extension is a pure library extension. It does not require changes to the standard components. The extension can be implemented in C++11, C++14, and C++17.

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

## Technical Specification

**TODO**

### Header `<json>` synopsis

```cpp
namespace std {
namespace experimental {

    // generic base type basic_json
    template < /*omitted*/ >
    class basic_json;

    // swap function
    template < /*omitted*/ >
    void swap(basic_json &, basic_json &);

    // hash
    template < /*omitted*/ >
    struct hash<basic_json>
    {
        std::size_t operator()(const basic_json &) const;
    };

    // creation functions: TODO
    template < /*omitted*/ > basic_json make_json( /*omitted*/ );

    // default json object class
    using json = basic_json<>;

    // user defined string literal
    json operator "" _json(const char *, std::size_t);
}
}
```

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

    template <typename Base> class json_reverse_iterator;
    class const_iterator;
    class iterator;
    class const_reverse_iterator;
    class reverse_iterator;
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

#### Construction

```cpp
basic_json() noexcept;
```

Default constructor.

*Postcondition:* The type of the JSON value is `value_t::null`.

*Throws:* Nothing.

```cpp
// explicit type construction, default content
explicit basic_json(value_t) noexcept;
```

Constructs explicitly an object representing a JSON value of the specified type.
The underlying data type (see template parameters) must be default constructible
and `noexcept`.

```cpp
// explicit null content construction
basic_json(std::nullptr) noexcept;
```

Constructs an empty JSON value of type `value_t::null`.

*Remarks:* Has the same effect as the default constructor.

*Throws:* Nothing.

```cpp
// construction from various value types
basic_json(string_type);
basic_json(boolean_type) noexcept;
basic_json(integral_signed_type) noexcept;
basic_json(integral_unsigned_type) noexcept;
basic_json(floating_point_type) noexcept;
basic_json(object_type);
basic_json(array_type);
```

Constructs a JSON value of the respective type, initialized with the
specified data.

*Postcondition:* JSON value is constructed, containing the specified data.

```cpp
// construction using initializer list
basic_json(std::initializer_list<basic_json>);
```

Constructs a JSON value containing the data from the specified initializer
list. The resulting type of the JSON value is `value_t::object`.

```cpp
// array construction with specified number of elements
basic_json(size_type count, const basic_json & value);
```

Constructs a JSON value with `count` number of elements of the specified
JSON value. The resulting type of the current object is `value_t::array`.

```cpp
// construction from iterators
template <typename InputIterator>
basic_json(InputIterator first, InputIterator last);
```

```cpp
// copy constructor, non-converting
basic_json(const basic_json &);
```

```cpp
// move constructor, non-converting
basic_json(basic_json &&);
```

#### Destruction

```cpp
~basic_json();
```

Destructs its containing data. Contained JSON values (objects, arrays) are
being destroyed as well.

#### Modifying Operators

```cpp
// copy assignment operator, non-converting
basic_json & operator=(const basic_json &);

// move assignment operator, non-converting
basic_json & operator=(basic_json &&);
```

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
  `value_t::object`                    | result of function `object_t::empty()`
  `value_t::array`                     | result of function `array_t::empty()`
  `value_t::discarded`                 | `false`

*Remarks:* This function does not return whether a string stored as JSON value
is empty - it returns whether the JSON container itself is empty which is
`false` in the case of a string.

*Remarks:* This function helps `basic_json` satisfying the [Container](http://en.cppreference.com/w/cpp/concept/Container)
requirements:
  - The complexity is constant.
  - Has the semantics of `begin() == end()`.

*Throws:* Nothing.

*Complexity:* Constant, as long as `array_t` and `object_t` satisfy the Container concept;
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
  `value_t::object`                    | result of function `object_t::size()`
  `value_t::array`                     | result of function `array_t::size()`
  `value_t::discarded`                 | `0`

*Remarks:* This function does not return the length of a string stored as JSON
value - it returns the number of elements in the JSON value which is `1` in the case of a string.

*Remarks:* This function helps `basic_json` satisfying the [Container](http://en.cppreference.com/w/cpp/concept/Container)
requirements:
  - The complexity is constant.
  - Has the semantics of `std::distance(begin(), end())`.

*Throws:* Nothing.

*Complexity:* Constant, as long as `array_t` and `object_t` satisfy the Container concept;
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
  `value_t::object`                    | result of function `object_t::max_size()`
  `value_t::array`                     | result of function `array_t::max_size()`
  `value_t::discarded`                 | `0` (same as `size()`)

*Remarks:* This function helps `basic_json` satisfying the [Container](http://en.cppreference.com/w/cpp/concept/Container)
requirements:
  - The complexity is constant.
  - Has the semantics of returning `b.size()` where `b` is the largest possible JSON value.

*Throws:* Nothing.

*Complexity:* Constant, as long as `array_t` and `object_t` satisfy the Container concept;
that is, their `max_size()` functions have constant complexity.

```cpp
size_type count(typename object_t::key_type key) const;
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
iterator       find(typename object_t::key_type key);
const_iterator find(typename object_t::key_type key) const;
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

#### Element Access

```cpp
// index operator
const_reference operator[](const json_pointer &) const;
reference operator[](const json_pointer &);

// at
const_reference at(const json_pointer &) const;
reference at(const json_pointer &)

// value access
string_type value(
    const json_pointer &,
    const string_type & default_value) const;

// value access to convertible type, SFINAE check to make sure
// the type is convertible
template <class ValueType, /* SFINAE omitted */ >
ValueType value(const json_pointer &, ValueType default_value) const;
```

#### Container Access

```cpp
// const iterator query
const_iterator begin() const noexcept;
const_iterator end() const noexcept;

const_iterator cbegin() const noexcept;
const_iterator cend() const noexcept;

// iterator query
iterator begin() noexcept;
iterator end() noexcept;

// reverse const iterator query
reverse_const_iterator rbegin() const noexcept;
reverse_const_iterator rend() const noexcept;

// reverse iterator query
reverse_iterator rbegin() noexcept;
reverse_iterator rend() noexcept;

// front/back
const_reference front() const noexcept;
reference front() noexcept;

const_reference back() const noexcept;
reference back() noexcept;
```

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
void swap(array_t &);
```

*Effect:* Exchanges the contents of a JSON array with those of the specified one.
Does not invoke any move, copy, or swap operations on individual elements.
All iterators and references remain valid. The past-the-end iterator is invalidated.

*Remarks:* No synchronization.

*Throws:* `std::domain_error` if JSON value is not of type `value_t::array`.
Example: `"cannot use swap() with string"`.

*Comlexity:* Constant.

```cpp
void swap(object_t &);
```

*Effect:* Exchanges the contents of a JSON object with those of the specified one.
Does not invoke any move, copy, or swap operations on individual elements.
All iterators and references remain valid. The past-the-end iterator is invalidated.

*Remarks:* No synchronization.

*Throws:* `std::domain_error` if JSON value is not of type `value_t::object`.
Example: `"cannot use swap() with string"`.

*Comlexity:* Constant.

```cpp
void swap(string_t &);
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
void push_back(const typename object_t::value_type &);
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
and added using `void push_back(const typename object_t::value_type&)`. Otherwise, 
the initializer list is converted to a JSON value and added using `void push_back(basic_json&&)`.

*Throws:*
  - If the JSON value type was wrong: `std::domain_error`.
  - The underlying data structure which holds the values also may throw an exception.
    Which one depends on the underlying data structure, which is defined by the template
    parameter `ArrayType`.

*Complexity:* Linear in the size of the initializer list.

*Remarks:* This function is required to resolve an ambiguous overload error,
because pairs like `{"key", "value"}` can be both interpreted as `object_t::value_type`
or `std::initializer_list<basic_json>`.

```cpp
reference operator+=(basic_json &&);
reference operator+=(const basic_json &);
reference operator+=(const typename object_t::value_type &);
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

*Complexity:* Amortized constant plus the complexity of the append operation of
the underlying data structure, which is defined by the template parameter `ArrayType`.

```cpp
template<class... Args> std::pair<iterator, bool> emplace(Args && ...);
```

```cpp
// insert
iterator insert(const_iterator pos, const basic_json & value);
iterator insert(const_iterator pos, basic_json && value);
iterator insert(const_iterator pos, size_type count, const basic_json & value);
iterator insert(const_iterator pos, const_iterator first, const_iterator last);
iterator insert(const_iterator pos, std::initializer_list<basic_json> values);

// erase
size_type erase(const typename object_t::key_type & key); // remove element from a JSON object given a key
void erase(const size_type index); // remove element from a JSON array given an index
template<class IteratorType, /* SFINAE omitted */ > IteratorType erase(IteratorType first, IteratorType last);
template<class IteratorType, /* SFINAE omitted */ > IteratorType erase(IteratorType pos);
```

#### Serialization / Deserialization

```cpp
// content to string
string_type str(int indent) const;

// output stream
friend std::ostream & operator<<(std::ostream & const basic_json &);

// input stream
friend std::isteram & operator>>(std::istream & basic_json &);
```

### Nested Class `basic_json::const_iterator`

Satisfies:

- RandomAccessIterator
- OutputIterator

```cpp
class const_iterator : public std::iterator< /*omitted */ >
{
public:
    // types
    using value_type        = typename basic_json::value_type;
    using difference_type   = typename basic_json::difference_type;
    using const_reference   = typename basic_json::const_reference;
    using reference         = typename basic_json::const_reference;
    using const_pointer     = typename basic_json::const_pointer;
    using pointer           = typename basic_json::const_pointer;
    using iterator_category = std::bidirectional_iterator_tag;

    // construction
    const_iterator();
    const_iterator(const const_iterator &) noexcept;
    const_iterator(const_iterator &&) noexcept;

    explicit const_iterator(const iterator &) noexcept;

    // assignment operators
    const_iterator & operator=(const const_iterator &) noexcept;
    const_iterator & operator=(const_iterator &&) noexcept;

    // element access operators
    const_reference operator*() const;
    const_poitner operator->() const;

    // movement operators
    const_iterator & operator++();    // pre-increment
    const_iterator   operator++(int); // post-increment

    const_iterator & operator--();    // pre-decrement
    const_iterator   operator--(int); // post-decrement

    const_iterator & operator+=(difference_type);
    const_iterator & operator-=(difference_type);

    const_iterator operator+(difference_type);
    const_iterator operator-(difference_type);

    // comparison operators
    bool operator==(const const_iterator &) const;
    bool operator!=(const const_iterator &) const;
    bool operator<=(const const_iterator &) const;
    bool operator< (const const_iterator &) const;
    bool operator>=(const const_iterator &) const;
    bool operator> (const const_iterator &) const;
};
```

### Nested Class `basic_json::iterator`

```cpp
class iterator : public const_iterator
{
public:
    // construction
    iterator();
    iterator(const iterator &) noexcept;
    iterator(iterator &&) noexcept;

    // assignment operators
    iterator & operator=(const iterator &) noexcept;
    iterator & operator=(iterator &&) noexcept;

    // element access operators
    reference operator*() const;
    poitner operator->() const;

    // movement operators
    iterator & operator++();    // pre-increment
    iterator   operator++(int); // post-increment

    iterator & operator--();    // pre-decrement
    iterator   operator--(int); // post-decrement

    iterator & operator+=(difference_type);
    iterator & operator-=(difference_type);

    iterator operator+(difference_type);
    iterator operator-(difference_type);
};
```

### Nested Class `basic_json::json_reverse_iterator`

```cpp
template<typename Base>
class json_reverse_iterator : public std::reverse_iterator<Base>
{
public:
    // types

    // shortcut to the reverse iterator adaptor
    using base_iterator = std::reverse_iterator<Base>;

    // the reference type for the pointed-to element
    using reference = typename Base::reference;

    // construction

    // create reverse iterator from iterator
    json_reverse_iterator(const typename base_iterator::iterator_type & it) noexcept;

    // create reverse iterator from base class
    json_reverse_iterator(const base_iterator & it) noexcept;

    // movement operators

    json_reverse_iterator   operator++(int);  // post-increment (it++)
    json_reverse_iterator & operator++();     // pre-increment (++it)

    json_reverse_iterator   operator--(int);  // post-decrement (it--)
    json_reverse_iterator & operator--();     // pre-decrement (--it)

    // add to iterator
    json_reverse_iterator & operator+=(difference_type i);

    // add to iterator
    json_reverse_iterator operator+(difference_type i) const;

    // subtract from iterator
    json_reverse_iterator operator-(difference_type i) const;

    // return difference
    difference_type operator-(const json_reverse_iterator & other) const;

    // access member functions

    reference operator[](difference_type n) const; // access to successor

    typename object_t::key_type key() const; // return the key of an object iterator

    reference value() const; // return the value of an iterator
};
```

### Nested Class `basic_json::const_reverse_iterator`

```cpp
using const_reverse_iterator = json_reverse_iterator<typename basic_json::const_iterator>;
```
	
### Nested Class `basic_json::reverse_iterator`

```cpp
using reverse_iterator = json_reverse_iterator<typename basic_json::iterator>;
```
	
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

## Acknowledgements

**TODO**

## References

- [RFC7159]	JavaScript Object Notation, <https://tools.ietf.org/html/rfc7159>

- [RFC6901]	JavaScript Object Notation Pointer, <https://tools.ietf.org/html/rfc6901>

- [RFC6902]	JavaScript Object Notation Patch, <https://tools.ietf.org/html/rfc6902>

- [nlohmann-json]	Example Implementation, <https://github.com/nlohmann/json>

