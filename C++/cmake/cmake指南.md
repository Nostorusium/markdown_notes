# CMAKE食用指南

对于编译器而言,它看不到任何工程项目结构.
编译器只是单纯的接受参数并执行命令.
但通过cmake指定各种路径和配置,我们得以利用它建立起不同的结构
VSCODE所提供的实时报错是由C++拓展插件实现的,编译器本身没有这个功能
在建立结构时要记得配置好插件的参数设置,并与cmake的设置匹配.

## 基本

安装cmake之后的步骤:
1. Configure配置,生成cmake的原生工程
   可以使用命令或者根据拓展提示进行项目配置,全自动.
2. Build构建,构建源文件
   根据CMakeLists.txt来编译源文件

```
cmake_minimum_required(VERSION 3.10)
project(test_Project)
ADD_EXECUTABLE(hello main.cpp)
```
这是最简单的三行指令,分别给出最低版本,项目名称与目标文件源文件
其中,project步骤会自动生成两个宏:
```
${PROJECT_SOURCE_DIR}
${PROJECT_NAME}
```
DIR为CMakeLists.txt所在的文件夹路径
后者为project中的名称
add_executable(目标文件 依赖文件)

### SET与FILE

```
set(SRC_LIST a.cpp b.cpp)
```
可以用set指定变量,并可以用${SRC_LIST}把它当做宏使用
set常用于写源代码路径
```
set(SRC_FILES
   ${PROJECT_SOURCE_DIR}/src/main.cpp
   ${PROJECT_SOURCE_DIR}/src/test.cpp
   ${PROJECT_SOURCE_DIR}/src/test.hpp
)
add_executable(${CMAKE_PROJECT_NAME} ${SRC_FILES})
```

若要使用通配符,应使用file

```
file(GLOB SRC_FILES
    "${PROJECT_SOURCE_DIR}/src/*.h"
    "${PROJECT_SOURCE_DIR}/src/*.cpp"
    "${PROJECT_SOURCE_DIR}/src/*.c"
    "${PROJECT_SOURCE_DIR}/src/*.cc"
)
```

### ADD_EXECUTABLE

CMAKE_RUNTIME_OUTPUT_DIRECTORY变量用于存放可执行文件的输出目录
通过set这个变量我们可以改变输出文件的路径

```
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)
add_executable(${CMAKE_PROJECT_NAME} ${SRC_FILES})
```

### 总结

至此我们已经可以完成简单的CMakeList的编写了.

```
cmake_minimum_required(VERSION 3.10)

project(test)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)
set(SRC_FILES
   "${PROJECT_SOURCE_DIR}/src/a.h"
   "${PROJECT_SOURCE_DIR}/src/a.cpp"
)

add_executable(objname ${SRC_FILES})
```

## 库

### lib

```
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)
```
修改CMAKE_LIBRARY_OUTPUT_DIRECTORY变量来改变库的位置


## 预编译头文件

预编译头文件的格式是target_precompile_headers(目标名 可见度控制 头文件)
可见度可以选PUBLIC,PRIVATE,INTERFACE.
以下列为例,注意顺序

```
# 输出文件名
set(OUTPUT_NAME "test")
# 源文件
set(SRC_FILES
   "${PROJECT_SOURCE_DIR}/src/test.cpp"
)
add_executable(${OUTPUT_NAME} ${SRC_FILES})
# 预编译
target_precompile_headers(${OUTPUT_NAME} PUBLIC 
   ${PROJECT_SOURCE_DIR}/src/preCompile.hpp
)
```

## TEMP

```
find_package(myLibrary REQUIRED)
```
使用find_package链接库,REQUIRED表示这个库是必须的

```
target_compile_features(${CMAKE_PROJECT_NAME} PRIVATE cxx_std_17)
```
打开对C++标准的支持

```
option(FEATURE_A "Enable feature A" ON)

if(FEATURE_A)
   add_definitions(-DUSE_FE)
endif()
```
选择性的编译部分源代码