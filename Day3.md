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
