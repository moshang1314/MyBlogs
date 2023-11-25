# 1 常规使用

[cmake 上](cnblogs.com/tangzhenqiang/p/4136728.html)

> ```cmake
> # 指定cmake的最低要求版本
> cmake_minimum_required(VERSION 3.10)
> #定义当前工程名字
> project(hello)
> # 指定C++标准
> set(CMAKE_CXX_STANDARD 11)
> # 设置debug模式，如果没有设置将不能调试断点
> set(CMAKE_BUILD_TYPE "Debug")
> # 配置
> set(CMAKE_CXX_FLAGs ${CMAKE_CXX_FLAGS} -g)
> # 指定输出路径
> set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
> # 搜索源文件, ${PROJECT_SOURCE}为cmake命令后面传入的CMakeLists.txt的路径
> aux_source_directory(${PROJECT_SOURCE_DIR}/src SRC_LIST)
> # file(GLOB SRC_LIST ${PROJECT_SOURCE_DIR}/src/*.cpp)
> # file(GLOB_RECURSE ${PROJECT_SOURCE_DIR}/src/*cpp)
> # 指定头文件搜索路径
> include_directories(${PROJECT_SOURCE_DIR}/include)
> # 添加可执行文件
> add_executable(app ${SRC_LIST})
> ```

