__This project is now in progress. These features might change in future release.__

[![Build Status](https://travis-ci.org/ToruNiina/libasd.svg?branch=master)](https://travis-ci.org/ToruNiina/libasd)

# libasd

A C++ header-only library to read a .asd file generated by high-speed AFM.

libasd has a binding to python3.

## Table of Contents

* [libasd](#libasd)
* [Python usage](#Usage-in-Python3)
* [C++ usage](#Usage-in-C++)
* [FAQ](#FAQ)
* [LICENSE](#Licencing-terms)


## Usage in Python3

__Especially, Python3 bindings are really unstable now.__
__These features might drastically change in future release.__

### Building Python Library

Using CMake and [pybind11](https://github.com/pybind/pybind11).
CMake automatically download pybind11 as a git submodule, so it can be built
by running following scripts.

```sh
$ mkdir build
$ cd build
$ cmake ..
$ make
```

It also build test codes. you can run tests by

```sh
$ make test # optional
```

You may need to add path/to/libasd.so to PYTHONPATH.
It usually generated in `build/python/`. Install script is comming soon.

### Example Code

__Especially, Python3 bindings are really unstable now.__
__The APIs used in this example is likely to be changed.__

```python
import libasd

header = libasd.read_header(file_name = "example.asd", version = 1)
print("image size = {}x{} ".format(header.x_pixel, header.y_pixel))
```

### With NumPy

WIP

## Usage in C++

libasd is a header-only library, so you don't need to build anything if you use
it as a C++ library, not python library.

The only thing you have to do is adding `path/to/libasd` to your include path like

```sh
$ g++ -I/path/to/libasd -std=c++11 your_code_main.cpp
```

Then you can read the data in the way described below.

```cpp
#include <libasd/libasd.hpp>
#include <fstream>
#include <iostream>

int main()
{
    std::ifstream ifs("example.asd");
    const auto data = asd::read_asd<double>(ifs);

    std::cout << "x_pixel = " << data.header.x_pixel << '\n';
    std::cout << "y_pixel = " << data.header.y_pixel << '\n';

    for(auto const& frame : data.frames)
    {
        for(auto const& line : frame)
        {
            for(auto const& pixel : line)
            {
                std::cout << pixel << ','; // height [nm] for topography, ...
            }
            std::cout << '\n';
        }
        std::cout << "\n\n";
    }
    std::cout << std::flush;
    return 0;
}
```

Here you can access each frame, line, and pixel intuitively by using range-based
for loops.

You can set file version and channel as a template parameter.

```cpp
const auto data = asd::read_asd<double, asd::ch<2>, asd::ver<0>>(ifs);
```

By default, these are set as channel 1, version 1 respectively.

You can access to each pixel by index instead of range-based for loops.

```cpp
    const std::size_t x_pixel = data.header.x_pixel;
    const std::size_t y_pixel = data.header.y_pixel;
    for(const auto& frame : data.frames)
    {
        for(std::size_t y=0; y<y_pixel; ++y)
        {
            for(std::size_t x=0; x<x_pixel; ++x)
            {
                std::cout << data[y][x]; // note: not [x][y].
            }
            std::cout << '\n';
        }
        std::cout << "\n\n";
    }
```

Be careful with the order of index. `Frame[y]` returns a `Line` at y.
So first you must specify `y` value of the pixel, not `x`.

## FAQ

### I need only file-header information. Frame data are not needed.

libasd provides `asd::read_header` function (`libasd.read_header` in python).
It reads only file-header information.
You can use this function in the same way as `read_asd`.

```cpp
#include <libasd/libasd.hpp>
#include <fstream>
#include <iostream>

int main()
{
    std::ifstream ifs("example.asd");
    const auto data = asd::read_header(ifs);
    // If you want to set file version information,
    // data = libasd.read_header<asd::ver<2>>("example.asd");
    // file version is set as asd::ver<1> by default.
    return 0;
}
```

```python
import libasd

data = libasd.read_header("example.asd");
# If you want to set file version information,
# data = libasd.read_header("example.asd", 2);
```

Note: It simply ignore the channel information because header format does not
depend on channel number. You can set channel number as unrealistic value here,
but it is not recommended because it is confusing.

### How to access the data in header/frame\_header object?

See documents.

### I don't want to use streams. What can I do?

You can pass a `char const*` to `read_asd` function in exactly the same way as streams.

### How it contains data?

You might think that libasd contains a frame as an array of arrays.

```cpp
typedef std::int16_t            pixel_type;
typedef std::vector<pixel_type> line_type;
typedef std::vector<line_type>  frame_type;
```

You might be afraid of the performance loss that is owing to a cache-miss
while you access to each line. But it is __not__ true.
Although the usage is easy, you don't have to be afraid of the performance cost.
Libasd contains frames as a single array, not an array of arrays.

```cpp
typedef std::int16_t            pixel_type;
typedef std::vector<pixel_type> frame_type;
```

To make the usage easier, libasd provides a proxy class to access each lines.
It wraps frame class and enable you to access a particular line as a container.
In range-based for loops, this proxy classes are used instead of
`std::vector<pixel_type>::(const_)iterator`.

You can use `std::vector<pixel_type>::(const_)iterator` if you want.

```cpp
const auto frame = data.frames.front();
for(auto iter = frame.raw_begin(), iend = frame.raw_end(); iter != iend; ++iter)
{
    std::cerr << *iter << ' ';
}
```

### Can I use my awesome container with libasd?

If you implemented or found a container or an allocator that has a great feature,
you may want to use it with libasd instead of `std::vector<T, std::allocator<T>>`.

In libasd, you can specify the container used in the classes by passing the
specialized `struct container_dispatcher` as a template parameter.

For example, `asd::container::vec` that is used by default is defined as follows.

```cpp
namespace asd {
namespace container {
struct vec
{
    template<typename T>
    struct rebind
    {
        typedef std::vector<T, std::allocator<T>> other;
    };
};

template<typename T, typename Alloc>
struct container_traits<std::vector<T, Alloc>>
{
    using ptr_accessibility = std::true_type;
    using value_type = T;
};

} // container
} // asd
```

libasd uses the container to declare the container in this way.

```cpp
typedef typename vec::template rebind<int>::other int_array;
```

The struct `container_traits` provides a tag to resolve overload of utility
functions. When the container has an interface to access a pointer that points
the first element and the container can be accessed as a traditional C-array,
it become `std::true_type`, otherwise, `std::false_type`.

Additionally, to deal with different interfaces of containers, libasd has
some helper functions. If it is needed(the container has a different interface
from standard containers), you should overload these functions.

```cpp
// example: add overload for std::vector.

namespace asd {
namespace container {

template<typename T, typename Alloc>
inline T const* get_ptr(const std::vector<T, Alloc>& v) noexcept
{
    return v.data();
}
template<typename T, typename Alloc>
inline std::size_t size(const std::vector<T, Alloc>& v) noexcept
{
    return v.size();
}
template<typename T, typename Alloc>
inline void resize(std::vector<T, Alloc>& v, const std::size_t N)
{
    return v.resize(N);
}
template<typename T, typename Alloc>
inline void clear(std::vector<T, Alloc>& v)
{
    return v.clear();
}

} // container
} // asd
```

After implementing and passing these structs and functions(if necessary),
you can use your awesome container/allocator class with libasd.

By default, `asd::container::vec` and `asd::container::deq` are defined in
`libasd/container_dispatcher.hpp`.
They are corresponds to `std::vector` and `std::deque`, respectively.
It should be noted that only randomly accessible containers can be used with
libasd.

Additionally, you can find `asd::container::boost_vec`,
`asd::container::boost_static_vec`, `asd::container::boost_small_vec`
and `asd::container::boost_deq` in the file
`libasd/boost/container_dispatcher.hpp`.
It is not included by default, but you can manually include this file.
Then you can use containers provided by the Boost.Container library if you
already installed it.

## Documents

WIP

## Licensing terms

Author
- Toru Niina

This product is licensed under the terms of the [MIT License](LICENSE).

- Copyright (c) 2017 Toru Niina

All rights reserved.
