# mio
A simple header-only cross-platform C++11 memory mapping library.

## Example
There are three ways to a create a mapping:

- Using the constructor, which throws on failure:
```c++
mio::mmap_source mmap(path, offset, size_to_map);
```

- Using the factory function:
```c++
std::error_code error;
mio::mmap_source mmap = mio::make_mmap_source(path, offset, size_to_map, error);
```

- Using the `map` member function:
```c++
std::error_code error;
mio::mmap_source mmap;
mmap.map(path, offset, size_to_map, error);
```

Moreover, in each case, you can provide either some string type for the file's path, or you can use an existing, valid file handle. However, mio does not check whether the provided file descriptor has the same access permissions as the desired mapping, so the mapping may fail.
```c++
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <mio/mmap.hpp>

int main()
{
    // NOTE: error checking omitted for brevity.
    const int fd = open("file.txt", O_RDONLY);
    mio::mmap_source mmap(fd, 0, mio::map_entire_file);
    // ...
}
```

General usage:
```c++
#include <mio/mmap.hpp>
#include <system_error> // for std::error_code
#include <cstdio> // for std::printf
#include <cassert>

int handle_error(const std::error_code& error)
{
    const auto& errmsg = error.message();
    std::printf("error mapping file: %s, exiting...\n", errmsg.c_str());
    return error.value();
}

int main()
{
    // Read-write memory map the whole file by using `map_entire_file` where the
    // length of the mapping is otherwise expected, with the factory method.
    std::error_code error;
    mio::mmap_sink rw_mmap = mio::make_mmap_sink(
        "file.txt", 0, mio::map_entire_file, error);
    if(error) { return handle_error(error); }

    // Iterate through the mapped region just as if it were any other container, and
    // change each byte's value (since this is a read-write mapping).
    for(auto& b : rw_mmap) {
        b += 10;
    }

    // Or just change one value with the subscript operator.
    const int answer_index = rw_mmap.size() / 2;
    rw_mmap[answer_index] = 42;

    // Don't forget to flush changes to disk, which is NOT done by the destructor for
    // more explicit control of this potentially expensive operation.
    rw_mmap.sync(error);
    if(error) { handle_error(error); }

    // We can then remove the mapping, after which rw_mmap will be in a default
    // constructed state, i.e. this has the same effect as if the destructor had been
    // invoked.
    rw_mmap.unmap();

    // Now create the same mapping, but in read-only mode.
    mio::mmap_source ro_mmap = mio::make_mmap_source(
        "file.txt", 0, mio::map_entire_file, error);
    if(error) { return handle_error(error); }

    const int the_answer_to_everything = ro_mmap[answer_index];
    assert(the_answer_to_everything == 42);
}
```

`mio::basic_mmap` has move-only semantics, but if multiple copies to the same mapping is required, use `mio::basic_shared_mmap` which has `std::shared_ptr` semantics and has the same interface as `mio::basic_mmap`.
```c++
#include <mio/shared_mmap.hpp>

mio::shared_mmap_source shared_mmap1("path", offset, size_to_map);
mio::shared_mmap_source shared_mmap2(std::move(mmap1)); // or use operator=
mio::shared_mmap_source shared_mmap3(std::make_shared<mio::mmap_source>(mmap1)); // or use operator=
mio::shared_mmap_source shared_mmap4;
shared_mmap4.map("path", offset, size_to_map, error);
```

It's possible to define the character type of a byte, though aliases for the most commonly used types are provided:
```c++
using mmap_source = basic_mmap_source<char>;
using ummap_source = basic_mmap_source<unsigned char>;
```
But it may be useful to define your own types, say when using the new `std::byte` type in C++17:
```c++
using mmap_source = mio::basic_mmap_source<std::byte>;
using mmap_sink = mio::basic_mmap_sink<std::byte>;
```

You can query the underlying system's page allocation granularity by invoking `mio::page_size()`, which is located in `mio/page.hpp`.

## Installation
mio is a header-only library, so just copy the contents in `mio/include` into your system wide include path, such as `/usr/include`, or into your project's lib folder.
