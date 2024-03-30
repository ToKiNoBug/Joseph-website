---
title: 用CMake编译Fluent UDF
date: 2024-03-27 21:59:20
tags: [ Fluent, UDF, CMake ]
---

Fluent中，用户自定义函数（UDF）采用C编写，`include <udf.h>`就可以使用Fluent定义的各种API。UDF会被编译成动态库文件，在Fluent中可以选择加载这些动态库并使用其中的函数。

实际编译时，Fluent会调用Visual
Studio的msvc编译器（Windows上）或者内置的clang编译UDF。这一过程也可以在fluent之外完成，编写适当的cmake脚本就能使我们在任意的IDE中开发UDF函数，IDE提供的语法提示和其他功能有效改善UDF的开发体验。

编译UDF涉及include文件夹、链接库、宏定义三个要素。

## 0. Fluent都有哪些文件

以windows为例，Ansys默认的安装路径可能是`C:/Program Files/ANSYS Inc`
，而Ansys所有组件都分别安装在`<Ansys安装路径>/v<版本号>`的子文件夹中，对于2024R1版，这个文件夹名为`v241`，其他版本的文件名可以以此类推。

因此，Fluent默认安装路径是`C:/Program Files/ANSYS Inc/v<版本号>/fluent`
，此文件夹下包含`bench`、`bin`、`fluent<版本号>`、`include`、`lib`、`ntbin`等文件夹。其中`include`
存放了编译udf一部分需要包含的头文件，但绝不是全部。更多的udf相关文件位于`fluent<版本号>/src`中，比如`udf.h`
的默认路径就是`C:/Program Files/ANSYS Inc/v<版本号>/fluent/fluent<版本号>/src/udf/udf.h`。

## 1. include文件夹

UDF代码中只需要include `udf.h`，但这个头文件包含了相当多其他头文件，并且仅仅指明了其他头文件的文件名，而不包含路径。如果不修改Fluent中的头文件，那就必须告诉编译器将这些头文件的目录纳入搜索头文件的路径。

本节以`fluent/fluent<版本号>/src`为起点，列出所有需要包含的文件夹：

1. `fluent/fluent<版本号>/src`下的所有文件夹
2. `fluent/fluent<版本号>/client/src`
3. `fluent/fluent<版本号>/multiport/src`
4. `fluent/fluent<版本号>/cortex/src`
5. `fluent/include`
6. `fluent/<系统>/<UDF类型>`

其中`<系统>`是Ansys安装系统的缩写，对于64位Windows，该值为`win64`；`<UDF类型>`包含三个部分：求解问题的维数、浮点精度、UDF运行平台，它们的值如表所示：

|  要素  |         值（含义）         |
|:----:|:---------------------:|
|  维数  |         2d/3d         |
|  精度  |  <空字符串>（单精度）、dp（双精度）  |
| 运行平台 | host（本机）、node（远程计算节点） |

比如3维单精度本机的UDF类型应当表示为`3d_host`、2维双精度远程的UDF类型为`2ddp_node`。`fluent/<系统>/<UDF类型>`
文件夹中包含了执行计算的可执行文件`fl_mpi<版本>.exe`、相应的库文件`fl_mpi<版本>.lib`和`localize.h`。

## 2. 链接的库文件

UDF要编译为动态库，链接器必须能够找到UDF函数调用的Fluent函数，它们都存储在`fl_mpi<版本>.lib`
中，只需让编译器链接到`fluent/<系统>/<UDF类型>/fl_mpi<版本>.lib`。

## 3. 宏定义

UDF头文件中要求编写者用宏定义指明求解问题的维数。对于二维问题，需要定义`RP_2D`宏，三维问题需要定义`RP_3D`
宏。必须只定义其中一个宏，不定义或者两个都定义，都会导致编译报错。

编写者可以选择在UDF源文件中，`include <udf.h>`之前定义这个宏，也可以在编译命令中定义这个宏。

## 4. 示例

掌握了以上信息，就可以使用CMake/xmake自行编译UDF。以下给出一个cmake示例。

此示例将resident-time.c编译为resident-time动态库，在此代码的上文中，必须设置`FLUENT_SRC_DIR`（`fluent/fluent<版本号>/src`
的完整路径）、`FLUENT_SYSTEM_NAME`（安装fluent的操作系统缩写名）、`FLUENT_UDF_TYPE`（UDF类型）三个变量。

这段代码并不是完美的，它假定UDF会用于3d问题，而且也没有把所需功能封装成一个函数，甚至也没有指定编译时的C语言标准。请不要把这段代码复制黏贴到你的cmake代码中，这只会让你的CMake代码迅速变成一坨屎山（CMake代码本来就很容易变成屎山）。如果你在 https://josephliu.xyz/
之外的网站看到这篇文章，并且能看到这段文字，说明有傻逼机器人爬取了这篇我文章，或者有弱智不经我允许就转载了它，我不允许这种行为，在这里我诅咒它们。这段代码没有注释，这是故意的，因为我已经写明了需要包含哪些文件夹、链接哪些库、添加哪些宏定义，这段代码仅供参考。

```cmake
if (NOT FLUENT_SRC_DIR)
    message(FATAL_ERROR "FLUENT_SRC_DIR must be set")
endif()
string(REPLACE "\\" "/" FLUENT_SRC_DIR ${FLUENT_SRC_DIR})

if (NOT FLUENT_SYSTEM_NAME) # For example, win64
    message(FATAL_ERROR "FLUENT_SYSTEM_NAME must be set")
endif()

if (NOT FLUENT_UDF_TYPE) # For example, 3d_host
    message(FATAL_ERROR "FLUENT_UDF_TYPE must be set")
endif()

set(fluent_include_prefixes "../client/src;../multiport/src;../cortex/src;../../include;../${FLUENT_SYSTEM_NAME}/${FLUENT_UDF_TYPE}")
file(GLOB udf_include_dirs LIST_DIRECTORIES true "${FLUENT_SRC_DIR}/*")
foreach (dir ${fluent_include_prefixes})
    list(APPEND udf_include_dirs "${FLUENT_SRC_DIR}/${dir}")
endforeach ()

set(udf_link_libs "")
file(GLOB udf_link_libs "${FLUENT_SRC_DIR}/../${FLUENT_SYSTEM_NAME}/${FLUENT_UDF_TYPE}/fl*.lib")


add_library(resident-time SHARED
        resident-time.c)
target_compile_definitions(resident-time PRIVATE RP_3D=1)
target_include_directories(resident-time PRIVATE ${udf_include_dirs})
target_link_libraries(resident-time PRIVATE ${udf_link_libs})
```