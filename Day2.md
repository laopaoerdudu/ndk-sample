
1，编译 java 文件，生成 .class 文件

```
javac HelloWorld.java
```

2，使用 javah 指令编译 `.class` 文件，生成 Java 与 C/C++ 之间进行通信的约定接口，即 `.h` 文件

>`com_wwe_HelloWorld.h`，为了可理解性，我们将其重命名为 HelloWorld.h 文件

```
javah -classpath /Users/david/jni-sample/src -jni com.wwe.HelloWorld

// 由于 -jni 指令在 javah 中是默认选项，因此我们可以忽略掉它
// 在 Dos 中， . 代表当前路径，所以我们可以简单的使用 . 来指定当前路径（/Users/david/jni-sample/src）
// 于是，一个简约的 javah 指令如下所示：

javah -classpath . com.wwe.HelloWorld
```

3，