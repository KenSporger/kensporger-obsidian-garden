---
{"dg-publish":true,"permalink":"/计算机科学/C++/JNI/"}
---


# 简介



## NDK

### 是什么

在Android OS上开发应用程序，Google提供了两种开发包：SDK和NDK。

+ SDK：Software Development Kit，Android开发必用的工具集，要求开发者必须使用Java语言进行开发。
+ NDK：[Native Develop Kit](https://developer.android.google.cn/ndk/index.html)，是一组能将C或C++（原生代码）嵌入到 Android 应用中的工具。Android是基于Linux的，其核心库很多都是C/C++编写的，NDK提供了一种使用系统库或自己的C/C++库的方式。一般情况下，是用NDK工具把C/C++编译为.so文件，然后在Java中调用。

> **原生语言**（native language）：区别于托管语言（managed language），两者的界限其实不是绝对的，一般来说，前者通常被静态编译成相应平台上的机器码，执行时直接执行这些机器码，拥有对硬件（CPU、内存等）有较大操作空间；后者不一定被静态编译成相应平台的机器码，执行的方式是由某个环境解释或动态编译，在内存等硬件的操作上由于被“托管”而受到限制（从而降低开发门槛）。像C/C++就属于原生语言，而Java因为有用到虚拟机，被认为是托管语言；而像Go虽然支持指针, 但很巧妙地约束的指针的使用范围，在托管程度上介于两者之间，很难明确判定。

总之，使用NDK的目的在于：

+ 进一步提升设备性能，以降低延迟或运行游戏或物理模拟等计算密集型应用；
+ 重复使用自己或其他开发者的C或C++库。



### 主要组件

NDK包含以下组件：

- **原生共享库**：NDK从C/C++源代码构建这些库或 `.so` 文件。
- **原生静态库**：NDK也可构建静态库或 `.a` 文件。
- **Java原生接口 (JNI)**：JNI 是Java和C++组件用于相互通信的接口。
- **应用二进制接口 (ABI)**：ABI可以非常精确地定义应用的机器代码在运行时应该如何与系统交互。NDK 根据这些定义构建 `.so` 文件。不同的ABI对应不同的架构：NDK为32位ARM、AArch64、x86及x86-64提供ABI支持。

+ **清单**：如果您编写的应用不包含Java组件，必须在[清单](https://developer.android.google.cn/guide/topics/manifest/manifest-intro)中声明 [NativeActivity](https://developer.android.google.cn/reference/android/app/NativeActivity) 类。

![计算机科学/C++/assets/JNI/a6c953cc6d5b8064138c3fa61dd1b26e_MD5.webp](/img/user/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6/C++/assets/JNI/a6c953cc6d5b8064138c3fa61dd1b26e_MD5.webp)



## JNI

### 是什么

Java调用C/C++在Java语言里面本来就有的，并非Android自创的，即JNI。JNI就是Java调用C++等原生语言的规范。当然，一般的Java程序使用的JNI标准可能和Android不一样，Android的JNI更简单。

> Tips: C/C++语言写出来的代码或模块，编译过程当中要依赖当前操作系统环境所提供的一些库函数，并和本地库链接在一起。而且编译后生成的二进制代码只能在本地操作系统环境下运行，因为不同的操作系统环境，有自己的本地库和CPU指令集，而且各个平台对标准C/C++的规范和标准库函数实现方式也有所区别。 因此，使用JNI接口的Java程序，如果想实现跨平台，就必须将C/C++组件的代码在不同操作系统平台下编译出相应的动态库。

### 优点

最重要的好处是它对底层Java VM的实现没有任何限制。因此，Java VM供应商可以在不影响VM的其他部分的情况下添加对JNI的支持。程序员可以编写本机应用程序或库的一个版本，并期望它能够与支持JNI的所有Java VM一起使用。



# 使用



## JNI函数和指针

![计算机科学/C++/assets/JNI/ff1986f6ebd81bac5900d5d2a6462ec3_MD5.gif](/img/user/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6/C++/assets/JNI/ff1986f6ebd81bac5900d5d2a6462ec3_MD5.gif)

JNI接口指针的设计类似于C++的虚函数表，虚拟机可以运行多张函数表。

例如：

```java
jdouble Java_pkg_Cls_f__ILjava_lang_String_2 (JNIEnv *env, jobject obj, jint i, jstring s)
{
     const char *str = (*env)->GetStringUTFChars(env, s, 0); 
     (*env)->ReleaseStringUTFChars(env, s, str); 
     return 10;
}
```

+ env：接口指针；
+ Obj：在本地方法中声明的对象引用；
+ i和s：用于传递的参数；



# 参考链接

[知乎：原生语言和托管语言的本质区别是什么](https://www.zhihu.com/question/294040278/answer/500773991)

[Android JNI(一) NDK与JNI基础](https://www.jianshu.com/p/87ce6f565d37)

[Google官方：NDK Guides](https://developer.android.google.cn/ndk/guides)

[JNI规范](https://blog.caoxudong.info/blog/2017/10/11/jni_functions_note#2.1)









# WPMZ JNI接口添加

```java
// JNIWPMZManager.java
public class JNIWPMZManager implements JNIProguardKeepTag {
  	// to bytes转换
  	public static byte[] ConvertV3MissionToV2(Wayline wayline, WaylineExecuteMissionConfig missionConfig, WaylineLocationCoordinate3D homeLocation) {
        byte[] jni_wayline = wayline.toBytes();
        byte[] jni_waylineMissionConfig = missionConfig.toBytes();
        byte[] jni_homeLocation = homeLocation.toBytes();
        return native_ConvertV3MissionToV2(jni_wayline, jni_waylineMissionConfig, jni_homeLocation);
    }
  
  	// native定义
  	private static native byte[] native_ConvertV3MissionToV2(byte[] wayline, byte[] waylineMissionConfig, byte[] waylineHomeLocation);
}
```

```java
// wpmz_jni_wrapper.cpp
// wrapper层
jbyteArray JNI_ConvertV3MissionToV2_1(JNIEnv *env, jobject obj, jbyteArray java_wayline, jbyteArray java_waylineMissionConfig, jbyteArray jni_waylineHomeLocation) {
    auto shared_java_wayline = wpmzDJIValueHandler(env, java_wayline).DJIValue<Wayline>();
    Wayline& wayline  = *shared_java_wayline;

    auto shared_java_waylineMissionConfig = wpmzDJIValueHandler(env, java_waylineMissionConfig).DJIValue<WaylineExecuteMissionConfig>();
    WaylineExecuteMissionConfig& waylineMissionConfig = *shared_java_waylineMissionConfig;

    auto shared_java_waylineHomeLocation = wpmzDJIValueHandler(env, jni_waylineHomeLocation).DJIValue<WaylineLocationCoordinate3D>();
    WaylineLocationCoordinate3D& waylineHomeLocation = *shared_java_waylineHomeLocation;
    auto convert_result = ConvertV3MissionToV2(wayline, waylineMissionConfig, waylineHomeLocation);
    if (convert_result.empty()) {
        return env->NewByteArray(0);
    }
    jsize str_size = convert_result.size();
    jbyteArray byte_array = env->NewByteArray(str_size);
    env->SetByteArrayRegion(byte_array, 0, str_size, (const jbyte *) convert_result.c_str());
    return byte_array;
}

// 注册
static const JNINativeMethod gJNIWPMZManagerMethods[] = {
  	// [B 代表jbyteArray ()里的是参数列表, 最后一个[B是返回类型
  	{"native_ConvertV3MissionToV2","([B[B)[B",(void *) JNI_ConvertV3MissionToV2},
    {"native_ConvertV3MissionToV2","([B[B[B)[B",(void *) JNI_ConvertV3MissionToV2_1}, // 重载函数名字要区分
};
```

