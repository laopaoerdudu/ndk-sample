#### Background

```
// AndroidJni.java
package com.wwe

public class AndroidJni {
    static{
        System.loadLibrary("main");
    }
    public native void dynamicLog();
    public native void staticLog();
}
```

C++ 代码如下：

```
#include <jni.h>

#define LOG_TAG "main.cpp"

#include "mylog.h"

static void nativeDynamicLog(JNIEnv *evn, jobject obj){

    LOGE("hell main");
}

JNIEXPORT void JNICALL Java_com_wwe_AndroidJni_staticLog (JNIEnv *env, jobject obj)
{
    LOGE("static register log ");

}

JNINativeMethod nativeMethod[] = {{"dynamicLog", "()V", (void*)nativeDynamicLog},};

JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *jvm, void *reserved) {

    JNIEnv *env;
    if (jvm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {

        return -1;
    }
    LOGE("JNI_OnLoad comming");
    jclass clz = env->FindClass("com/wwe/AndroidJni");

    env->RegisterNatives(clz, nativeMethod, sizeof(nativeMethod)/sizeof(nativeMethod[0]));

    return JNI_VERSION_1_4;
}
```

#### 静态注册 native 方法

Java 给我们提供了 `javah` 的工具帮助生成相应的头文件，生成了对应的JNI函数。
生成的 JNI 函数包含两个固定的参数变量，分别是 `JNIEnv` 和 `jobject`，jobject 就是当前与之链接的 native 方法隶属的类对象(类似于Java中的this)。
这两个变量都是 Java 虚拟机生成并在调用时传递进来的。

我们在开发的时候直接 copy 过去就可以了。
进入项目的目录 `app/build/intermediates/classes/debug` 下，运行如下命令：

(AndroidJni.java)
(这里 -d 指定生成 .h 文件存放的目录(如果没有就会自动创建)

```
javah -d jni com.wwe.AndroidJni
```

总结：

静态注册 native 方法的过程，就是 Java 层声明的 native 方法和 JNI 函数是一一对应的，
那么有没有方法让 Java 层的 native 方法和任意的 JNI 函数链接起来，当然是可以的，这就得使用动态注册的方法。接下来就看看如何实现动态注册的。

#### JNI_OnLoad 函数 （动态注册）

当我们使用 `System.loadLibarary()` 方法加载 so 库的时候，Java 虚拟机就会找到 JNI_OnLoad 函数并调用该函数，
因此可以在该函数中做一些初始化的动作，其实这个函数就是相当于 Activity 中的 onCreate() 方法。
该函数前面有三个关键字，分别是 `JNIEXPORT`、`JNICALL` 和 `jint`，其中 `JNIEXPORT` 和 `JNICALL` 是两个宏定义，用于指定该函数是 JNI 函数。
jint 是 JNI 定义的数据类型，因为 Java 层和 C/C++ 的数据类型或者对象不能直接相互的引用或者使用，JNI 层定义了自己的数据类型，用于衔接 Java 层和 JNI 层，
这里的 jint 对应 Java 的 int 数据类型，该函数返回的 int 表示当前使用的 JNI 的版本，其实类似于 Android 系统的 API 版本一样，
不同的 JNI 版本中定义的一些不同的 JNI 函数。该函数会有两个参数，其中 `*jvm` 为 Java 虚拟机实例，JavaVM 结构体定义了以下函数：

```
DestroyJavaVM
AttachCurrentThread
DetachCurrentThread
GetEnv // 使用了 GetEnv 函数获取 JNIEnv 变量
```

JNI_OnLoad 函数中有如下代码：
其实 JNIEnv 结构体是指向一个函数表的，该函数表指向了对应的 JNI 函数，我们通过调用这些 JNI 函数实现 JNI 编程

```
JNIEnv *env;
if (jvm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {
    return -1;
}
```

上面介绍了如何获取 JNIEnv 结构体指针，得到这个结构体指针后我们就可以调用 JNIEnv 中的 `RegisterNatives(...)` 函数完成动态注册 native 方法了。该方法如下：

```
jint RegisterNatives(jclass clazz, const JNINativeMethod* methods, jint nMethods)
```

- 第一个参数 `clazz` 是 Java 层指包含 native 方法的对象

```
// 获取 class 对象
jclass clz = env->FindClass("com/wwe/AndroidJni");
```

- 第二个参数 `methods` 是 JNINativeMethod 结构体指针

```
// 它的定义如下
typedef struct {
    const char* name; // Java 层 native 方法的名字
    const char* signature; // Java 层 native 方法的描述符
    void* fnPtr; // 对应 JNI 函数的指针
} JNINativeMethod;
```

- 第三个参数 `nMethods` 为注册 native 方法的数量。

>一般会动态注册多个 native 方法，首先会定义一个 `JNINativeMethod` 数组，然后将该数组指针作为 `RegisterNative(...)` 函数的参数传入，
>所以这里定义了如下的 `JNINativeMethod` 数组：

```
// dynamicLog 是在 AndroidJni.java 定义的
// nativeDynamicLog 是在 C++ 中定义的
JNINativeMethod nativeMethod[] = {{"dynamicLog", "()V", (void*)nativeDynamicLog}};
```
  
最后调用 `RegisterNative(...)` 函数完成动态注册：

```
env->RegisterNatives(clz, nativeMethod, sizeof(nativeMethod)/sizeof(nativeMethod[0]));
```

#### JNIEnv 结构体

JNIEnv 这个结构体，老厉害了，指向一个函数表，该函数表指向一系列的 JNI 函数，
我们通过调用这些 JNI 函数可以实现与 Java 层的交互，这里简单的看看几个定义的函数：
更多的函数大家还是看看 API 文档吧！

```
..........
// GetFieldID() 函数是获取 Java 对象中某个变量的 ID
jfieldID GetFieldID(jclass clazz, const char* name, const char* sig)

// GetBooleanField() 函数是根据变量的 ID 获取数据类型为 Boolean 的变量
jboolean GetBooleanField(jobject obj, jfieldID fieldID)

jmethodID GetMethodID(jclass clazz, const char* name, const char* sig)

// CallVoidMethod()根据 methodID 调用对应对象中的方法，并且该方法的返回值为 Void 类型。
CallVoidMethod(jobject obj, jmethodID methodID, ...)

CallBooleanMethod(jobject obj, jmethodID methodID, ...)
..........
```

JNI 也定义了一些引用类型以便 JNI 层调用，具体的引用类型如下：

```
jobject                     (all Java objects)
|
|-- jclass                  (java.lang.Class objects)
|-- jstring                 (java.lang.String objects)
|-- jarray                  (array)
|     |--jobjectArray       (object arrays)
|     |--jbooleanArray      (boolean arrays)
|     |--jbyteArray         (byte arrays)
|     |--jcharArray         (char arrays)
|     |--jshortArray        (short arrays)
|     |--jintArray          (int arrays)
|     |--jlongArray         (long arrays)
|     |--jfloatArray        (float arrays)
|     |--jdoubleArray       (double arrays)
|
|--jthrowable
```

当需要调用 Java 中的某个方法的时候我们首先要获取它的 ID，根据 ID 调用 JNI 函数获取该方法，变量的获取过程也是同样的过程，
这些 ID 的结构体定义如下：

```
struct _jfieldID;              /* opaque structure */ 
typedef struct _jfieldID *jfieldID;   /* field IDs */ 

struct _jmethodID;              /* opaque structure */ 
typedef struct _jmethodID *jmethodID; /* method IDs */ 
```



