#! https://zhuanlan.zhihu.com/p/146434531
# 使用 Unity Build 加速 CMake 编译



## 一、使用效果

![未使用 Unity Build](Accelerate-build-with-unity-build/before.png)

未使用 Unity Build

![使用 Unity Build](Accelerate-build-with-unity-build/after.png)

使用 Unity Build



## 二、原理说明

C/C++ 的编译系统和其他高级语言存在很大的差异。

其他高级语言中，编译单元是**整个module**，即module下所有源码，会在同一个编译任务中执行。

而 C/C++ 中，因为历史遗留问题（早年的硬件内存不支持同时加载整个项目的源码进行编译），编译单元是以**文件**为单位。



即，C/C++ 的编译方式是这样的：

- 对每个 `.c`/`.cc`/`.cxx`/`.cpp` 源文件，启动一个独立的编译器进程进行编译，生成 `.o`/`.obj` 临时文件；
- 编译完成所有源文件后，使用链接器，将所有临时文件合并链接到目标二进制。



以上编译方式存在几个很严重的性能问题：

1. 每个编译单元，都需要独立解析所有包含的头文件:
   1. 如果N个源文件引用到了同一个头文件，则这个头文件需要解析N次（对于`protobuf`这类动辄几千上万行的头文件简直就是鬼故事）；
   2. 如果头文件中有模板(STL/Boost)，则该模板在每个cpp文件中使用时都会做一次实例化，N个源文件中的`std::vector<int>`会实例化N次。
2. 每个源文件独立编译，导致编译优化时只能基于本文件内容进行优化，很难跨编译单元提供代码优化；
3. 基于问题2，C/C++ 跨编译单元的优化只能交给链接器，而链接阶段是单进程，无法并行加速，导致大项目链接极慢。

问题1可以通过并行编译解决，但只是治标不治本，总开销依然有大量的资源浪费；也可通过预编译头PCH解决，与本方法差别见后文。

问题2目前暂无解决手段，新标准的`module`功能有望改善此问题。

问题3可以通过动态库方式降低链接耗时，即拆分为多个二进制目标。（静态库无效，静态库本质上相当于把 .o/.obj 做了一次打包，最后生成二进制时合并到链接器中）



本文提出一个 Unity Build 思路，可以大幅提升编译速度，原理如下：

- 建立临时文件`unity_build.cpp`
- 在该文件中，`#include`包含所有**源代码**文件（不是包含头文件）
- 不编译分散的源代码文件，而是只编译`unity_build.cpp`

该方式规避了问题1中的额外开销，对于越复杂的代码（模板/宏用得多），编译速度提升越快。

同时，对于1.2的模板多次实例化问题，Unity Build 只会实例化一次模板，对问题3的链接也有一定加速。

但考虑到多进程并行编译，并不是把所有代码都打包到同一个`unity_build.cpp`中效果最好。

打包思路可以参考`module`，即将同一个模块的源代码打包为一个文件，这样该模块中大量公用的头文件就可以避免被多次解析。



## 三、与 PCH 区别

1、`pch`不能针对性地对一部分代码生效。比如只想把某三个文件打包编译，`pch`就不适用。如果专门写个`add_library`把他们仨单独打包又太麻烦了，还得配置`target_include_directories`啥的。如果不用`target_xxx`系列指令的话，又不太符合 target-based 的 Modern CMake 风格，少了内味/Taste。

2、这个方法比`pch`更激进，相当于把所有头文件全都塞一起了——用`pch`开发的时候，不可能所有`cpp`里只有一行`#include <stdafx.h>`吧。



## 四、参考代码

```cmake
# Unity Build 函数，将输入的文件列表打包输出为一个 unit_build.cpp文件
# input_src：文件列表，调用时通过字符串传递，如 "${PROJECT_SRC}"。若在.cmake文件中调用，则需要提供绝对路径
# output_file：输出文件，存放在 ${CMAKE_BINARY_DIR}/${output_file}，即build目录
function(UNITY_BUILD output_file input_src)
    file(WRITE ${CMAKE_BINARY_DIR}/${output_file} "//  ${output_file}\n")
    foreach(filename ${input_src})
        file(APPEND ${CMAKE_BINARY_DIR}/${output_file} "#include \"${filename}\"\n")
    endforeach()
    message("Generate unity build file: ${CMAKE_BINARY_DIR}/${output_file}")
endfunction(UNITY_BUILD)


# 使用范例
set(MODULE_SRC
    a.cpp
    b.cpp
    c.cpp
)
UNITY_BUILD(module_unity_build.cpp "${MODULE_SRC}")
add_executable(${PROJECT_NAME}
    main.cpp
    ${CMAKE_BINARY_DIR}/module_unity_build.cpp
)
```

注意：

- 谨慎使用全局变量。不同源文件的全局变量可能存在重名，导致重定义错误。可以考虑把全局变量改为类静态成员。
- 尽量规避宏定义。宏定义为全局最高优先级替换，在 Unity Build 中，可能会破坏其他源文件中的标识符，导致编译错误（说的就是你俩，`windows.h`的`max`和，`xlib.h`的`Status`！）。
