#### JavaVM / JNIEnv

- 一个 JNIEnv 指针仅在其相关联的线程中有效。不能将这个指针从一个线程中传递给另一个线程，或者在多线程中缓存和使用它。

- Java 虚拟机在同一个线程传递给本地方法相同的 JNIEnv 指针，但是从不同线程中调用本地方法时传递的是不同的 JNIEnv 指针。
但是这些 JNIEnv 指针的方法都是同一份。

- JNIEnv 无法被线程共享

```
// 获取 JNIEnv 有几种方式：
JNIEnv *env;
(*jvm)->AttachCurrentThread(jvm, (void **)&env, NULL); 


(*jvm)->GetEnv((void **) env, JNI_VERSION_1_6);
```

当从 Java 层传递一个字符串过来之后，它的类型是 `jstring`。如果，我们想打印一下这个字符串，那么可以使用 `GetStringUTFChars` 这个函数，它会返回一个 `const char *` 类型（编码是 MUTF 类型，具体看参考文档 JNI TIPS），
我们就可以使用 printf 来打印它了。为啥要是 const 的呢？是因为 Java 中的 string 是不可变的，不允许更改。

还有一个 `GetStringChars` 函数，它返回 `jchar*` 类型，我们不能直接打印。
Java 中的字符都是 Unicode 编码，显然这个 jchar * 指向的是 unicode 编码的字符串。

```
bool x ;
const char *chars = env->GetStringUTFChars(hello, &x);

if(x)

// =========
str = (*env)->GetStringUTFChars(env, prompt, NULL);

// 避免内存泄漏
(*env)->ReleaseStringUTFChars(env, prompt, str);

```

####   JNI 访问 Java 的字段与方法

JNIEnv 提供了一个 FindClass 方法来寻找某一个类，这个里面的逻辑其实就是 ClassLoader 的逻辑。这一套方法有点像 Java 里面的反射的使用方式。

```
jmethodID mid;
jclass runnableIntf =(*env)->FindClass(env, "java/lang/Runnable"); 
mid = (*env)->GetMethodID(env, runnableIntf, "run", "()V"); 
``` 

FindClass / GetMethodID / GetFieldID 这些都会有性能问题，所以有一个小技巧就是将这些结果缓存起来，后面直接使用缓存的字段就好了。

```
jmethodID MID_InstanceMethodCall_callback; 

JNIEXPORT void JNICALL Java_InstanceMethodCall_initIDs(JNIEnv *env, jclass cls) {
    MID_InstanceMethodCall_callback = (*env)->GetMethodID(env, cls, "callback", "()V"); 
}
```

一个典型的虚拟机实现执行 Java/native 调用比执行 Java/Java 调用大概慢两到三倍。有了 method id 与 field id 就可以尝试访问方法与字段了，常用的方法：

```
CallVoidMethod
CallVoidMethodV
CallVoidMethodA
```

FindClass 返回的是一个局部引用，而局部引用在函数返回后会被 JVM 自动回收。查了一些资料以及经过自己的测试，发现 GetMethodID/GetFieldID 返回的是弱全局引用。

我写了这样的一个 Demo：

```
// 缓存一个 RefActivity 的引用以便后续使用
// cache_class 是一个指针，第一次被赋值后，指向 RefActivity 的 class 的一个局部引用，这个引用在方法结束之后会被回收
// 但是 cache_class 的值仍然指向那个局部引用的地址（地址里面没有东西了），所以导致 cache_class 成了一个野指针。
// 该方法第一次执行时没有问题，第二次执行时就会报错：
jclass cache_class;

extern "C"
JNIEXPORT void JNICALL
Java_com_aprz_mytestdemo_jni_RefActivity_cache(JNIEnv *env, jobject thiz) {
    if (cache_class == nullptr) {
        cache_class = env->FindClass("com/aprz/mytestdemo/jni/RefActivity");
    }
    LOGE("cache_class = %p", cache_class);
    jmethodID just_print = env->GetMethodID(cache_class, "justPrint", "()V");
    if (just_print) {
        env->CallVoidMethodA(thiz, just_print, {});
    }
}
```

同样的方式缓存 method id ：

