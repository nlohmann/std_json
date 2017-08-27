| Document Number | P0760R0                                   |
|-----------------|-------------------------------------------|
| Date            | 2017-08-27                                |
| Project         | Programming Language C++, Library Evolution Working Group |
| Reply-to        | Niels Lohmann <<mail@nlohmann.me>><br>Mario Konrad <<mario.konrad@gmx.net>> |

# JSON Library

<a name="introduction"></a>
## Introduction

This paper presents a proposal for *JavaScript Object Notation* [RFC7159] parsing and generation.
It proposes a library extension.


<a name="toc"></a>
## Table of Contents

- [Motivation](#motivation)
- [Design Goals](#design-goals)
- [Scope](#scope)
- [Examples](#examples)
  - [Handling in C++](#examples-handling-in-cpp)
  - [Literals](#examples-literals)
  - [Parsing](#examples-parse)
  - [Stringify](#examples-stringify)
  - [Read from iterator range](#examples-iterator-range)
  - [Playing nice with streams](#examples-streams)
  - [STL-like access](#examples-stl-access)
  - [Conversion from STL containers](#examples-conv-from-stl)
  - [Implicit Conversions](#examples-implicit-conv)
  - [Arbitrary Type Conversions](#examples-arbitrary-conv)
  - [Access using JSON Pointers](#examples-access-json-pointers)
  - [Flatten and Unflatten](#examples-flatten-unflatten)
  - [Support for JSON Patch](#examples-json-patch)
- [Terms and definitions](#terms-defs)
- [Technical Specification](#tech-spec)
  - [Header `<json>` synopsis](#header-synopsis)
  - [Enumeration `json_type`](#enum-json_type)
  - [Class Template `json_serializer`](#class-json_serializer)
  - [Class Template `json_parser`](#class-json_parser)
  - [Class Template `json_policy`](#class-json_policy)
  - [Class Template `basic_json`](#class-basic_json)
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
    - [Construction](#class-json_pointer-construction)
    - [Non-Modifying Operators](#class-json_pointer-non-modifying-op)
    - [Serialization](#class-json_pointer-serialization)
  - [Function Templates `to_json`](#func-to_json)
  - [Function Templates `from_json`](#func-from_json)
  - [Function Templates `make_json`](#func-make_json)
  - [Function Templates `flatten` and `unflatten`](#func-flatten-unflatten)
  - [User Defined Literals](#func-user-defined-literals)
  - [Template Function Specialization `swap`](#func-swap)
  - [Template Specialization `hash`](#func-hash)
- [Limitations and Interoperability](#limitations)
- [Acknowledgements](#acknowledgements)
- [References](#references)


<a name="motivation"></a>
## Motivation

Data represented in JSON format is very widely used for serialization of data. Prominent
uses are the numerous frameworks used by websites for asynchronous communication, as well
as configuration data.

The format itself is human readable and very lightweight.

There are numerous libraries written in C and C++, usable by C++, but none in the standard.
In comparison, JSON is part of the respective standard libraries of languages like Python, Swift, or D.

<a name="design-goals"></a>
## Design Goals

```
 ____                          ___________               __________
|    |_   parse/deserialize   |           | <---------- |  ________|__
| JSON | -------------------> | C++ json  |  interface  | | other     |
| text | <------------------- | container | ----------> |_| C++ types |
|______|  stringify/serialize |___________|               |___________|

```

Major:

- **Intuitive syntax**. In languages such as Python, JSON feels like a first class data type.
  Handling JSON in C++ should be as easy in an idiomatic way.

- **Ease of use**. Library extension which provides easy access and handling of data.
  Customizable for user defined data. Value semantics. This library extension aims to
  provide a DOM (domain object model), residing in memory, like other containers.

- **Arbitrary Data**. Handling of JSON at runtime to be able to process arbitrary data.
  No compile time structure definitions.

- **Integration**. Playing nice with existing containers and algorithms in the standard library.
  For the user, the interface of a JSON value should be a natural intersection of known
  STL containers like `std::map` or `std::vector` - just like JSON supports different types like
  key/value pairs and arrays.

- **RFC7159 Conformance**. Fully conformance to the JSON specification and explicitly stating the
  choice of possible design decisions such as limitations or behavioral details.

Minor:

- **Speed**. This proposal does not primarily focus on run time performance while sacrificing
  the major design goals. Unnecessary overhead should be avoided, but not at all costs.
  This may not suit everyone everywhere, but for a presumably large audience performance is
  good enough while providing a fairly easy data structure to work with.


<a name="scope"></a>
## Scope

This extension is a pure library extension. It does not require changes to the standard components.
The extension can be implemented in C++17.


<a name="examples"></a>
## Examples

In this section, we present examples how the JSON integration into C++ looks like. For these examples,
let us consider this JSON value:

```js
{
    "pi": 3.141,
    "flag": true,
    "name": "Ned Flanders",
    "nothing": null,
    "answer": {
        "everything": 42
    },
    "list": [0, 3, 6, 9, 12],
    "object": {
        "currency": "USD",
        "value": 42.99
    }
}
```

In this section, we use the terms "JSON value", "object", and "array" as described in
Sect. [Terms and definitions](#terms-defs).

<a name="examples-handling-in-cpp"></a>
### Handling in C++

The JSON value above can be constructed comfortably by using index operators:

```cpp
json data;

// floating-point number value
data["pi"] = 3.141;

// boolean value
data["flag"] = true;

// string value
data["name"] = "Ned Flanders";
data["name"] = std::string{"Ned Flanders"};

// JSON's null value
data["nothing"] = nullptr;

// object inside another object
data["answer"]["everything"] = 42;

// array of values
data["list"] = { 0, 3, 6, 9, 12 };

// object, defined as list of pairs
data["object"] = {{ "currency", "USD" }, { "value", 42.99 }};
```

The same JSON value can be represented more directly in C++:

```cpp
json data = {
    { "pi", 3.141 },
    { "flag", true },
    { "name", "Ned Flanders" },
    { "nothing", nullptr },
    { "answer",
        { "everything", 42 }
    },
    { "list", { 0, 3, 6, 9, 12 }},
    { "object", {
        { "currency", "USD" },
        { "value", 42.99 }
    }}
};
```

Of course there is also support for explicit JSON value construction, in case
the implicit does not suit the user's needs. For example:

```cpp
// a way to express the empty array []
json empty_array_explicit = json::array();

// ways to express the empty object {}
json empty_object_implicit = json({});
json empty_object_explicit = json::object();

// a way to express an _array_ of key/value pairs [["currency", "USD"], ["value", 42.99]]
json array_not_object = { json::array({"currency", "USD"}), json::array({"value", 42.99}) };
```


<a name="examples-literals"></a>
### Literals

Creating JSON data from a given string is straightforward:

```cpp
// create object from string literal
json j = "{ \"happy\": true, \"pi\": 3.141 }"_json;

// or even nicer with a raw string literal
auto j2 = R"(
  {
    "happy": true,
    "pi": 3.141
  }
)"_json;
```

Note that without appending the `_json` suffix, the passed string literal is not parsed,
but just used as JSON string value. That is, `json j = "{ \"happy\": true, \"pi\": 3.141 }"`
would just store the string `"{ "happy": true, "pi": 3.141 }"` rather than parsing the
actual JSON text.

<a name="examples-parse"></a>
### Parsing ("Deserialization")

The above example can also be expressed explicitly using `make_json(...)`:

```cpp
// parse explicitly
auto j3 = make_json("{ \"happy\": true, \"pi\": 3.141 }");
```

<a name="examples-stringify"></a>
### Stringify ("Serialization")

You can also get a string representation (i.e., serialize the JSON value to a JSON text):

```cpp
// explicit conversion to string
std::string s = j.str();
assert(s == "{\"happy\":true,\"pi\":3.141}");

// serialization with pretty printing
// pass in the amount of spaces to indent
std::cout << j.str(4) << std::endl;
```

This yields the output on the console:
```
{
    "happy": true,
    "pi": 3.141
}
```

<a name="examples-streams"></a>
### Playing nice with streams

The JSON value as proposed plays nice with streams:

```cpp
// parse/deserialize from standard input
json j;
std::cin >> j;

// stringify/serialize to standard output
std::cout << j;

// the setw manipulator is overloaded to set the indentation for pretty printing
std::cout << std::setw(4) << j << std::endl;
```

These operators work for any subclasses of `std::istream` or `std::ostream`.
Here is the same example with files:

```cpp
// read a JSON file (i.e., a file containing a JSON text)
std::ifstream ifs("config.json");
json j;
ifs >> j;

// write prettified JSON text to another file
std::ofstream ofs("pretty.json");
ofs << std::setw(4) << j << std::endl;
```

<a name="examples-iterator-range"></a>
### Read from iterator range

It is also possible to read JSON from an iterator range; that is, from any container
accessible by iterators whose content is stored as contiguous byte sequence, for
instance a `std::vector<uint8_t>`:

```cpp
std::vector<uint8_t> v = {'t', 'r', 'u', 'e'};
json j = make_json(v.begin(), v.end());
```

You may leave the iterators for the range `[begin, end)`:

```cpp
std::vector<uint8_t> v = {'t', 'r', 'u', 'e'};
json j = make_json(v);
```


<a name="examples-stl-access"></a>
### STL-like access

The proposed JSON value satisfies *Reversible Container* requirement.

Example JSON arrays (handling like `std::vector`):

```cpp
// create an array using push_back: ["foo", 1, true]
json j;
j.push_back("foo");
j.push_back(1);
j.push_back(true);

// also use emplace_back, now: ["foo", 1, true, 1.78]
j.emplace_back(1.78);

// iterate the array
for (json::iterator it = j.begin(); it != j.end(); ++it) {
  std::cout << *it << '\n';
}

// range-based for
for (auto & element : j) {
  std::cout << element << '\n';
}

// indexed access
const std::string tmp = j[0];
j[1] = 42;
bool foo = j.at(2);

// comparison
assert(j == "[\"foo\", 1, true]"_json);

// container operations
assert(j.size() == 3);
assert(j.empty() == false);
assert(j.type() == json_type::array);
j.clear();    // the array is empty again
```

Example JSON objects (handling like `std::map`):

```cpp
// create an object: {"foo": 23, "bar": false}
json o;
o["foo"] = 23;
o["bar"] = false;

// also use emplace; result: {"foo": 23, "bar": false, "weather": "sunny"}
o.emplace("weather", "sunny");

// find an entry
if (o.find("foo") != o.end()) {
  // there is an entry with key "foo"
}

// or simpler using count()
assert(o.count("foo") == 1);
assert(o.count("fob") == 0);

// special iterator member functions for objects
for (json::iterator it = o.begin(); it != o.end(); ++it) {
  std::cout << it.key() << " : " << it.value() << "\n";
}

// output:
// bar : false
// foo : 23
// weather : sunny
```

<a name="examples-conv-from-stl"></a>
### Conversion from STL containers

Any sequence container (`std::array`, `std::vector`, `std::deque`, `std::forward_list`,
`std::list`) whose values can be used to construct JSON types (e.g., integers, floating point
numbers, booleans, string types, or again STL containers described in this section) can be used
to create a JSON array.

The same holds for similar associative containers (`std::set`, `std::multiset`,
`std::unordered_set`, `std::unordered_multiset`), but in these cases the order of the
elements of the array depends how the elements are ordered in the respective STL container.
Refer to Sect. [Limitations and Interoperability](#limitations) for details on the ordering
of keys in objects.

Examples:

```cpp
std::vector<int> c_vector {1, 2, 3, 4};
json j_vec(c_vector); // [1, 2, 3, 4]

std::deque<double> c_deque {1.2, 2.3, 3.4, 5.6};
json j_deque(c_deque); // [1.2, 2.3, 3.4, 5.6]

std::list<bool> c_list {true, true, false, true};
json j_list(c_list); // [true, true, false, true]

std::forward_list<int64_t> c_flist {12345678909876, 23456789098765, 34567890987654, 45678909876543};
json j_flist(c_flist); // [12345678909876, 23456789098765, 34567890987654, 45678909876543]

std::array<unsigned long, 4> c_array {{1, 2, 3, 4}};
json j_array(c_array); // [1, 2, 3, 4]

std::set<std::string> c_set {"one", "two", "three", "four", "one"};
json j_set(c_set); // only one entry for "one" is used: ["four", "one", "three", "two"]

std::unordered_set<std::string> c_uset {"one", "two", "three", "four", "one"};
json j_uset(c_uset); // only one entry for "one" is used, maybe ["two", "three", "four", "one"]

std::multiset<std::string> c_mset {"one", "two", "one", "four"};
json j_mset(c_mset); // both entries for "one" are used, maybe ["one", "two", "one", "four"]

std::unordered_multiset<std::string> c_umset {"one", "two", "one", "four"};
json j_umset(c_umset); // both entries for "one" are used, maybe ["one", "two", "one", "four"]
```

Likewise, any associative key-value containers (`std::map`, `std::multimap`,
`std::unordered_map`, `std::unordered_multimap`) whose keys can construct an `std::string` and
whose values can be used to construct JSON types (see examples above) can be used to to create
a JSON object. Note that in case of multimaps only one key is used in the JSON object and the
value depends on the internal order of the STL container. Refer to
Sect. [Limitations and Interoperability](#limitations) for details on duplicate keys in objects.

Examples:

```cpp
std::map<std::string, int> c_map {{"one", 1}, {"two", 2}, {"three", 3}};
json j_map(c_map); // {"one": 1, "three": 3, "two": 2}

std::unordered_map<const char*, double> c_umap {{"one", 1.2}, {"two", 2.3}, {"three", 3.4}};
json j_umap(c_umap); // {"one": 1.2, "two": 2.3, "three": 3.4}

std::multimap<std::string, bool> c_mmap {{"one", true}, {"two", true}, {"three", false}, {"three", true}};
json j_mmap(c_mmap); // only one entry for key "three" is used, maybe {"one": true, "two": true, "three": true}

std::unordered_multimap<std::string, bool> c_ummap {{"one", true}, {"two", true}, {"three", false}, {"three", true}};
json j_ummap(c_ummap); // only one entry for key "three" is used, maybe {"one": true, "two": true, "three": true}
```

<a name="examples-implicit-conv"></a>
### Implicit Conversions

The type of the JSON object is determined automatically by the expression to store.
Likewise, the stored value is implicitly converted.

Examples:

```cpp
// strings
std::string s1 = "Hello, world!";
json js = s1;
std::string s2 = js;

// booleans
bool b1 = true;
json jb = b1;
bool b2 = jb;

// numbers
int i = 42;
json jn = i;
double f = jn;
```

You can also explicitly ask for the value:

```cpp
std::string vs = js.as<std::string>();
bool vb = jb.as<bool>();
int vi = jn.as<int>();
```

<a name="examples-arbitrary-conv"></a>
### Arbitrary Type Conversions

Every type can be serialized in JSON, not just STL-containers and scalar types. Usually,
you would do something along those lines:

```cpp
namespace ns {
    // a simple struct to model a person
    struct person {
        std::string name;
        std::string address;
        int age;
    };
}

ns::person p = {"Ned Flanders", "744 Evergreen Terrace", 60};

// convert to JSON: copy each value into the JSON object
json j;
j["name"] = p.name;
j["address"] = p.address;
j["age"] = p.age;

// ...

// convert from JSON: copy each value from the JSON object
ns::person p {
    j["name"].as<std::string>(),
    j["address"].as<std::string>(),
    j["age"].as<int>()
};
```

Through customization points, this boilerplate can be replaced to simplify the code:

```cpp
// create a person
ns::person p {"Ned Flanders", "744 Evergreen Terrace", 60};

// conversion: person -> json
json j = p;

std::cout << j << std::endl;
// {"address":"744 Evergreen Terrace","age":60,"name":"Ned Flanders"}

// conversion: json -> person
ns::person p2 = j;

// that's it
assert(p == p2);
```

This is achieved by providing conversion function for the custom type:

```cpp
using std::experimental::json;

namespace ns {
    void to_json(json & j, const person & p)
    {
        j = json{{"name", p.name}, {"address", p.address}, {"age", p.age}};
    }

    void from_json(const json & j, person & p)
    {
        p.name = j["name"].as<std::string>();
        p.address = j["address"].as<std::string>();
        p.age = j["age"].as<int>();
    }
} // namespace ns
```

Requirements for the custom type:
- DefaultConstructible



<a name="examples-access-json-pointers"></a>
### Access using JSON Pointers

For the given example `data` the following table shows the result for various
JSON pointers (see [RFC6901]). This is an alternative to address structured values.

| JSON Pointer            | Result |
| ----------------------- | ------ |
| `"/pi"`                 | 3.414  |
| `"/anwser/everything"`  | 42     |
| `"/list/1"`             | 3      |

written in code:

```cpp
const json::json_pointer p = "/answer/everything";
data[p] = 42;
```

For convenience, there is also an user defined literal:

```cpp
data["/answer/everything"_json_pointer] = 42;
```



<a name="examples-flatten-unflatten"></a>
### Flatten and Unflatten

Support for the common operation of `flatten` and `unflatten` of hierachical data.
The flat JSON value is an object with key/value pairs that represent the
structure where the keys are JSON pointers. The `unflatten` operation goes the other
way. Please note, both operations do not alter the original data, they return a copy.

Example:

```cpp
json j_hierachical =
{
    {"pi", 3.141},
    {"happy", true},
    {"name", "Ned"},
    {"nothing", nullptr},
    {
        "answer", {
            {"everything", 42}
        }
    },
    {"list", {1, 0, 2}},
    {
        "object", {
            {"currency", "USD"},
            {"value", 42.99},
            {"", "empty string"},
            {"/", "slash"},
            {"~", "tilde"},
            {"~1", "tilde1"}
        }
    }
};

json j_flat =
{
    {"/pi", 3.141},
    {"/happy", true},
    {"/name", "Ned"},
    {"/nothing", nullptr},
    {"/answer/everything", 42},
    {"/list/0", 1},
    {"/list/1", 0},
    {"/list/2", 2},
    {"/object/currency", "USD"},
    {"/object/value", 42.99},
    {"/object/", "empty string"},
    {"/object/~1", "slash"},
    {"/object/~0", "tilde"},
    {"/object/~01", "tilde1"}
};

auto flat = flatten(j_hierarchical);   // basically: flat == j_flat
auto hierarchical = unflatten(j_flat); // basically: hierarchical == j_hierarchical
```


<a name="examples-json-patch"></a>
### Support for JSON Patch

JSON patch [RFC6902] is supported, which allows to describe differences between two
JSON values - effectively allowing patch and diff operations known from Unix.

```cpp
// a JSON value
json j_original = R"({
  "baz": ["one", "two", "three"],
  "foo": "bar"
})"_json;

// a JSON patch (RFC6902)
json j_patch = R"([
  { "op": "replace", "path": "/baz", "value": "boo" },
  { "op": "add", "path": "/hello", "value": ["world"] },
  { "op": "remove", "path": "/foo"}
])"_json;

// apply the patch
json j_result = j_original.patch(j_patch);
```

results in:

```
{
   "baz": "boo",
   "hello": ["world"]
}
```

Computing the difference:

```cpp
// calculate a JSON patch from two JSON values
json::diff(j_result, j_original);
```

results in:

```
[
  { "op": "replace", "path": "/baz", "value": ["one", "two", "three"] },
  { "op": "remove","path": "/hello" },
  { "op": "add", "path": "/foo", "value": "bar" }
]
```


<a name="terms-defs"></a>
## Terms and definitions

The terminology used in this paper is intended to be consistent with terms used in
the JSON domain [RFC7159].

### value

The term *value* ([RFC7159] chapter 3) refers to an entity that represents information and can be
one of the following:

- object
- array
- number
- string
- boolean
- null

Objects and arrays are *structured values*, whereas strings, boolean, and null are *primitive values*.

### boolean

Values for *boolean* can be
- true
- false

### object

The term *object* ([RFC7159] chapter 4) refers to a structured value, consisting of zero
or more unordered name/value pairs. The term *name* refers to the key that identifies a value. This
is always of type *string*.

### array

The term *array* ([RFC7159] chapter 5) refers to a structured value, consisting of zero or
more values.

### string

The term *string* ([RFC7169] chapter 7) refers to a string consisting of zero
or more [Unicode] characters, beginning and ending with quotation marks.

### number

The term *number* ([RFC7159] chapter 6) represents a numerical value and is one of the
following types:

- integral signed
- integral unsigned
- floating point

Speical cases like *inf* and *NaN* are not permitted.

### JSON text

A *JSON text* ([RFC7159] chapter 2) is a serialized JSON value that conforms to the JSON grammar.


<a name="tech-spec"></a>
## Technical Specification

[TODO]


<a name="header-synopsis"></a>
### Header `<json>` synopsis

```cpp
namespace std {
inline namespace json_v1 {

    // JSON value types
    enum class json_type /*omitted*/;

    // Serializier, customization point
    template < /*omitted*/ > class json_serializer;

    // Parser, customization point
    template <class Base> class json_parser;

    // Policy class to configure the basic_json class template
    template < /*omitted*/ > struct json_policy;

    // generic base type basic_json
    template <class Policy> class basic_json;

    // function templates `to_json` for various types, customization points
    // ...

    // function templates `from_json` for various types, customization points
    // ...

    // function templates `make_json`
    // ...

    // function templates `flatten` and `unflatten
    // ...

    // default json object class
    using json = basic_json<json_policy<>>;

    // user defined literals
    json               operator "" _json(const char *, std::size_t);
    json::json_pointer operator "" _json_pointer(const char *, std::size_t);
}

// swap
template <class Policy> void swap(json_v1::basic_json<Policy> &,
                                  json_v1::basic_json<Policy> &);

// hash
template <> struct hash<json_v1::json>
{
    std::size_t operator()(const json_v1::json &) const;
};
}
```


<a name="enum-json_type"></a>
### Enumeration `json_type`

```cpp
enum class json_type : /*unspecified*/
{
    null                     = /*unspecified*/ ,
    boolean                  = /*unspecified*/ ,
    number_integral_signed   = /*unspecified*/ ,
    number_integral_unsigned = /*unspecified*/ ,
    number_floating_point    = /*unspecified*/ ,
    object                   = /*unspecified*/ ,
    array                    = /*unspecified*/ ,
    string                   = /*unspecified*/
};
```

*Remarks:* The semantics of the enumerators is ascending nature. This will be
relevant for lexicographical ordering.


<a name="class-json_serializer"></a>
### Class Template `json_serializer`

```cpp
template <class = void, class = void> struct json_serializer
{
    template <class BaseType, class ValueType>
    static void to_json(BaseType & j, ValueType && val) noexcept( /*omitted*/ );

    template <class BaseType, class ValueType>
    static void from_json(BaseType && j, ValueType & val) noexcept( /*omitted*/ );
};
```


<a name="class-json_parser"></a>
### Class Template `json_parser`

```cpp
template <class Base> class json_parser
{
public:
    template <class Iterator>
    json_parser(Iterator first, Iterator last);

    Base operator()();
};
```


<a name="class-json_policy"></a>
### Class Template `json_policy`

```cpp
template <
    class BooleanType                                              = bool,
    class IntegralSignedType                                       = std::int64_t,
    class IntegralUnsignedType                                     = std::uint64_t,
    class FloatingPointType                                        = double,
    class StringType                                               = std::string,
    template <class T, typename... Args> class ArrayType           = std::vector,
    template <class T, class U, typename... Args> class ObjectType = std::map,
    template <class U> class AllocatorType                         = std::allocator,
    template <class T, class SFINAE = void> class Serializer       = json_serializer
    template <class Base> class Parser                             = json_parser
    >
struct json_policy
{
    template <class Base> struct traits
    {
        using allocator_type         = AllocatorType<Base>;
        using pointer                = typename std::allocator_traits<allocator_type>::pointer;
        using const_pointer          = typename std::allocator_traits<allocator_type>::const_pointer;

        using boolean_type           = BooleanType;
        using integral_signed_type   = IntegralSignedType;
        using integral_unsigned_type = IntegralUnsignedType;
        using floating_point_type    = FloatingPointType;
        using string_type            = StringType;
        using array_type             = ArrayType<Base, AllocatorType<Base>>;
        using object_type            = ObjectType<
                                           string_type,
                                           Base,
                                           std::less<string_type>,
                                           AllocatorType<std::pair<const string_type, Base>>>;
    };

    template <class U, class SFINAE> using serializer_type = Serializer<U, SFINAE>;

    template <class Base> using parser_type = Parser<Base>;
};
```


<a name="class-basic_json"></a>
### Class Template `basic_json`

```cpp
template <class Policy> class basic_json
{
public:
    // type aliases

    using value_type             = basic_json;
    using reference              = value_type &;
    using const_reference        = const value_type &;
    using difference_type        = std::ptrdiff_t;
    using size_type              = std::size_t;

    using policy_traits          = typename Policy::template traits<basic_json>;

    using allocator_type         = typename policy_traits::allocator_type;
    using pointer                = typename policy_traits::pointer;
    using const_pointer          = typename policy_traits::const_pointer;

    using boolean_type           = typename policy_traits::boolean_type;
    using integral_signed_type   = typename policy_traits::integral_signed_type;
    using integral_unsigned_type = typename policy_traits::integral_unsigned_type;
    using floating_point_type    = typename policy_traits::floating_point_type;
    using string_type            = typename policy_traits::string_type;
    using array_type             = typename policy_traits::array_type;
    using object_type            = typename policy_traits::object_type;

    template <class U, class SFINAE = void>
    using serializer_type        = typename Policy::template serializer_type<U, SFINAE>;

    using parser_type            = typename Policy::template parser_type<basic_json>;

    // nested classes and types

    class const_iterator         = /*implementation defined*/ ;
    class iterator               = /*implementation defined*/ ;
    class const_reverse_iterator = /*implementation defined*/ ;
    class reverse_iterator       = /*implementation defined*/ ;

    class json_pointer;

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


<a name="class-basic_json-construction"></a>
#### Construction

##### Default constructor

```cpp
basic_json() noexcept;
```

*Effect:* Default constructor.

*Postcondition:* The type of the JSON value is `json_type::null`.

*Remarks:* Is trivially default constructible.

*Throws:* Nothing.

##### Constructor for default initialition according to value type

```cpp
explicit basic_json(json_type) noexcept;
```

*Effect:* Constructs an empty JSON value with the specified type. The contained data
will be initialized, according to the following table:

| Type                         | Value                                               |
| ---------------------------- | --------------------------------------------------- |
| `json_type::null`            | -                                                   |
| `json_type::object`          | Default constructed container of type `object_type` |
| `json_type::array`           | Default constructed container of type `array_type`  |
| `json_type::string`          | Default constructed string of type `string_type`    |
| `json_type::boolean`         | `false`                                             |
| `json_type::number_integer`  | `0`                                                 |
| `json_type::number_unsigned` | `0`                                                 |
| `json_type::number_float`    | `0.0`                                               |

*Throws:* Nothing.

*Complexity:* Constant.

##### Constructor for null value

```cpp
basic_json(std::nullptr) noexcept;
```

*Effect:* Constructs an empty JSON value of type `json_type::null`.

*Remarks:* Has the same effect as the default constructor. The rationale for this
constructor is to use the `nullptr` literal as a placeholder for JSON's null value.

*Throws:* Nothing.

#### Constructor from compatible types

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
type, e.g. any of `json_type`.

#### Construct with multiple copies of values

```cpp
basic_json(size_type count, const basic_json & value);
```

*Effect:* Constructs a JSON value with `count` number of elements of the specified
JSON value. The resulting type of the current object is `json_type::array`.

*Postcondition:*
- The JSON value is of type `json_type::array` and contains `count` copies
  of the specified value.
- If the specified `count` is `0`, the resulting JSON value is of type `json_type::array`,
  with an empty container.

*Throws:* `std::bad_alloc` if the container to hold the copies could not be constructed.

*Complexity:* Linear in number of elements.

#### Construct from an iterator range

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
- `std::domain_error` if the iterators originate from a JSON value of type `json_type::null`.
- `std::out_of_range` if the iterators originate from a primitive JSON value and do
  not meet the precondition for `first` and `last`.
- `std::bad_alloc` if the allocation of memory for the types `json_type::object`, `json_type::array`
  or `json_type::string` fails.

*Complexity:*
- For structured types (`json_type::object`, `json_type::array`): linear in number of number
  of elements in the range `[first,last)`.
- For primitive types: constant.

##### Copy constructor

```cpp
basic_json(const basic_json &);
```

*Effect:* Copy constructor.

*Throws:* `std::bad_alloc` if the allocation for object, array and string type failed.
Will never be thrown if the specified JSON value is primitive.

##### Move constructor

```cpp
basic_json(basic_json &&) noexcept;
```

*Effect:* Move constructor.

*Postcondition:* The moved-from JSON value will be left as type `json_type::null`.

*Throws:* Nothing.

*Complexity:* Constant.

##### Construct from initializer list

```cpp
basic_json(std::initializer_list<basic_json> init,
           bool automatic_type_deduction = true,
           json_type type_override = json_type::array);
```

*Effect:* Constructs a JSON value initialized with the specified parameters.
The type of the JSON value will be `json_type::array` or `json_type::object`, depending on the
specified of the parameters, according to the following rules:

1. If the initializer list is empty, an empty JSON value of type `json_type::object` is created.
2. If the initializer list consists of pairs whose first element is a string, a JSON value
   of type `json_type::object` is created, containing elements constructed from the pairs,
   with their first value (string) as key and the second value as data.
3. In all other cases, a JSON value of type `json_type::array` is created, containing copies
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
The value of the parameter `automatic_type_deduction` controls the following behavior:
- `true`: the constructor uses the rules above to determine the type for the JSON
  value (`json_type::object` or `json_type::array`).
- `false`: the constructor tries to use the type defined by `type_override`, which must be
  either `json_type::object` or `json_type::array`. If it is `json_type::object`, rule 2 must
  be valid or an exception is thrown.

*Throws:*
- `std::domain_error`: if the constructor is forced to construct a JSON value of
  type `json_type::object`, but is unable to because the initializer list does not contain
  pairs with the first element (key) a string.
- `std::bad_alloc`: if the underlying container can not be constructed, containing copies
  of the elements in the initializer list.

*Complexity:* The same complexity to create the underlying data structure defined
by `array_type` or `object_type`, and linear time in size of the initializer list.

##### Factory to create JSON array from initializer list

```cpp
static basic_json array(std::initializer_list<basic_json> = std::initializer_list<basic_json>{});
```

*Effect:* Factory method. Creates and returns a JSON value of type `json_type::array` containing
the elements specified by the initializer list, containing JSON values.

*Remarks:* This member function is needed to disambiguate the creation of JSON values of
type `json_type::array` and `json_type::object` for empty initializer lists and initializer lists
containing pairs.

*Complexity:* The same complexity to create the underlying data structure defined by `array_type`
and linear time in size of the initializer list.

##### Factory to create JSON object from initializer list

```cpp
static basic_json object(std::initializer_list<basic_json> = std::initializer_list<basic_json>{});
```

*Effect:* Factory method. Creates and returns a JSON value of type `json_type::object` containing
the elements specified by the initializer list, containing JSON values.

The initializer list must contain pairs, the type of the first element must be a string.

*Remarks:* This member function is needed to disambiguate the creation of JSON values of
type `json_type::array` and `json_type::object` for empty initializer lists and initializer lists
containing pairs.

*Throws:*
- `std::domain_error` if the a key of the key-value pairs is not a string.
  it is not possible to create all pairs.

*Complexity:* The same complexity to create the underlying data structure defined by `object_type`
and linear time in size of the initializer list.


<a name="class-basic_json-destruction"></a>
#### Destruction

```cpp
~basic_json();
```

*Effect:* Destroys the JSON value and frees all allocated memory.


<a name="class-basic_json-modifying-operators"></a>
#### Modifying Operators

##### Copy assignment

```cpp
basic_json & operator=(const basic_json &) noexcept( /*omitted*/ );
```

*Effect:* Copy assignment operator.

*Complexity:* Linear.

##### Move assignment

```cpp
basic_json & operator=(basic_json &&) noexcept( /*omitted*/ );
```

*Effect:* Move assignment operator.

*Complexity:* Linear.


<a name="class-basic_json-non-modifying-operators"></a>
#### Non-Modifying Operators

##### Comparison: equality

```cpp
friend bool operator==(const_reference lhs, const_reference rhs) noexcept;
```

*Effect:* Compares two JSON values for equality according to the following rules:
- Two JSON values are equal if (1) they are from the same type and (2)
  their stored values are equal.
- Integer numbers are automatically converted before comparison.
- Floating-point numbers are automatically converted before
  comparison. Floating-point numbers are compared indirectly: two
  floating-point numbers `f1` and `f2` are considered equal if neither
  `f1 > f2` nor `f2 > f1` holds. [TODO]
- Two JSON null values are equal.
- The types `object_type` and `array_type` must provide their own
  comparison `operator==` which will be used to compare stored values.

*Throws:* Nothing.

*Complexity:* Type dependent:

| Value type                   | Complexity                                           |
| ---------------------------- | ---------------------------------------------------- |
| `json_type::null`            | constant                                             |
| `json_type::object`          | depending on comparison of equality of `object_type` |
| `json_type::array`           | depending on comparison of equality of `array_type`  |
| `json_type::string`          | depending on comparison of equality of `string_type` |
| `json_type::boolean`         | constant                                             |
| `json_type::number_integer`  | constant                                             |
| `json_type::number_unsigned` | constant                                             |
| `json_type::number_float`    | constant                                             |

##### Comparison: inequality

```cpp
friend bool operator!=(const_reference lhs, const_reference rhs) noexcept;
```

*Effect:* Compares two JSON values for inequality by calculating the equivalent
of `not (lhs == rhs)`.

*Throws:* Nothing.

*Complexity:* Equal to `operator==(const_reference)`.

##### Comparison: lexicographical

```cpp
friend bool operator< (const_reference lhs, const_reference rhs) noexcept;  // (1)
friend bool operator<=(const_reference lhs, const_reference rhs) noexcept;  // (2)
friend bool operator> (const_reference lhs, const_reference rhs) noexcept;  // (3)
friend bool operator>=(const_reference lhs, const_reference rhs) noexcept;  // (4)
```

*Effect:* Lexicographical comparison for *less* (1), *less or equal* (2), *greater* (3)
and *greater or equal* (4). The following rules apply:

1. First comparison of `type()`: the types are in ascending order as specified by `json_type`
   (null < boolean < number < object < array < string).
   If both, `lhs` and `rhs` are of type `json_type::null`, the result is always `false`.
2. Second comparison if `type()` of `lhs` and `rhs` is the same: lexicographical comparison
   of the contained data type.

*Throws:* Nothing.

*Complexity:* Type dependent:

| Value type                   | Complexity                                            |
| ---------------------------- | ----------------------------------------------------- |
| `json_type::null`            | constant                                              |
| `json_type::object`          | depending on lexicographical comparison `object_type` |
| `json_type::array`           | depending on lexicographical comparison `array_type`  |
| `json_type::string`          | depending on lexicographical comparison `string_type` |
| `json_type::boolean`         | constant                                              |
| `json_type::number_integer`  | constant                                              |
| `json_type::number_unsigned` | constant                                              |
| `json_type::number_float`    | constant                                              |

##### Comparison with scalar types: equality

```cpp
template <typename ScalarType, /*SFINAE omitted*/ >
friend bool operator==(const_reference, const Scalartype) noexcept;

template <typename ScalarType, /*SFINAE omitted*/ >
friend bool operator==(const Scalartype, const_reference) noexcept;
```

*Effect:* Compare scalar types to the JSON value by converting the scalar
value into a JSON value, followed by a comparison of equality.

*Remarks:* Same properties as `bool operator==(const_reference, const_reference) noexcept;`

##### Comparison with scalar types: inequality

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

##### Query JSON value type

```cpp
constexpr json_type type() const noexcept;
```

*Effect:* Returns the type of the JSON value as a value of the `json_type` enumeration.

*Throws:* Nothing.

*Complexity:* Constant.

##### Query JSON value type (implicit)

```cpp
constexpr operator json_type () const noexcept;
```

*Effect:* Implicit type cast of the type of the JSON value to a value of the `json_type` enumeration.

*Throws:* Nothing.

*Complexity:* Constant.

##### Check: is JSON value of primitive type?

```cpp
constexpr bool is_primitive() const noexcept;
```

*Effect:* This function returns `true` if the JSON type is primitive, which is one of the following types:
  - `json_type::string`
  - `json_type::number_integral_signed`
  - `json_type::number_integral_unsigned`
  - `json_type::number_floating_point`
  - `json_type::boolean`
  - `json_type::null`

If not one of those types, it returns `false`.

*Throws:* Nothing.

*Complexity:* Constant.

##### Check: is JSON value of structured type?

```cpp
constexpr bool is_structured() const noexcept;
```

*Effect:* This function returns `true` if the JSON type is structured, which is one of the following types:
  - `json_type::object`
  - `json_type::array`

If not one of those types, it returns `false`.

*Throws:* Nothing.

*Complexity:* Constant.

##### Check: is JSON value a number?

```cpp
constexpr bool is_number() const noexcept;
```

*Effect:* This function returns `true` if the JSON type is a number, which is one of the following types:
  - `json_type::number_integral_signed`
  - `json_type::number_integral_unsigned`
  - `json_type::number_floating_point`

If not one of those types, it returns `false`.

*Throws:* Nothing.

*Complexity:* Constant.

##### Check: is JSON value of given type?

```cpp
constexpr bool is_null()              const noexcept;
constexpr bool is_boolean()           const noexcept;
constexpr bool is_string()            const noexcept;
constexpr bool is_integral_signed()   const noexcept;
constexpr bool is_integral_unsigned() const noexcept;
constexpr bool is_floating_point()    const noexcept;
constexpr bool is_object()            const noexcept;
constexpr bool is_array()             const noexcept;
```

*Effect:* This function returns `true` if the JSON type is of the specific one, `false` otherwise.

*Throws:* Nothing.

*Complexity:* Constant.

##### Check: Is a value stored?

```cpp
bool empty() const noexcept;
```

*Effect:* Returns `true` if a JSON value has no elements, `false` otherwise.
The return value depends on the different types and is defined as follows:

| Value type                            | return value                              |
| ------------------------------------- | ----------------------------------------- |
| `json_type::null`                     | `true`                                    |
| `json_type::boolean`                  | `false`                                   |
| `json_type::string`                   | `false`                                   |
| `json_type::number_integral_signed`   | `false`                                   |
| `json_type::number_integral_unsigned` | `false`                                   |
| `json_type::number_floating_point`    | `false`                                   |
| `json_type::object`                   | result of function `object_type::empty()` |
| `json_type::array`                    | result of function `array_type::empty()`  |

*Remarks:* This function does not return whether a string stored as JSON value
is empty -- it returns whether the JSON container itself is empty which is
`false` in the case of a string. Note that the null value is always empty.

*Remarks:* This function helps `basic_json` satisfying the [Container](http://en.cppreference.com/w/cpp/concept/Container)
requirements:
  - The complexity is constant.
  - Has the semantics of `begin() == end()`.

*Throws:* Nothing.

*Complexity:* Constant, as long as `array_type` and `object_type` satisfy the Container concept;
that is, their `empty()` functions have constant complexity.

##### Query the number of stored values

```cpp
size_type size() const noexcept;
```

*Effect:* Returns the number of elements in a JSON value.
The return value depends on the different types and is defined as follows:

| Value type                            | return value                             |
| ------------------------------------- | ---------------------------------------- |
| `json_type::null`                     | `0`                                      |
| `json_type::boolean`                  | `1`                                      |
| `json_type::string`                   | `1`                                      |
| `json_type::number_integral_signed`   | `1`                                      |
| `json_type::number_integral_unsigned` | `1`                                      |
| `json_type::number_floating_point`    | `1`                                      |
| `json_type::object`                   | result of function `object_type::size()` |
| `json_type::array`                    | result of function `array_type::size()`  |

*Remarks:* This function does not return the length of a string stored as JSON
value - it returns the number of elements in the JSON value which is `1` in the case of a string.
Note the null value always has a size of `0`.

*Remarks:* This function helps `basic_json` satisfying the [Container](http://en.cppreference.com/w/cpp/concept/Container)
requirements:
  - The complexity is constant.
  - Has the semantics of `std::distance(begin(), end())`.

*Throws:* Nothing.

*Complexity:* Constant, as long as `array_type` and `object_type` satisfy the Container concept;
that is, their size() functions have constant complexity.

##### Query the maximal number of storable values

```cpp
size_type max_size() const noexcept;
```

*Effect:* Returns the maximum number of elements a JSON value is able to hold due to
system or library implementation limitations. See [Limitations and Interoperability](#limitations)
for more information.
The return value depends on the different types and is defined as follows:

| Value type                            | return value                                 |
| ------------------------------------- | -------------------------------------------- |
| `json_type::null`                     | `0` (same as `size()`)                       |
| `json_type::boolean`                  | `1` (same as `size()`)                       |
| `json_type::string`                   | `1` (same as `size()`)                       |
| `json_type::number_integral_signed`   | `1` (same as `size()`)                       |
| `json_type::number_integral_unsigned` | `1` (same as `size()`)                       |
| `json_type::number_floating_point`    | `1` (same as `size()`)                       |
| `json_type::object`                   | result of function `object_type::max_size()` |
| `json_type::array`                    | result of function `array_type::max_size()`  |

*Remarks:* This function helps `basic_json` satisfying the [Container](http://en.cppreference.com/w/cpp/concept/Container)
requirements:
  - The complexity is constant.
  - Has the semantics of returning `b.size()` where `b` is the largest possible JSON value.

*Throws:* Nothing.

*Complexity:* Constant, as long as `array_type` and `object_type` satisfy the Container concept;
that is, their `max_size()` functions have constant complexity.

##### Count occurrences of key in object

```cpp
size_type count(typename object_type::key_type key) const;
```

*Effect:* Returns the number of occurrences of a key in a JSON object.
This method always returns `0` when executed on a JSON type that is not of
type `json_type::object`.

*Remarks:* If `object_type` is the default `std::map` type, the return value will
always be `0` (`key` was not found) or `1` (`key` was found).

*Throws:* Nothing.

*Complexity:* Depends on the underlying type of `object_type` for lookups.
`basic_json` adds constant complexity.

##### Find element in object

```cpp
iterator       find(typename object_type::key_type key);
const_iterator find(typename object_type::key_type key) const;
```

*Effect:* Finds an element in a JSON object with key equivalent to `key`.
If the element is not found or the JSON value is not an object, the result
of `end()` is returned.
This method always returns the result of `end()` when executed on a JSON type
that is not of type `json_type::object`.

*Remarks:* There is a const and a non-const overload of this function.

*Throws:* Nothing.

*Complexity:* Depends on the underlying type of `object_type` for lookups.
`basic_json` adds constant complexity.


<a name="class-basic_json-element-access"></a>
#### Element Access

##### Array element access by index (no bound checking)

```cpp
const_reference operator[](size_type) const;
      reference operator[](size_type);
```

*Effect:* Returns a reference to an element (a JSON value itself) of a JSON value of
type `json_type::array` at the specified index. Overloads for const and non-const versions.

The non-const overload will, in case the specified index is greater or equal to `size()`,
extend the array to a size capable of holding the specified index. All other values
in the new array between the previous size and the new size will be filled with JSON values
of type `json_type::null`. The reference to the newly created JSON value which is returned is
a default constructed JSON value.

The non-const overload also will convert the JSON value into a type `json_type::array`, if
it was of type `json_type::null`.

*Precondition:* The JSON value must be of type `json_type::array` or `json_type::null`.

*Postcondition:* The non-const overload invalidates all iterators, pointers and references.

*Remarks:*
- No synchronization.
- const overload: No index checks are being performed. Access to indices other than `0..size()-1`
  are undefined behavior.

*Throws:* `std::domain_error` if the type of the JSON value is different than `json_type::array`
or `json_type::null`.

*Complexity:*
- const overload: The complexity of a random element access of the underlying data structure
defined by `array_type`. Constant overhead by `basic_json`.
- non-const overload: If the specified index is in the range `0..size()-1`, it is the same
complexity as the const overload. If not, it is the same complexity as for the underlying
data structure defined by `array_type` to create a container of sufficient size.
Constant overhead by `basic_json`.

##### Object element access by key (no bound checking)

```cpp
const_reference operator[](const typename object_type::key_type &) const;
      reference operator[](const typename object_type::key_type &);
```

*Effects:* Returns a reference to an element (a JSON value itself) of a JSON value of
type `json_type::object` for the specified key. Overloads for const and non-const versions.

The non-const overload will, in case the specified key is not present in the JSON value,
default construct a JSON value for the key and return the reference to it.

The non-const overload also will convert the JSON value into a type `json_type::object`, if
it was of type `json_type::null`.

*Precondition:* The JSON value must be of type `json_type::object` or `json_type::null`.

*Postcondition:* The non-const overload invalidates all iterators, pointers and references.

*Remarks:*
- No synchronization.
- const overload: No checks are being performed. Specifying a non-existent key is undefined
  behavior.

*Throws:*
- const overload: `std::domain_error` if the type of the JSON value is different than `json_type::object`.
- non-const overload:  `std::domain_error` if the type of the JSON value is different than
  `json_type::object` or `json_type::null`.

*Complexity:*
- const overload: The complexity of an element access of the underlying data structure
  defined by `object_type`. Constant overhead by `basic_json`.
- non-const overload: If the specified key exists, it is the same complexity as the const overload.
  If not, it is the same complexity as for the underlying data structure defined by `object_type` to
  create a container and the element for the corresponding key. Constant overhead by `basic_json`.

##### Object element access by key (no bound checking); string literal overloads

```cpp
template <typename T> const_reference operator[](T *) const;
template <typename T>       reference operator[](T *);

template <typename T, std::size_t N> const_reference operator[](T * (&key)[N]) const;
template <typename T, std::size_t N>       reference operator[](T * (&key)[N]);
```

##### Object element access by JSON pointer (no bound checking)

```cpp
const_reference operator[](const json_pointer &) const;
      reference operator[](const json_pointer &);
```

*Effect:* Returns a reference to a JSON value specified by the JSON pointer.
Overloads for const and non-const versions.

The non-const overload will insert the requested JSON value, in particular:
- If the JSON pointer points to an object key which does not exist, the JSON value
  is default constructed.
  If the JSON value on which the member function is called upon is of type `json_type::null`,
  it will be converted to `json_type::object`.
- If the JSON pointer points to array index which does not exist, the array will
  be expanded to a be able to hold the specified index. All newly created JSON values
  in the array are constructed and of type `json_type::null`. The JSON value at the
  specified index is default constructed.
  If the JSON value on which the member function is called upon is of type `json_type::null`,
  it will be converted to `json_type::array`.
- The JSON pointer special value `-` is treated as array index past the last existing,
  i.e. appending to the existing array.
  If the JSON value on which the member function is called upon is of type `json_type::null`,
  it will be converted to `json_type::array`.

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
  defined by `object_type` or `array_type`, constant for primitive types.
  Constant overhead by `basic_json`.
- non-const overload: If the specified key exists, it is the same complexity as the const overload.
  If not, it is the same complexity as for the underlying data structure defined by `object_type`
  or `array_type` to create a container and the element for the corresponding key or index.
  Constant for primitive types. Constant overhead by `basic_json`.

##### Array element access by index (with bound checking)

```cpp
const_reference at(size_type) const;
      reference at(size_type);
```

*Effect:* Returns a reference to the element specified by an array index.
Overloads for const and non-const versions.

*Precondition:* The JSON value which this member function is called upon must be of
type `json_type::array`.

*Remarks:*
- Performs bound checks.
- No synchronization.

*Throws:*
- `std::domain_error` if the JSON value is not of type `json_type::array`.
- `std::out_of_range` if the specified index is not within the range `0..size()-1`.

*Complexity:* The same as for a random access of the underlying data structure defined
by `array_type`. Constant overhead by `basic_json`.

##### Object element access by key (with bound checking)

```cpp
const_reference at(const typename object_type::key_type &) const;
      reference at(const typename object_type::key_type &);
```

*Effect:* Returns a reference to the element for the specified key.
Overloads for const and non-const versions.

*Precondition:* The JSON value which this member function is called upon must be of
type `json_type::object`.

*Remarks:*
- Performs bound checks.
- No synchronization.

*Throws:*
- `std::domain_error` if the JSON value is not of type `json_type::object`.
- `std::out_of_range` if the specified key does not exist, equivalent to `find(key)==end()`.

*Complexity:* The same as for finding an element with a specified key of the underlying data
structure defined by `object_type`. Constant overhead by `basic_json`.

##### Object element access by JSON pointer (with bound checking)

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
defined by `object_type` or `array_type`, constant for primitive types.
Constant overhead by `basic_json`.

##### Object element access by key with default value

```cpp
template <class ValueType, /*SFINAE omitted*/ >
ValueType value(const typename object_type::key_type &, ValueType default_value) const;

string_type value(const typename object_type::key_type &, const char * default_value) const;

template <class ValueType, /*SFINAE omitted*/ >
ValueType value(const json_pointer &, ValueType default_value) const;

string_type value(const json_pointer &, const char * default_value) const;
```

*Effect:* Returns a copy of the data of an JSON value specified by a key or a default value,
if the key does not exist within the called JSON value of type `json_type::object`.

The JSON value to be found must be convertible into `ValueType`.

There is an overload specialized for `const char *` returning a `string_type`.

There is also an overload pair which takes a JSON pointer instead of an object key. If the
JSON pointer can not be resolved, the default value is returned.

*Remarks:* No synchronization.

*Throws:* `std::domain_error` if the JSON value which the member function is called upon is not
  of type `json_type::object`.

*Complexity:* The same complexity of for the underlying data structure defined by `object_type`
to find an element. Constant overhead by `basic_json`.


<a name="class-basic_json-container-access"></a>
#### Container Access

##### Iterator to the first element

```cpp
const_iterator begin()  const noexcept;
const_iterator cbegin() const noexcept;
      iterator begin()        noexcept;
```

*Effect:* Returns an iterator to the first element, overloads for const and non-const versions.

*Throws:* Nothing.

*Complexity:* Constant.

##### Iterator past the last element

```cpp
const_iterator end()  const noexcept;
const_iterator cend() const noexcept;
      iterator end()        noexcept;
```

*Effect:* Returns an iterator to one past the last element, overloads for const and
non-const versions.

*Throws:* Nothing.

*Complexity:* Constant.

##### Iterator to the first element of the reversed container

```cpp
reverse_const_iterator rbegin() const noexcept;
reverse_iterator       rbegin()       noexcept;
```

*Effect:* Returns an iterator to the reverse-beginning, overloads for const and non-const versions.

*Throws:* Nothing.

*Complexity:* Constant.

##### Iterator past the last element of the reversed container

```cpp
reverse_const_iterator rend() const noexcept;
reverse_iterator       rend()       noexcept;
```

*Effect:* Returns an iterator to the reverse-end, overloads for const and non-const versions.

*Throws:* Nothing.

*Complexity:* Constant.

##### Return first element

```cpp
const_reference front() const noexcept;
      reference front()       noexcept;
```

*Effect:* Returns a reference to the first element of the JSON value, overloads for
const and non-const references. Depending on the type of the JSON value:
- `json_type::object` and `json_type::array`: the function returns a reference to the first
   element in the underlying container.
- primitive type (except `json_type::null`): the function returns a reference to the value.

*Throws:* `std::domain_error` if the function was called on a JSON value with a type
of `json_type::null`.

*Remarks:* Calling this function on a structured JSON value (types `json_type::object` and
`json_type::array`) with empty underlying containers, the behavior is *undefined*.

*Complexity:*
- For primitive JSON value types (except type `json_type::null`): Constant.
- For JSON value of type `json_type::object`, the complexity to access the first element of
  the underlying container defined by `object_type`.
- For JSON value of type `json_type::array`, the complexity to access the first element of
  the underlying container defined by `array_type`.


##### Return last element

```cpp
const_reference back() const noexcept;
      reference back()       noexcept;
```

*Effect:* Returns a reference to the last element of the JSON value, overloads for
const and non-const references. Depending on the type of the JSON value:
- `json_type::object` and `json_type::array`: the function returns a reference to the last
  element in the underlying container.
- primitive type (except `json_type::null`): the function returns a reference to the value.

*Throws:* `std::domain_error` if the function was called on a JSON value with a type
of `json_type::null`.

*Remarks:* Calling this function on a structured JSON value (types `json_type::object` and
`json_type::array`) with empty underlying containers, the behavior is *undefined*.

*Complexity:*
- For primitive JSON value types (except type `json_type::null`): Constant.
- For JSON value of type `json_type::object`, the complexity to access the last element of
  the underlying container defined by `object_type`.
- For JSON value of type `json_type::array`, the complexity to access the last element of
  the underlying container defined by `array_type`.


<a name="class-basic_json-container-operations"></a>
#### Container Operations

##### Clears the content

```cpp
void clear() noexcept;
```

*Requires:* Nothing.

*Effect:* Clears the JSON value.

*Postcondition:* Clears the content of a JSON value and resets it to the default value as
if `basic_json(type())` had been called:

| Type                         | Value after `clear()` |
| ---------------------------- | --------------------- |
| `json_type::null`            | `null`                |
| `json_type::object`          | `{}`                  |
| `json_type::array`           | `[]`                  |
| `json_type::string`          | `""`                  |
| `json_type::boolean`         | `false`               |
| `json_type::number_integer`  | `0`                   |
| `json_type::number_unsigned` | `0`                   |
| `json_type::number_float`    | `0.0`                 |

*Remarks:* No synchronization.

*Throws:* Nothing.

##### Swap content of two JSON values

```cpp
void swap(reference) noexcept;
```

*Effect:* Exchanges the contents of the JSON value with those of the specified one.
Does not invoke any move, copy, or swap operations on individual elements.
All iterators and references remain valid. The past-the-end iterator is invalidated.

*Remarks:* No synchronization.

*Requires:* Nothing.

*Throws:* Nothing.

*Complexity:* Constant.

##### Swap content of array in JSON value

```cpp
void swap(array_type &);
```

*Effect:* Exchanges the contents of a JSON array with those of the specified one.
Does not invoke any move, copy, or swap operations on individual elements.
All iterators and references remain valid. The past-the-end iterator is invalidated.

*Remarks:* No synchronization.

*Throws:* `std::domain_error` if JSON value is not of type `json_type::array`.
Example: `"cannot use swap() with string"`.

*Complexity:* Constant.

##### Swap content of object in JSON value

```cpp
void swap(object_type &);
```

*Effect:* Exchanges the contents of a JSON object with those of the specified one.
Does not invoke any move, copy, or swap operations on individual elements.
All iterators and references remain valid. The past-the-end iterator is invalidated.

*Remarks:* No synchronization.

*Throws:* `std::domain_error` if JSON value is not of type `json_type::object`.
Example: `"cannot use swap() with string"`.

*Complexity:* Constant.

##### Swap content of string in JSON value

```cpp
void swap(string_type &);
```

*Effect:* Exchanges the contents of a JSON string with those of the specified one.
Does not invoke any move, copy, or swap operations on individual elements.
All iterators and references remain valid. The past-the-end iterator is invalidated.

*Remarks:* No synchronization.

*Throws:* `std::domain_error` when JSON value is not of type `json_type::string`
Example: `"cannot use swap() with boolean"`.

*Complexity:* Constant.

##### Add element to array

```cpp
void push_back(basic_json &&);
void push_back(const basic_json &);
void push_back(const typename object_type::value_type &);
```

[TODO: Discuss. The third overload uses `object_type::value_type` which seems strange in the context of adding to an array.]

*Requires:* The JSON value which the data is appended to must be of type
`json_type::array` or `json_type::null`.

*Effect:* Appends data to the JSON value. If the type was `json_type::null`, an
empty JSON value of type `json_type::array` is created and the specified data appended.
The appended data is stored in form of a JSON value and is owned by the JSON value
it was appended to.

*Remarks:* No synchronization.

*Throws:*
  - If the JSON value type was wrong: `std::domain_error`.
  - The underlying data structure which holds the values also may throw an exception.
    Which one depends on the underlying data structure, which is defined by the template
    parameter `array_type`.

*Complexity:* The operation relies on its underlying type for handling
arrays, which is defined by the template parameter `array_type`. The operation
`basic_json::push_back` does not introduce a higher complexity than amortized *O(1)*.

##### Add elements to object or array

```cpp
void push_back(std::initializer_list<basic_json>);
```

*Effect:* This function allows to use `push_back` with an initializer list.

In case
  1. the current JSON value is of type `json_type::object`, and
  2. the initializer list contains only two elements, and
  3. the first element of the initializer list is a string,

the initializer list is converted into an object element (JSON value of type `json_type::object`)
and added using `void push_back(const typename object_type::value_type&)`. Otherwise,
the initializer list is converted to a JSON value and added using `void push_back(basic_json&&)`.

*Throws:*
  - If the JSON value type was wrong: `std::domain_error`.
  - The underlying data structure which holds the values also may throw an exception.
    Which one depends on the underlying data structure, which is defined by the template
    parameter `array_type`.

*Complexity:* Linear in the size of the initializer list.

*Remarks:* This function is required to resolve an ambiguous overload error,
because pairs like `{"key", "value"}` can be both interpreted as `object_type::value_type`
or `std::initializer_list<basic_json>`.

##### Add element to array

```cpp
reference operator+=(basic_json &&);
reference operator+=(const basic_json &);
reference operator+=(const typename object_type::value_type &);
```

*Remarks:* The same requirements, effects, exceptions and complexity as
`void push_back(basic_json &&);`. Except, it returns the added JSON value
as reference.

##### Add elements to object or array

```cpp
reference operator+=(std::initializer_list<basic_json>);
```

*Remarks:* The same requirements, effects, exceptions and complexity as
`void push_back(std::initializer_list<basic_json>);`. Except, it returns the
added JSON value as reference.

##### Construct element in-place to array

```cpp
template<class... Args>
reference emplace_back(Args && ...);
```

*Effect:* Appends a new JSON value from the passed parameters to the JSON value.
If the function is called on a JSON value of type `json_type::null`, an empty
JSON value of type `json_type::array` is created before appending the newly created
value from the arguments. A reference to the newly created object is returned.

*Requires:* A `basic_json` object must be constructible from the template argument types.

*Throws:* `std::domain_error` when called on a JSON value of type other than
`json_type::array` or `json_type::null`.

*Remarks:* No synchronization.

*Complexity:* Amortized constant plus the complexity of the append operation of
the underlying data structure, which is defined by the template parameter `array_type`.

##### Construct element in-place to object

```cpp
template<class... Args>
std::pair<iterator, bool> emplace(Args && ...);
```

*Effect:* Inserts a new JSON value into a JSON value of type `json_type::object`,
constructed in-place with the given arguments, if there is no element with the key in the
container. If the function is called on a JSON value of type `json_type::null`, an empty
JSON value of type `json_type::object` is created before appending the newly created value.

If a JSON value with the same key already exists, its content will be replaced with
the newly created JSON value from the specified parameters. If an insertion of the newly
created JSON value is not possible, the underlying container remains unchanged.

The function returns a pair, containing:
  - *first:* an iterator to the inserted JSON value or the already existing JSON value if
    no insertion happened because of an existing key. If the insertion failed, the result
    of `end()` will be returned.
  - *second:* a `bool` denoting whether the insertion took place.

*Requires:* A `basic_json` object must be constructible from the template argument types.

*Throws:* `std::domain_error` when called on a JSON value of type other than
`json_type::object` or `json_type::null`.

*Remarks:* No synchronization.

*Complexity:* Amortized constant plus the complexity of the insert operation of
the underlying data structure, which is defined by the template parameter `object_type`.

##### Insert element to array

```cpp
iterator insert(const_iterator, const basic_json &);
iterator insert(const_iterator, basic_json &&);
```

*Effect:* Inserts a JSON value before the position specified by the iterator.
The function returns an iterator pointing to the newly inserted JSON value.

*Throws:*
- `std::domain_error` if the JSON value which the member function is called upon,
  is not of type `json_type::array`.
- `std::invalid_argument` if the specified iterator is not an iterator of `*this`.

*Postcondition:* If the insertion was not possible, the JSON value which the member
function was called upon, remains in the same state as before the function call.

*Complexity:* Constant plus the complexity of the insert operation of the underlying
data structure, defined by the type `array_type`.

##### Insert number of copies of element to array

```cpp
iterator insert(const_iterator, size_type, const basic_json &);
```

*Effect:* Inserts a number of copies of the specified JSON value before the given position,
specified by the iterator. The function returns an iterator pointing to the first newly
inserted JSON value. If the number of elements to insert is `0`, the function inserts no
elements and returns the position specified by the iterator.

*Throws:*
- `std::domain_error` if the JSON value which the member function is called upon,
  is not of type `json_type::array`.
- `std::invalid_argument` if the specified iterator is not an iterator of `*this`.

*Postcondition:* If the insertion was not possible, the JSON value which the member
function was called upon, remains in the same state as before the function call.
Either all new JSON values can be inserted or none.

*Complexity:* Linear in number of copies to insert, plus the complexity of the insert
operation of the underlying data structure, defined by the type `array_type`.

##### Insert elements from range to array

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
  is not of type `json_type::array`.
- `std::invalid_argument` if the specified iterator is not an iterator of `*this`.
- `std::invalid_argument` if the specified iterators `first` and `last` do not belong
  to the same JSON value.
- `std::invalid_argument` if the specified iterators `first` or `last` are iterators
  of the container for which `insert` is being called.

*Precondition:* The JSON value which the elements are inserted into must be of
type `json_type::array`.

*Postcondition:* If the insertion was not possible, the JSON value which the member
function was called upon, remains in the same state as before the function call.
Either all new JSON values can be inserted or none.

*Complexity:* Linear in number of copies to insert, i.e. `std::distance(first, last)`,
plus the complexity of the insert operation of the underlying data structure, defined by
the type `array_type`.

##### Insert elements to array

```cpp
iterator insert(const_iterator, std::initializer_list<basic_json>);
```

*Effect:* Inserts JSON values from the initializer list before the position
specified by the iterator. The function returns an iterator pointing to the first
newly inserted JSON value. If the initializer list is empty, the function inserts
no elements and returns the specified iterator.

*Remarks:* No synchronization.

*Precondition:* The JSON value which the elements are inserted into must be of
type `json_type::array`.

*Postcondition:* If the insertion was not possible, the JSON value which the member
function was called upon, remains in the same state as before the function call.
Either all new JSON values can be inserted or none.

*Throws:*
- `std::domain_error` if the JSON value which the member function is called upon,
  is not of type `json_type::array`.
- `std::invalid_argument` if the specified iterator is not an iterator of `*this`.

*Complexity:* Linear in size of the initializer list, plus the complexity of the
insert operation of the underlying data structure, defined by the type `array_type`.

##### Remove object element by key

```cpp
size_type erase(const typename object_type::key_type &);
```

*Effect:* Removes elements from a JSON value of type `json_type::object`.
The function returns the number of removed elements, which is zero or a positive
number.

*Postcondition:* References and iterators to the erased elements are invalidated.
Other references and iterators are not affected.

*Throws:* `std::domain_error` if the type of the JSON value was other than `json_type::object`.

*Remarks:* No synchronization.

*Complexity:* The same complexity to find and erase all occurrences of the specified key
of the underlying data structure.

##### Remove array element by index

```cpp
void erase(const size_type);
```

*Effect:* Removes an element at the specified index (in the range `[0, size()-1]`)
from a JSON value of type `json_type::array`.

*Postcondition:* References and iterators are invalidated.

*Throws:*
- `std::domain_error` if the JSON value which the member function is called upon,
  is not of type `json_type::array`.
- `std::out_of_range` if the specified index is not in the range `[0, size()-1]`.

*Remarks:* No synchronization.

*Complexity:* The same complexity to erase the element at the specified index in
the underlying data structure.

##### Remove element by iterator

```cpp
template<class IteratorType, /*SFINAE omitted*/ >
IteratorType erase(IteratorType pos);
```

*Effects:* Erases the element from the JSON value at the position specified by the iterator.
The member function returns an iterator pointing to the element after the erased element.
Depending on the type of the JSON value:
- `json_type::object` / `json_type::array`: the element at the specified position will
  be erased.
- Primitive types (except `json_type::null`): the resulting JSON value will be of
  type `json_type::null`, its content will be erased.

*Precondition:* The iterator type `IteratorType` must be of `iterator` or `const_iterator`.

*Postcondition:* References and iterators are invalidated.

*Throws:*
- `std::domain_error` if the JSON value which the member function is called upon,
  is of type `json_type::null`.
- `std::invalid_argument` if the specified iterator does not belong to the current
  JSON value.
- `std::out_of_range` if the specified position is not in the range `[begin(), end())`.

*Remarks:* No synchronization.

*Complexity:* Depends on the type of the JSON value:
- `json_type::object`: The complexity of erasure of an element of the underlying
  data structure, defined by `object_type`. Constant overhead by `basic_json`.
- `json_type::array`: The complexity of erasure of an element of the underlying
  data structure, defined by `array_type`. Constant overhead by `basic_json`.
- `json_type::string`: Complexity of the destruction of the `string_type`. Constant
  overhead by `basic_json`.
- others: Constant.

##### Remove elements in range

```cpp
template<class IteratorType, /*SFINAE omitted*/ >
IteratorType erase(IteratorType first, IteratorType last);
```

*Effect:* Erases all elements from the JSON value specified by the range `[first, last)`.
The function returns an iterator pointing to the element after the last erased. Erasing an
empty range is a no-op. Depending on the type of the JSON value:
- `json_type::object` / `json_type::array`: The elements in the specified range are
  erased from the JSON value.
- Primitive types (except `json_type::null`): the resulting JSON value will be of
  type `json_type::null`, its content will be erased.

*Precondition:* The iterator type `IteratorType` must be of `iterator` or `const_iterator`.

*Postcondition:* References and iterators are invalidated.

*Throws:*
- `std::domain_error` if the JSON value which the member function is called upon,
  is of type `json_type::null`.
- `std::invalid_argument` if of of the specified iterators does not belong to the current
  JSON value.
- `std::out_of_range` if the specified position is not in the range `[begin(), end())`.

*Remarks:* No synchronization.

*Complexity:* Depends on the type of the JSON value:
- `json_type::object`: The complexity of erasure of the range of elements of the underlying
  data structure, defined by `object_type`. Constant overhead by `basic_json`.
- `json_type::array`: The complexity of erasure of the range of elements of the underlying
  data structure, defined by `array_type`. Constant overhead by `basic_json`.
- `json_type::string`: Complexity of the destruction of the `string_type`. Constant
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

A JSON pointer defines a string syntax for identifying a specific value
within a JSON document.

```cpp
class json_pointer
{
    friend class basic_json;

public:
    // construction

    json_pointer();
    explicit json_pointer(const std::string &);

    json_pointer(const json_pointer &) = default;
    json_pointer & operator=(const json_pointer &) = default;

    json_pointer(json_pointer &&) = default;
    json_pointer & operator=(json_pointer &&) = default;

    // non-modifying operators

    bool operator==(const json_pointer &) const noexcept;
    bool operator!=(const json_pointer &) const noexcept;

    // serialization

    std::string str() const;
    operator std::string() const;
};
```

Excerpt from [RFC6901] / Section 3, the ABNF of the JSON pointer syntax:

```
json-pointer    = *( "/" reference-token )
reference-token = *( unescaped / escaped )
unescaped       = %x00-2E / %x30-7D / %x7F-10FFFF   ; %x2F ('/') and %x7E ('~') are excluded from 'unescaped'
escaped         = "~" ( "0" / "1" )                 ; representing '~' and '/', respectively
```


<a name="class-json_pointer-construction"></a>
#### Construction

```cpp
json_pointer();
```

*Effect:* Default constructor. Same behavior as the empty JSON pointer: Points to the whole JSON document.

*Complexity:* Constant.

```cpp
explicit json_pointer(const std::string &);
```

*Effect*: Constructs a `json_pointer` object, initialized with the specified string.

*Remarks:* The syntax of the specified string must be as described in [RFC6901] section 3.

*Throws:*
- `std::domain_error` if the specified string is non-empty and has an invalid syntax.


<a name="class-json_pointer-non-modifying-op"></a>
#### Non-Modifying Operators

```cpp
bool operator==(const json_pointer &) const noexcept;
```

*Effect:* Compares two JSON pointers for equality, i.e. point to the same location in
a JSON structure.

*Throws:* Nothing.

```cpp
bool operator!=(const json_pointer &) const noexcept;
```

*Effect:* Compares two JSON pointers for inequality, i.e. do not point to the same location in
a JSON structure.

*Throws:* Nothing.


<a name="class-json_pointer-serialization"></a>
#### Serialization

```cpp
std::string str() const;
```

*Returns:* A string representation of the JSON pointer.

*Requires:* Invariance. For each `json_pointer` `ptr` it holds: `ptr == json_pointer(ptr.str())`.

```cpp
operator std::string() const;
```

*Effect:* Typecast operator. Same as member function `str()`.


<a name="func-to_json"></a>
### Function Templates `to_json`

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

| Overload | Implicitly convertible into             | Resulting JSON value type             |
| -------- | --------------------------------------- | ------------------------------------- |
| (1)      | `BasicJsonType::boolean_type`           | `json_type::boolean`                  |
|          | `BasicJsonType::integral_signed_type`   | `json_type::number_integral_signed`   |
|          | `BasicJsonType::integral_unsigned_type` | `json_type::number_integral_unsigned` |
|          | `BasicJsonType::floating_point_type`    | `json_type::number_floating_point`    |
|          |                                         | `json_type::null`                     |
| (2)      | `BasicJsonType::integral_signed_type`   | `json_type::number_integral_signed`   |
| (3)      | `BasicJsonType::string_type`            | `json_type::string`                   |
| (4)      | `BasicJsonType::array_type`             | `json_type::array`                    |
| (5)      | `BasicJsonType::object_type`            | `json_type::object`                   |

*Postcondition:*
- The output parameter of type `BasicJsonType` is a valid JSON value of appropriate
  type `json_type` (see table *Precondition*).
- If the value is a floating point number and either `inf` or `NAN`, the resulting
  JSON value is of type `json_type::null`.

*Remarks:* This function is a customization point. Custom implementations of `to_json` may
be provided by the user to map custom data to JSON values.

*Throws:*
- Overloads (1) and (2): Nothing.
- Overloads (3), (4), (5): `std::bad_alloc` if the memory allocation to hold the copy of the specified
  data fails.

*Complexity:*
- Overloads (1) and (2): Constant.
- Overload (3): The same complexity to copy the the string, defined by the type `string_type`.
- Overloads (4), (5): The same complexity as the underlying data structures, defined by
  either `array_type` or `object_type` to create the container and linear in number of elements to copy.


<a name="func-from_json"></a>
### Function Templates `from_json`

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
- all: The specified JSON value must not be of type `json_type::null`.
- (11): The specified forward list must contain the same data type as the JSON value.

*Postcondition:* The specified data type holds a copy of the contents of the given
JSON value.

*Remarks:* This function is a customization point. Custom implementations of `from_json` may
be provided by the user to map custom data to JSON values.

*Throws:*
- `std::domain_error` if the JSON value is not of the appropriate type `json_type`, according
  to the specified value type.
- (11): `std::domain_error` if the JSON value is of type `json_type::null` or the data
  type contained within the JSON value does not match the `value_type` of the specified
  forward list.

*Complexity:*
- Overloads (1), (2), (3), (4), (5), (10): Constant.
- Overload (6): The same complexity as to copy a string defined by the type `string_type`.
- Overload (7), (8): The same complexity as to copy data into the specified container,
  defined by `array_type`.
- Overload (9): The same complexity as to copy data into the specified container,
  defined by `object_type`.
- Overload (11): The same complexity as to copy data into the specified list.


<a name="func-make_json"></a>
### Function Templates `make_json`

```cpp
// from iterators
template <class Iterator, class Policy = json_policy<>>
basic_json<Policy> make_json(Iterator first, Iterator last);

// from array
template <class T, std::size_t N, class Policy = json_policy<>>
basic_json<Policy> make_json(const T (&arr)[N]);

// from string literal
template <class CharT, class Policy = json_policy<>>
basic_json<Policy> make_json(const std::basic_string_view<CharT> s);

// from container
template <class Container, class Policy = json_policy<> /*TODO:SFINAE*/>
basic_json<Policy> make_json(const Container & c);
```

[TODO: specification]


<a name="func-flatten-unflatten"></a>
### Function Templates `flatten` and `unflatten`

```cpp
template <class BaseType> BaseType flatten(const BaseType & value);

template <class BaseType> BaseType unflatten(const BaseType & value);
```

[TODO: specification]


<a name="func-user-defined-literals"></a>
### User Defined Literals

```cpp
json operator "" _json(const char *, std::size_t);
```

*Effect:* Constructs a JSON value (with all necessary sub-nodes) from the specified string.

*Remarks:*
- Only complete and syntactically correct JSON data representation will result in success,
  partial construction is not supported.
- The result of this user defined literal applies only to the default configuration
  of the class template `basic_json`.

```cpp
json::json_pointer operator "" _json_pointer(const char *, std::size_t);
```

*Effect:* Constructs a JSON pointer from the specified string.

*Remarks:*
- The result of this user defined literal applies only to the default configuration
  of the class template `basic_json`.


<a name="func-swap"></a>
### Template Function Specialization `swap`

```cpp
template <> void swap(json_v1::json &, json_v1::json &)
    noexcept(
           is_nothrow_move_constructible<json_v1::json>::value
        && is_nothrow_move_assignable<json_v1::json>::value);
```

*Effect:* Exchanges the given JSON values.


<a name="func-hash"></a>
### Template Specialization `hash`

```cpp
template <> struct hash<json_v1::json>
{
    std::size_t operator()(const json_v1::json &) const;
};
```

*Effect:* The template specialization to calculate the hash value for the specified
JSON value.

<a name="limitations"></a>
## Limitations and Interoperability

### Limitations

Section 9 of RFC7159 allows a conforming implementation to:

- set limits on the size of texts that it accepts
- set limits on the maximum depth of nesting
- set limits on the range and precision of numbers
- set limits on the length and character contents of strings

## Interoperability

Section 4 of RFC7159 states that keys in an object SHOULD be unique and

> Implementations whose behavior does not depend on member
  ordering will be interoperable in the sense that they will not be
  affected by these differences.

Sect. 6 of RFC7159:

> Note that when such software is used, numbers that are integers and
  are in the range [-(2**53)+1, (2**53)-1] are interoperable in the
  sense that implementations will agree exactly on their numeric
  values.

Section 8.1 of RFC7159:

> JSON texts that are encoded in UTF-8 are interoperable in the sense
  that they will be read successfully by the maximum number of
  implementations

<a name="acknowledgements"></a>
## Acknowledgements

[TODO]


<a name="references"></a>
## References

- [RFC7159]	JavaScript Object Notation, <https://tools.ietf.org/html/rfc7159>

- [RFC6901]	JavaScript Object Notation Pointer, <https://tools.ietf.org/html/rfc6901>

- [RFC6902]	JavaScript Object Notation Patch, <https://tools.ietf.org/html/rfc6902>

- [nlohmann-json]	Example Implementation, <https://github.com/nlohmann/json>

- [Unicode] The Unicode Consortium, "The Unicode Standard", <http://www.unicode.org/versions/latest/>
