---
layout: post
usemathjax: true
title: Reflection-based JSON in C++ at Gigabytes per Second
tags: Modern C++, JSON, Serialization, Reflection, C++26, Performance
---

JSON (JavaScript Object Notation) is a popular format for storing and transmitting data. It uses human-readable text to represent structured data in the form of attribute–value pairs and arrays. E.g., `{"age":5, "name":"Daniel", toys:["wooden dog", "little car"]}`.
Ingesting and producing JSON documents can be a performance bottleneck.

Thankfully, a few JSON parsers have shown that we can process JSON at high speeds, reaching gigabytes per second[^1][^2].

However, producing and ingesting JSON data can remain a chore in C++. The programmer often needs to address potential errors such as unexpected content.

Often, the programmer only needs to map the content to and from a native C/C++ data structure. E.g., our JSON example
might correspond to the following C++ structure:

```cpp
struct kid {
    int age;
    std::string name;
    std::vector<std::string> toys;
};
```

Hence, we would often like to write simple code such as `kid k; store_json(k, out)` or `kid k; load_json(k, in)`. Ideally, we would like this code to be fast and safe. That is, we would like to write and read our data structures at gigabytes per second. Further, we want proper validation especially since the JSON data might come from the Internet.

A library such as simdjson already allows elegant deserialization, e.g., we can make the following example work:

```cpp
struct kid {
  int age;
  std::string name;
  std::vector<std::string> toys;
};

void demo() {
  auto json_str =
      R"({"age": 12, "name": "John", "toys": ["car", "ball"]})"_padded;
  simdjson::ondemand::parser parser;
  auto doc = parser.iterate(json_str);
  kid k = doc.get<kid>();
}
```