```
jmethodID cache_method_id;

extern "C"
JNIEXPORT void JNICALL
Java_com_aprz_mytestdemo_jni_RefActivity_cacheMethod(JNIEnv *env, jobject thiz) {
    if (cache_method_id == nullptr) {
        // 由于 GetMethodID 返回的是弱全局引用，那么只要有别的地方引用它，它就不会被回收，
        // 所以这样使用没有问题。多次运行是 ok 的。
        cache_method_id = env->GetMethodID(env->GetObjectClass(thiz), "justPrint", "()V");
    }
    if (cache_method_id) {
        env->CallVoidMethodA(thiz, cache_method_id, {});
    }
}
```

局部引用在方法执行完之后会被回收，但是局部引用能够删除的就尽量立即删除

```
void tools(JNIEnv *env, jobject thiz) {
    jclass tmp_class = env->GetObjectClass(thiz);
    jmethodID tmp_method = env->GetMethodID(tmp_class, "justPrint", "()V");
    
    // JNIEnv 提供了一个 DeleteLocalRef 方法用来删除局部引用
    // 手动释放内存
    env->DeleteLocalRef(tmp_class);
}
```

同样的，全局引用与弱全局引用也有对应的删除方法：

```
env->DeleteGlobalRef()
env->DeleteWeakGlobalRef()
```

#### 引用的比较

给定两个引用（不管是全局、局部还是弱全局引用），我们只需要调用 IsSameObject 来判断它们两个是否指向相同的对象。

```
// 例如：
// 如果 obj1 和 obj2 指向相同的对象，则返回 JNI_TRUE（或者1），否则返回 JNI_FALSE（或者 0）。
// 但需要注意的是，如果比较的是弱全局引用，那么 true 表示的是该引用被回收了。
（*``env``)->IsSameObject(env, obj1, obj2)
```

jvalue 是一个联合类型，它里面储存的是真正的参数值

```
typedef union jvalue {
    jboolean    z;
    jbyte       b;
    jchar       c;
    jshort      s;
    jint        i;
    jlong       j;
    jfloat      f;
    jdouble     d;
    jobject     l;
} jvalue;
```

#### 异常处理

当 JNI 调用 Java 方法时，Java 方法有可能会异常，那么我们需要做对应的处理
因为出了异常之后，JNI 后面的 code logic 而是会继续执行，看下面的一个例子：

```
extern "C"
JNIEXPORT void JNICALL
Java_com_aprz_mytestdemo_jni_ExceptionActivity_toNative(JNIEnv *env, jobject thiz) {
    jclass exception_activity_class = env->GetObjectClass(thiz);
    jmethodID do_something_method_id = env->GetMethodID(exception_activity_class, "doSomething",
                                                        "()V");
    if (do_something_method_id) {
        // 这里出了异常，doSomething 是一个 java 方法，它会抛出一个空指针异常
        // 但是 LOGE 这行代码还是会调用
        env->CallVoidMethod(thiz, do_something_method_id);
    }

    LOGE("i will print even exception occurred!!!!");

    env->DeleteLocalRef(exception_activity_class);
}
```

所以我们需要一种手段来检测异常是否出现，从而做对应的处理。

```
// ExceptionOccurred 会检测到这个异常
// ExceptionDescribe 会输出一个关于这个异常的描述性信息
// ExceptionClear 方法清除这个异常。
// 还有一个 ExceptionCheck 方法，它的功能与 ExceptionOccurred 类似，但是它不会产生一个局部引用，所以使用起来比较方便。
// 异常检查的代码写起来很麻烦，但是确实是必要的，特别是你申请了资源的时候，出了异常必须要释放才行。
exc = (*env)->ExceptionOccurred(env); 
if (exc) {
    (*env)->ExceptionDescribe(env); 
    (*env)->ExceptionClear(env); 
}
```

#### JNI_Onload

JNI 提供了注册函数的两种方式，一种是静态注册，第二种是动态注册（RegisterNatives），
动态注册的时机通常就是在 JNI_OnLoad 中，动态注册相比较于静态注册的好处就是可以批量注册，而且还不会暴露出注册的函数。
看了一下源码流程，其实静态注册也是走的动态注册的流程。当一个类被加载的时候，会去设置

####  Hook JN I调用  

FindClass 走的是 classLoader 加载类的流程，我们知道，每个 classLoader 都会指定 path 用来加载 dex 文件，
那么有了类的描述符之后，就会挨个的去找，从而获取对应的 DexFile 文件。如果 classLoader 为 null 的话，就会去系统启动路径去找。

