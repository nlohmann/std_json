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
// comparison: equal, non-converting
bool operator==(const basic_json &) noexcept;

// comparison: not equal, non-converting
bool operator!=(const basic_json &) noexcept;
```

The comparison for equal and non-equal compare the JSON object, including all children.

#### Query Member Functions

```cpp
// type query
constexpr value_t type() const noexcept;
constexpr operator value_t () const noexcept;

// individual type query
constexpr bool is_primitive() const noexcept;
constexpr bool is_structured() const noexcept;
constexpr bool is_null() const noexcept;
constexpr bool is_boolean() const noexcept;
constexpr bool is_string() const noexcept;
constexpr bool is_number() const noexcept;
constexpr bool is_integral_signed() const noexcept;
constexpr bool is_integral_unsigned() const noexcept;
constexpr bool is_floating_point() const noexcept;
constexpr bool is_object() const noexcept;
constexpr bool is_array() const noexcept;
constexpr bool is_discarded() const noexcept;

// size and capacity
bool empty() const noexcept; // checks whether the container is empty
size_type size() const noexcept; // returns the number of elements
size_type max_size() const noexcept; // returns the maximum possible number of elements

// other member functions

// returns the number of occurrences of a key in a JSON object
size_type count(typename object_t::key_type key) const;

// find an element in a JSON object
iterator find(typename object_t::key_type key);
const_iterator find(typename object_t::key_type key) const;
```

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
// swap
void swap(reference other) noexcept;
void swap(array_t &);
void swap(object_t &);
void swap(string_t &);
```

```cpp
void push_back(basic_json &&);
void push_back(const basic_json &);
void push_back(const typename object_t::value_type &);
void push_back(std::initializer_list<basic_json>);
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
reference operator+=(basic_json &&);
reference operator+=(const basic_json &);
reference operator+=(const typename object_t::value_type &);
reference operator+=(std::initializer_list<basic_json>);

// emplace
template<class... Args> void emplace_back(Args && ...);
template<class... Args> std::pair<iterator, bool> emplace(Args && ...);

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

