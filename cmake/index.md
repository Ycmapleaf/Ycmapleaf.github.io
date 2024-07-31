# Cmake


## CMake-Install
[CMake/files](https://cmake.org/files/)
```shell
wget https://cmake.org/files/v3.24/cmake-3.24.0-linux-x86_64.sh
sh cmake-3.24.0-linux-x86_64.sh --prefix=/usr/local --exclude-subdir
```
或
```shell
wget https://cmake.org/files/v3.24/cmake-3.24.0-linux-x86_64.tar.gz
export PATH=/root/cmake-3.24.0-linux-x86_64/bin:$PATH
```

## CMake
### config

| 参数  | 含义                                     |
| --- | -------------------------------------- |
| -S  | 指定源文件根目录，必须包含一个CMakeLists.txt文件        |
| -B  | 指定构建目录，构建生成的中间文件和目标文件的生成路径             |
| -D  | 指定变量，格式为-D `<var>=<value>`，-D后面的空格可以省略 |
### build
`cmake --build [<dir> | --preset <preset>]`

| 参数                       | 含义                     |
| ------------------------ | ---------------------- |
| --target                 | 指定构建目标代替默认的构建目标，可以指定多个 |
| --parallel/-j [`<jobs>`] | 指定构建目标时使用的进程数          |

## CMakeLists.txt
### Attribute
#### CMAKE_CURRENT_LIST_FILE
当前正在处理的 CMakeLists.txt 文件的完整路径

结果：/root/Cache-Management-ycy/main-project/CMakeLists.txt

### Operation
#### get_filename_component
`get_filename_component` 函数用于从给定的文件路径中提取特定的部分。

```cmake
get_filename_component(MLIR_INSTALL_PREFIX "${CMAKE_CURRENT_LIST_FILE}" PATH)
```

获取当前 CMakeLists.txt 文件的目录路径，并将其存储在 `MLIR_INSTALL_PREFIX` 变量中。

结果：/root/Cache-Management-ycy/main-project

#### add_executable
通过`add_executable`命令来往构建系统中添加一个可执行构建目标，同样需要指定编译需要的源文件。

```cmake
add_executable(demo main.cpp)
```

#### target_link_libraries
`target_link_libraries`命令来声明构建此可执行文件需要链接的库。

```cmake
target_link_libraries(demo MLIRIR)
```

#### get_property
`get_property` 函数用于查询并获取全局或局部属性值。
```cmake
get_property(dialect_libs GLOBAL PROPERTY MLIR_DIALECT_LIBS)
```

#### pkg_check_modules
`pkg_check_modules` 是 CMake 中的一个宏，用于查询由 `pkg-config` 管理的库，可以自动找到这些库的编译和链接标志。

指定一个变量名来存储查询结果，以及库的名称。
```cmake
pkg_check_modules(<variable> <module> [<module>...])

pkg_check_modules(PNG libpng)
```

## CMake-Tools
cmake -G Ninja .. \
    -DLLVM_ENABLE_ASSERTIONS=ON \
    -DCMAKE_BUILD_TYPE=RELEASE\