找到 DexFile 之后，会创建一个 kclass 对象，就是 native 层的 Class。
然后设置各种信息：类索引号，静态成员变量和实例成员变量，函数索引号等。
创建 class 的时候，会加载类的各个成员，比如类方法，这个时候就会创建 ArtField 与 ArtMethod 等。

对于 ArtMethod，会去 LinkCode，这个顾名思义，就是链接该方法对应的指令了。
如果这个方法是一个 native 方法，就会去注册这个方法，就像动态注册一样，其实就是设置了 entry_point_from_jni_ 的值，
只不过它设置的是一个 stub，等到真正调用的时候才会去 dlsym 查询 so 的函数地址，然后替换成真的，这种模式很常用。

所以，FindClass 其实就是加载了一个类，创建了对应 Class 对象，以及为每个类方法创建了“Stub”对象，方法真正被调用的时候才会创建真的。

```
class ArtMethod {
…………
protect:
HeapReference declaring_class_;
HeapReference> dex_cache_resolved_methods_;
HeapReference> dex_cache_resolved_types_;
uint32_t access_flags_;
uint32_t dex_code_item_offset_;
uint32_t dex_method_index_;
uint32_t method_index_;
struct PACKED(4) PtrSizedFields {
void* entry_point_from_interpreter_;
void* entry_point_from_jni_;
void* entry_point_from_quick_compiled_code_;
#if defined(ART_USE_PORTABLE_COMPILER)
void* entry_point_from_portable_compiled_code_;
#endif
} ptr_sized_fields_;
static GcRoot java_lang_reflect_ArtMethod_;
｝
……
```

存在 entry_point_from_jni_，entry_point_from_interpreter_ 这些字段，这些字段其实就是方法的指令入口地址：

entry_point_from_jni_：如果该方法是一个 native 方法，那么这个地址就是加载进来的 so 中的方法的地址。

entry_point_from_interpreter_：这个是使用解释器执行时，指令的地址。

entry_point_from_quick_compiled_code_：这个是非解释器执行时的指令地址（就是 aot 模式）。

了解了上面的知识，我们再来看 CallVoidMethod，有了 method id，就相当于有了 ArtMethod，
而 ArtMethod 里面又有方法对应的指令入口，直接跳转过去执行就好了，当然不像我说的这么简单，还要处理参数，栈帧等。

Java 方法对应的 jni 函数的地址在 entry_point_from_jni_ 这个里面，那么我们只需要替换这个值为我们写的方法的地址，就可以监控方法的执行了。
实现这个功能的难点在于，如何找到 ArtMethod 里面的 entry_point_from_jni_ 这个字段。

因为每个版本，每个 rom 都可能不一样，所以有一种取巧的方式是写两个相邻的 native method：

```
external fun x1()
external fun x2()
```

然后，获取到他们的 jmethodid：

```
jclass entry_point_activity = env->GetObjectClass(thiz);

// jmethodI D实际上就是 ArtMethod 的指针，所以，我们就获取到了这两个对象的地址。
jmethodID x1 = env->GetMethodID(entry_point_activity, "x1", "()V");
jmethodID x2 = env->GetMethodID(entry_point_activity, "x2", "()V");
```

对象里面储存的是各个字段，我们只需要从对象的地址开始往后遍历，总能走到 entry_point_from_jni_ 这个字段，
然后跟我们声明的 JNI 函数比较一下即可：

```
intptr_t offsetToFind =
        reinterpret_cast<intptr_t >(x1) - reinterpret_cast<intptr_t >(x2);

if (offsetToFind < 0) {
    offsetToFind = -offsetToFind;
}
if (offsetToFind > 128) {
    offsetToFind = 128;
}

uintptr_t jni_entry_point_offset = 0;
for (uintptr_t curOffset = 0; curOffset < offsetToFind; curOffset += sizeof(void *)) {
    uintptr_t curAddr = reinterpret_cast<uintptr_t >(x1) + curOffset;
    if ((*reinterpret_cast<void **>(curAddr)) ==
        Java_com_aprz_mytestdemo_jni_EntryPointsActivity_x1) {
        jni_entry_point_offset = curOffset;
        break;
    }
}
```

需要注意的是，这种 ArtMethod 计算方法只在 11以下 管用，11及以上 ArtMethod 的获取需要使用其他的方式，这个可以去翻翻源码来解决！

#### Reference

[听说这样学JNI，效果不是一般的好](https://mp.weixin.qq.com/s/8ylMbdcIBf4XM7ShyP3a4g)



























