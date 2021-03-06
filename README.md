## WIP fork adding VS2017 support and other portability changes

Microsoft C++ compiler has matured a lot and is now able to tackle codebases
like this one. Most of the changes are related to the lack of some GCC/clang
extensions.

### Important caveats

To compile with VS2017 the new experimental preprocessor must be enabled for
all the compilation units that include libnop headers. This new preprocessor
gives warnings with the built-in C and Windows platform headers, so I recommend
to use libnop external struct/table serializers and concentrate all the code
in a single file, then enabling the new preprocessor just for that one file.

The new preprocessor is enabled with the /experimental:preprocessor command
line option (right click a file or project, go into C/C++ -> Command Line,
input in Additional Options). You must also enable C++17 mode in C/C++ ->
Language (I haven't tested any lower version). Finally a very recent version
of VS2017 is required, I'm using 15.9.6.

### Changes so far

  * Remove use of `...` in switch case
  * Put `[[gnu::used]]` behind a macro
  * Put `__has_cpp_attribute` behind a macro
  * Do not use accessors like &VectorVal[VectorLen] or &ArrayVal[ArrayLen] to
    get a pointer just after the end of the memory block for those types. This
    can trigger debug asserts inside operator[] in some STL implementations.
  * Fix an apparent bug in the writer method for the integral version of
    std::array


# libnop: C++ Native Object Protocols

libnop is a header-only library for serializing and deserializing C++ data
types without external code generators or runtime support libraries. The only
mandatory requirement is a compiler that supports the C++14 standard.

Note: This is not an officially supported Google product at this time.

## Goals

libnop has the following goals:

  * Make simple serialization tasks easy and complex tasks tractable.
  * Remove the need to use code generators and schema files to describe data
    types, formats, and protocols: perform these tasks naturally within the C++
    language.
  * Avoid additional runtime support requirements for serialization.
  * Provide contemporary features such as bidirectional binary compatability,
    data validation, type safety, and type fungibility.
  * Handle intrinsic types, common STL types and containers, and user-defined
    types with a minimum of effort.
  * Produce optimized code that is easy to analyze and profile.
  * Avoid internal dynamic memory allocation when possible.

## Getting Started

Take a look at [Getting Started](docs/getting-started.md) for an introduction to
the library.

## Quick Examples

Here is a quick series of examples to demonstrate how libnop is used. You can
find more examples in the repository under [examples/](examples/).

### Writing STL Containers to a Stream

```C++
#include <iostream>
#include <map>
#include <sstream>
#include <utility>
#include <vector>

#include <nop/serializer.h>
#include <nop/utility/stream_writer.h>

int main(int argc, char** argv) {
  using Writer = nop::StreamWriter<std::stringstream>;
  nop::Serializer<Writer> serializer;

  serializer.Write(std::vector<int>{1, 2, 3, 4});
  serializer.Write(std::vector<std::string>{"foo", "bar", "baz"});

  using MapType =
      std::map<std::uint32_t, std::pair<std::uint64_t, std::string>>;
  serializer.Write(
      MapType{{0, {10, "foo"}}, {1, {20, "bar"}}, {2, {30, "baz"}}});

  const std::string data = serializer.writer().stream().str();
  std::cout << "Wrote " << data.size() << " bytes." << std::endl;
  return 0;
}
```

### Simple User-Defined Types

```C++
#include <cstdint>
#include <iostream>
#include <sstream>
#include <string>
#include <vector>

#include <nop/serializer.h>
#include <nop/structure.h>
#include <nop/utility/stream_writer.h>

namespace example {

struct Person {
  std::string name;
  std::uint32_t age_years;
  std::uint8_t height_inches;
  std::uint16_t weight_pounds;
  NOP_STRUCTURE(Person, name, age_years, height_inches, weight_pounds);
};

}  // namespace example

int main(int argc, char** argv) {
  using Writer = nop::StreamWriter<std::stringstream>;
  nop::Serializer<Writer> serializer;

  serializer.Write(example::Person{"John Doe", 37, 72, 180});
  serializer.Write(std::vector<example::Person>{
      {"John Doe", 37, 72, 180}, {"Jane Doe", 36, 69, 130}});

  const std::string data = serializer.writer().stream().str();
  std::cout << "Wrote " << data.size() << " bytes." << std::endl;
  return 0;
}
```

### More Complex User-Defined Types

```C++
#include <array>
#include <iostream>
#include <sstream>
#include <string>

#include <nop/serializer.h>
#include <nop/structure.h>
#include <nop/utility/stream_writer.h>

namespace example {

// Contrived template type with private members.
template <typename T>
struct UserDefined {
 public:
  UserDefined() = default;
  UserDefined(std::string label, std::vector<T> vector)
      : label_{std::move(label)}, vector_{std::move(vector)} {}

  const std::string label() const { return label_; }
  const std::vector<T>& vector() const { return vector_; }

 private:
  std::string label_;
  std::vector<T> vector_;

  NOP_STRUCTURE(UserDefined, label_, vector_);
};

}  // namespace example

int main(int argc, char** argv) {
  using Writer = nop::StreamWriter<std::stringstream>;
  nop::Serializer<Writer> serializer;

  serializer.Write(example::UserDefined<int>{"ABC", {1, 2, 3, 4, 5}});

  using ArrayType = std::array<example::UserDefined<float>, 2>;
  serializer.Write(
      ArrayType{{{"ABC", {1, 2, 3, 4, 5}}, {"XYZ", {3.14, 2.72, 23.14}}}});

  const std::string data = serializer.writer().stream().str();
  std::cout << "Wrote " << data.size() << " bytes." << std::endl;
  return 0;
}
```