However, for the time being, it requires a [bit of effort](https://github.com/simdjson/simdjson/blob/master/doc/basics.md#adding-support-for-custom-types) on the part of the programmer---as they need to write a custom function.

Similarly, going from the C++ structures to JSON can be made
convenient, but typically only after the programmer has done
a bit of work, writing custom code. E.g., in the popular JSON for Modern C++ ([nlohmann/json](https://github.com/nlohmann/json)) library, we need to provide glue functions:

{% raw %}
```cpp
    void to_json(json& j, const person& p) {
        j = json{{"name", p.name}, {"address", p.address}, {"age", p.age}};
    }

    void from_json(const json& j, person& p) {
        j.at("name").get_to(p.name);
        j.at("address").get_to(p.address);
        j.at("age").get_to(p.age);
    }
```
{% endraw %}

We know that this extra effort (writing data-structure aware code) is unnecessary because other programming languages (Java, C#, Zig, Rust, Python, etc.) allow us to 'magically'
serialize and deserialize with almost no specialized code.

One key missing feature in [C++ is sufficiently powerful *reflection*](https://stackoverflow.com/questions/8948087/is-there-any-major-programming-language-that-doesnt-support-any-form-of-reflect). Reflection in programming languages refers to a mechanism that allows code to introspect its own structure.
That is, if you cannot conveniently tell C++ that if a class
has a `std::string name` attribute, then you should automatically search for a string value matching the key `name` in the JSON.

Thankfully, C++ is soon getting reflection in C++26[^3]. And it is getting powerful reflection: reflective metaprogramming. That is, future versions of C++ will allow us to solve the serialization and deserialization problem we described at compile time: a software library can automate the production and consumption of JSON to and from a native data structure. 
Such a feature will simplify the life of the C++ programmer. Furthermore, because it can be done automatically as part of well tested library at compile time, it can generate fast and safe code.


[Herb Sutter wrote about this development](https://herbsutter.com/2024/07/02/trip-report-summer-iso-c-standards-meeting-st-louis-mo-usa/):

> This is huge, because reflection (including generation) will be by far the most impactful feature C++ has ever added since C++98, and it will dominate the next decade and more of C++ usage.


Though many programming languages have reflection, the
reflective metaprogramming provided by the upcoming
C++ standard is unique to our knowledge. Some programming
languages have long had powerful reflection
capabilities, but they are mostly a runtime feature:
they do not allow the equivalent of generating
code. Other programming languages allow compile-time
logic generation but they do not always allow optimal
performance. We expect that the C++ approach might be
especially appropriate for high performance.

Though current C++ compilers do not yet support reflection, we have access to prototypical compilers. In particular,
engineers from Bloomberg maintain a [fork of LLVM](https://github.com/bloomberg/clang-p2996/tree/p2996). Though such an implementation should not be used in production and is subject to bugs and other limitations, it should be sufficient to test the performance: how fast can serialization-based JSON processing be in C++?
Importantly, we are not immediately concerned with compilation speed: the [Bloomberg fork is purposefully
suboptimal in this respect](https://www.reddit.com/r/cpp/comments/1c7ugyg/metaprogramming_benchmark_p2996_vs_p1858_vs/?rdt=55615).

To test out this new C++ feature, [we wrote a prototype](https://github.com/simdjson/experimental_json_builder/)
that can serialize and deserialize data structures to and from JSON strings automatically. Given an instance of `kid`, we can convert it to a JSON string without any effort, the compile-time reflection does the work:

```cpp
  kid k{12, "John", {"car", "ball"}};
  std::print("My JSON is {}\n", simdjson::json_builder::to_json_string(k));
```


### Benchmark Results with Speed-up


Automatically converting your C++ data types into JSON strings is convenient. But is it fast? To find out, we have used 2 instances for benchmarking, an "Artificial instance" that was obtained using the [JSON Generator](https://json-generator.com/) Website and the other one is the [twitter.json](https://github.com/simdjson/simdjson/blob/master/jsonexamples/twitter.json) file that is used by many other serialization libraries for benchmarking. 


We test on two systems (Apple and Intel/Linux) with the experimental LLVM Bloomberg compiler.

M3 MAX Macbook Pro:

| Test Instance            | Library               | Speed (MB/s) |  speedup |
|--------------------------|-----------------------|--------------|----------|
| Twitter Serialization    | nlohmann/json         | 100          |          |
|                          | our C++26 serializer  | 1900         |   19×    |
| Artificial Serialization | nlohmann/json         | 50           |          |
|                          | our C++26 serializer  | 1800         |   36×    |

Intel Ice Lake:

| Test Instance            | Library              | Speed (MB/s) |  speedup |
|--------------------------|----------------------|--------------|----------|
| Twitter Serialization    | nlohmann/json        | 110          |          |
|                          | our C++26 serializer | 1800         |    16×   |
| Artificial Serialization | nlohmann/json        | 50           |          |
|                          | our C++26 serializer   | 1400         |   28×    |


Our benchmark showed that we are roughly 20× faster than [nlohmann/json](https://github.com/nlohmann/json). This was achieved by combining the new reflection capabilities with [some bit twiddling tricks](https://lemire.me/blog/2024/05/31/quickly-checking-whether-a-string-needs-escaping/) and a [string_builder class](https://github.com/simdjson/experimental_json_builder/blob/main/src/string_builder.hpp) to help minimize the overhead of memory allocations.

Importantly, our implementation requires little code. The C++26 reflection capabilities do all the hard work.
In our benchmarks, we further compare against hand-written functions as well as an 
existing C++ reflection library (reflect-cpp). 
Hand-written code can be slightly faster than our reflection-based code, but the difference is
small (10% to 20%). We can be twice as fast as reflect-cpp although it is also a very fast library.

## Conclusion

One results suggest that C++26 with reflection is a powerful tool that should allow C++ programmers to easily serialize
and deserialize data structures at gigabytes per second.
In the near future, the [simdjson library](https://github.com/simdjson/simdjson) will adopt
reflection for both serialization and deserialization.
Indeed, we expect that this will help users get 
high performance in some important cases with little effort.

## Credit
Joint work with the amazing [Daniel Lemire](https://twitter.com/lemire)!

We are grateful to @the-moisrex for teaching us about tag dispatching and initiating its adoption in the simdson library.

## Appendix: complete code example


```cpp
struct kid {
  int age;
  std::string name;
  std::vector<std::string> toys;
};

/**
 * The following function would print:
 *
 *   I am 12 years old
 *   I have a car
 *   I have a ball
 *   My name is John
 *   My JSON is {"age":12,"name":"John","toys":["car","ball"]}
 *
 */
void demo() {
  simdjson::padded_string json_str =
      R"({"age": 12, "name": "John", "toys": ["car", "ball"]})"_padded;
  simdjson::ondemand::parser parser;
  auto doc = parser.iterate(json_str);
  kid k = doc.get<kid>();
  std::print("I am {} years old\n", k.age);
  for (const auto &toy : k.toys) {
    std::print("I have a {}\n", toy);
  }
  std::print("My name is {}\n", k.name);

  std::print("My JSON is {}\n", simdjson::json_builder::to_json_string(k));
}
```

[^1]: Langdale, Geoff, and Daniel Lemire. "[Parsing gigabytes of JSON per second](https://arxiv.org/abs/1902.08318)" The VLDB Journal 28.6 (2019): 941-960.

[^2]: Keiser, John, and Daniel Lemire. "[On‐demand JSON: A better way to parse documents?](https://arxiv.org/abs/2312.17149)" Software: Practice and Experience 54.6 (2024): 1074-1086.

[^3]: Wyatt Childers, Peter Dimov, Dan Katz, Barry Revzin, Andrew Sutton, Faisal Vali, Daveed Vandevoorde, [Reflection for C++26]([https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2996r3.html#converting-a-struct-to-a-tuple](https://isocpp.org/files/papers/P2996R4.html#converting-a-struct-to-a-tuple)), WG21 P2996R3, 2024-05-22