> ![image-20230601100434747](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230601100434747.png)
>
> **制作静态库：**
>
> ```cmake
> cmake_minimum_required(VERSION 3.10)
> project(music)
> include_directories(${PROJECT_SOURCE_DIR}/include)
> file(GLOB SRC_LIST ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp)
> # 设置库生成路径
> set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)
> add_library(music STATIC ${SRC_LIST})
> ```
>
> **制作动态库：**
>
> ```cmake
> cmake_minimum_required(VERSION 3.10)
> project(music)
> include_directories(${PROJECT_SOURCE_DIR}/include)
> file(GLOB SRC_LIST ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp)
> # 设置库生成路径
> set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)
> add_library(music SHARED ${SRC_LIST})
> ```
>
> **链接静态库：**
>
> ```cmake
> cmake_minimum_required(VERSION 3.10)
> project(lqs)
> # 搜索指定目录下源文件
> file(GLOB SRC_LIST ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp)
> # 包含头文件路径
> include_directories(${PROJECT_SOURCE_DIR}/include)
> # 包含静态库路径
> link_directories(${PROJECT_SOURCE_DIR}/lib)
> # 链接静态库
> link_libraries(music)
> add_executable(app main.cpp)
> ```
>
> **链接动态库：**
>
> ```cmake
> cmake_minimum_required(VERSION 3.10)
> project(test)
> include_directories(${PROJECT_SOURCE_DIR}/include)
> file(GLOB SRC_LIST ${CMAKE_CURRENT_SOURCE_DIR}/*.cpp)
> # 指定要链接的库的路径
> link_directories(${PROJECT_SOURCE_DIR}/lib)
> add_executable(app ${SRC_LIST})
> # 指定要链接的动态库（target_link_libraries既可以链接动态库也可以链接静态库）
> target_link_libraries(app music)
> ```
>
> ([CMake 保姆级教程（下） | 爱编程的大丙 (subingwen.cn)](https://subingwen.cn/cmake/CMake-advanced/))

> **嵌套Cmake**
>
> > 众所周知，Linux 的目录是树状结构，所以嵌套的 CMake 也是一个树状结构，最顶层的 CMakeLists.txt 是根节点，其次都是子节点。因此，我们需要了解一些关于 CMakeLists.txt 文件变量作用域的一些信息：
> >
> > * 根节点 CMakeLists.txt 中的变量全局有效
> > * 父节点 CMakeLists.txt 中的变量可以在子节点中使用
> > * 子节点 CMakeLists.txt 中的变量只能在当前节点中使用
>
> ```shell
> $ tree -L 2
> .
> ├── calc
> │   ├── CMakeFiles
> │   ├── cmake_install.cmake
> │   └── Makefile
> ├── CMakeCache.txt
> ├── CMakeFiles
> │   ├── 3.10.2
> │   ├── cmake.check_cache
> │   ├── CMakeDirectoryInformation.cmake
> │   ├── CMakeOutput.log
> │   ├── CMakeTmp
> │   ├── feature_tests.bin
> │   ├── feature_tests.c
> │   ├── feature_tests.cxx
> │   ├── Makefile2
> │   ├── Makefile.cmake
> │   ├── progress.marks
> │   └── TargetDirectories.txt
> ├── cmake_install.cmake
> ├── Makefile
> ├── sort
> │   ├── CMakeFiles
> │   ├── cmake_install.cmake
> │   └── Makefile
> ├── test1
> │   ├── CMakeFiles
> │   ├── cmake_install.cmake
> │   └── Makefile
> └── test2
>     ├── CMakeFiles
>     ├── cmake_install.cmake
>     └── Makefile
> 
> 11 directories, 21 files
> ```
>
> * include 目录：头文件目录
> * calc 目录：目录中的四个源文件对应的加、减、乘、除算法
>   * 对应的头文件是 include 中的 calc.h
> * sort 目录 ：目录中的两个源文件对应的是插入排序和选择排序算法
>   * 对应的头文件是 include 中的 sort.h
> * test1 目录：测试目录，对加、减、乘、除算法进行测试
> * test2 目录：测试目录，对排序算法进行测试
>
> 总目录下的`CMakeLists.txt`：
>
> ```cmake
> cmake_minimum_required(VERSION 3.10)
> project(test)
> # 定义变量
> # 库的生成路径
> set(LIB_PATH ${CMAKE_CURRENT_SOURCE_DIR}/lib)
> # 测试程序生成的路径
> set(EXEC_PATH ${CMAKE_CURRENT_SOURCE_DIR}/bin)
> # 头文件目录
> set(HEAD_PATH ${CMAKE_CURRENT_SOURCE_DIR}/include)
> # 库的名称
> set(CALC_LIB calc)
> set(SORT_LIB sort)
> #可执行程序的名称
> set(APP_NAME_1 test1)
> set(APP_NAME_2 test2)
> # 添加子目录
> add_subdirectory(calc)
> add_subdirectory(sort)
> add_subdirectory(test1)
> add_subdirectory(test2)
> ```
>
> `calc`目录下的`CMakeLists.txt`用以生成算术动态库：
>
> ```cmake
> cmake_minimum_required(VERSION 3.10)
> project(CALCLIB)
> message("-------------------\n" ${CMAKE_CURRENT_SOURCE_DIR})
> aux_source_directory(./ SRC)
> include_directories(${HEAD_PATH})
> set(LIBRARY_OUTPUT_PATH ${LIB_PATH})
> add_library(${CALC_LIB} SHARED ${SRC})
> ```
>
> `sort`目录下的`CMakeLists.txt`用以生成排序算法动态库：
>
> ```cmake
> project(SORTLIB)
> aux_source_directory(./ SRC)
> include_directories(${HEAD_PATH})
> set(LIBRARY_OUTPUT_PATH ${LIB_PATH})
> add_library(${SORT_LIB} SHARED ${SRC})
> ```
>
> `test1`目录下的`CMakeLists.txt`用以生成可执行程序`test1`：
>
> ```cmake
> project(CALCTEST)
> aux_source_directory(./ SRC)
> include_directories(${HEAD_PATH})
> link_directories(${LIB_PATH})
> set(EXECUTABLE_OUTPUT_PATH ${EXEC_PATH})
> add_executable(${APP_NAME_1} ${SRC})
> target_link_libraries(${APP_NAME_1} ${CALC_LIB})
> ```
>
> `test2`目录下的`CMakeLists.txt`用以生成可执行程序`test2`：
>
> ```cmake
> cmake_minimum_required(VERSION 3.0)
> project(SORTTEST)
> aux_source_directory(./ SRC)
> include_directories(${HEAD_PATH})
> set(EXECUTABLE_OUTPUT_PATH ${EXEC_PATH})
> link_directories(${LIB_PATH})
> add_executable(${APP_NAME_2} ${SRC})
> target_link_libraries(${APP_NAME_2} ${SORT_LIB})
> ```



