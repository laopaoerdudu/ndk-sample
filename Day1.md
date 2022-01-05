#### Work flw

- Gradle 调用您的外部构建脚本 CMakeLists.txt。

- CMake 按照构建脚本中的命令将 C++ 源代码文件 native-lib.cpp 编译到共享对象库中，并将其命名为 libnative-lib.so，Gradle 随后会将其打包到应用中。

- 在运行时，应用的 MainActivity 会使用 System.loadLibrary() 加载原生库。现在，应用就可以使用库的原生函数 stringFromJNI() 了。

- MainActivity.onCreate() 调用 stringFromJNI()，后者会返回“Hello from C++”，并使用它更新 TextView。

所有的 C/C++ 源代码文件，必须放在 src/main/cpp 目录下

#### 配置 CMake，添加 NDK API

CMake 构建脚本是一个纯文本文件，您必须将其命名为 CMakeLists.txt，配置的是关于库的一些信息。

如果您在构建脚本中指定 `native-lib` 作为共享库的名称，CMake 就会创建一个名为 `libnative-lib.so` 的文件。

为了让 CMake 能够在编译时找到头文件，您需要向 CMake 构建脚本添加 include_directories() 命令，并指定头文件的路径：

```
add_library(...)

# Specifies a path to native header files.
include_directories(src/main/cpp/include/)
```

通过使用多个 `add_library()` 命令，您可以为 CMake 定义要根据其他源代码文件构建的更多库。

NDK 还以源代码的形式包含一些库，您需要构建这些代码并将其关联到您的原生库。
以下命令可以指示 CMake 将 android_native_app_glue.c（负责管理 NativeActivity 生命周期事件和触摸输入）构建至静态库，并将其与 native-lib 关联：

```
add_library( app-glue
             STATIC
             ${ANDROID_NDK}/sources/android/native_app_glue/android_native_app_glue.c )

# You need to link static libraries against your shared native library.
target_link_libraries( native-lib app-glue ${log-lib} )
```

向 CMake 构建脚本添加 `find_library()` 命令以找到 NDK 库
您可以使用此变量在构建脚本的其他部分引用 NDK 库。以下示例会找到 Android 专有的日志支持库，并将其路径存储在 log-lib 中：

```
// <android/log.h> 包含用于记录到 logcat 的 API。
find_library( log-lib
              log )
```

为了让您的原生库能够调用 log 库中的函数，您需要使用 CMake 构建脚本中的 target_link_libraries() 命令来关联这些库

#### 添加其他预构建库

```
// 由于库已构建，因此您需要使用 IMPORTED 标志指示 CMake 您只想要将此库导入到您的项目中
add_library( imported-lib
             SHARED
             IMPORTED )
```






































