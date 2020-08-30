FRIDA hook Native基础
=====================

## 0. 致谢

以下内容全部来源于 @**r0ysue**

![肉丝的星球-大家一起来玩](../1_environment_preparation/static/r0ysue.jpg)

[相关内容全部上传百度网盘 提取码：ljt1](https://pan.baidu.com/s/1__B7dnZ2xKBdXTNE_sklZg)

[大黑客sakura的同一课程的学习笔记](https://eternalsakura13.com/2020/07/04/frida/

)

## 1. NDK开发入门

完整项目在 1_native_tutorial.7z

- 相关链接
    - [NDK与JNI基础](https://www.jianshu.com/p/87ce6f565d37)
    - [JNI基础](https://www.jianshu.com/p/fde40a8d80d3)
    - [NDK官网](https://developer.android.google.cn/ndk/index.html)

JNI是做了二进制兼容的，所以C代码编译后可以在任意平台中运行。

### 1.1 新建一个native项目

- Android Studio 新建native项目
    1. 开启Android Studio
    2. Start a new Android Studio project
    3. Native C++
    4. 设置项目名和路径 Language选择Java
    5. Next
    6. Finish(选择默认的Toolchain Default)
    7. 安装NDK(Install NDK '21.0.6113669' and sync project)
    8. Build
    9. Make Project(成功打包一个apk，生成在 /root/Codes/native_tutorial/app/build/outputs/apk/debug)

构建的app-debug.apk解压后的lib文件夹下有arm64-v8a / armeabi-v7a / x86 / x86_64 四种架构的so

NDK编译的工具链在/root/Android/Sdk/ndk/21.0.6113669/toolchains

Android Studio自带platform-tools在 /root/Android/Sdk/platform-tools

### 1.2 Native的基本开发

就是一个基础的native HelloWorld，相关的一些点都在代码的备注部分。

/root/Codes/native_tutorial/app/src/main/java/com/example/native_tutorial/MainActivity.java

```
package com.example.native_tutorial;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.util.Log;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {

    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Example of a call to a native method
        TextView tv = findViewById(R.id.sample_text);
        tv.setText(stringFromJNI());

        Log.i("native-lib", invokeMethodWithoutExternC());
    }

    /**
     * A native method that is implemented by the 'native-lib' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI();

    public native String invokeMethodWithoutExternC();
}
```

/root/Codes/native_tutorial/app/src/main/cpp/native-lib.cpp

```
#include <jni.h>
#include <string>


// JNI调用android的log
// https://www.jianshu.com/p/3c1aff6f100f
#include <android/log.h>
#define TAG "JNI_LOG"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, TAG, __VA_ARGS__)
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG, TAG, __VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, TAG, __VA_ARGS__)

/*
 * 1. 静态注册 art对于JNI函数的查询方式决定了native方法的函数名构成
 *     Java_com_example_native_1tutorial_MainActivity_stringFromJNI
 * 2. extern "C"
 *     避免C++编绎器按照C++的方式去编绎C函数(效果就是保持函数签名不变)
 *     Java_com_example_native_1tutorial_MainActivity_stringFromJNI 这种类型的native函数是必须带extern "C"，不然会导致 incorrect linkage
 * 3. 前两个是默认参数 env + jobject(实例方法) / jclass(静态方法)
 * 4. JNIEnv 和 JavaVM
 *     JNIEnv只在当前线程有效, 一个虚拟机一个JavaVM
 * 5. JNIEXPORT 和 JNICALL
 *     需要具体看C代码中的定义
 */
extern "C" JNIEXPORT jstring JNICALL
Java_com_example_native_1tutorial_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    // C/C++方法收到的是JNI类型，返回值也是JNI类型(例如jstring, jintArray)，但是程序内部使用的是C/C++类型，所以中间需要JNI类型与C/C++类型的相互转换。
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}


/*
 * C++ 使用 name mangling 实现重载函数签名之间的区分，可以使用 `c++filt` 还原原始的函数签名，内存中实际存在的是 name mangling 后的函数签名
 * 不使用extern "C"就会被 name mangling
 * # c++filt _Z20methodWithoutExternCv
 * methodWithoutExternC()
 */
std::string methodWithoutExternC() {
    std::string str = "methodWithoutExternC";
    return str;
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_native_1tutorial_MainActivity_invokeMethodWithoutExternC(
        JNIEnv *env,
        jobject thiz) {
    std::string str = methodWithoutExternC();
    return env->NewStringUTF(str.c_str());
}

```

## 2. JNIEnv和反射

完整项目在 2_Reflect.7z

- 照例还是一些文章
    - [FART源码解析及编译镜像支持到Pixel2(xl)](https://www.anquanke.com/post/id/201896)
    - [Java高级特性——反射](https://www.jianshu.com/p/9be58ee20dee)

### 2.1 JNIEnv基础

/root/Codes/Reflect/app/src/main/java/com/example/reflect/MainActivity.java

```
package com.example.reflect;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.util.Log;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {

    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Example of a call to a native method
        TextView tv = findViewById(R.id.sample_text);
        tv.setText(stringFromJNI());

        Log.i("native-lib", "getStringLengthFromJNI(\"hello\") = " + String.valueOf(getStringLengthFromJNI("hello")));
        Log.i("native-lib", "changeString2HelloFromJNI(\"Hello World!\") = " + changeString2HelloFromJNI("Hello World!"));
        Log.i("native-lib", "staticStringFromJNI() = " + MainActivity.staticStringFromJNI());
    }

    /**
     * A native method that is implemented by the 'native-lib' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI();

    public native int getStringLengthFromJNI(String j_str);

    public native String changeString2HelloFromJNI(String j_str);

    public static native String staticStringFromJNI();

}

```

/root/Codes/Reflect/app/src/main/cpp/native-lib.cpp

```
#include <jni.h>
#include <string>

#include <android/log.h>
#define TAG "JNI_LOG"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, TAG, __VA_ARGS__)
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG, TAG, __VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, TAG, __VA_ARGS__)


extern "C" JNIEXPORT jstring JNICALL
Java_com_example_reflect_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}

extern "C" JNIEXPORT jint JNICALL
Java_com_example_reflect_MainActivity_getStringLengthFromJNI(
        JNIEnv *env,
        jobject thiz,
        jstring j_str) {
    int length = env->GetStringUTFLength(j_str);
    return length;
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_reflect_MainActivity_changeString2HelloFromJNI(
        JNIEnv *env,
        jobject thiz,
        jstring j_str) {
    // C/C++方法收到的是JNI类型，返回值也是JNI类型(例如jstring, jintArray)，但是程序内部使用的是C/C++类型，所以中间需要JNI类型与C/C++类型的相互转换。
    const char * c_str = env->GetStringUTFChars(j_str, nullptr);
    LOGI("original j_str = %s", c_str);
    env->ReleaseStringUTFChars(j_str, c_str);

    jstring result = env->NewStringUTF("Hello");
    return result;
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_reflect_MainActivity_staticStringFromJNI(
        JNIEnv *env,
        jclass clazz) {
    // 实例方法第二个参数是jobject，static方法第二个参数是jclass
    std::string hello = "staticStringFromJNI";
    return env->NewStringUTF(hello.c_str());
}
```

### 2.1 反射

Xposed 和 FRIDA 都是基于反射

- 推荐项目
    - [jnitrace](https://github.com/chame1eon/jnitrace)
    - [frida-hook-libart](https://github.com/lasting-yang/frida_hook_libart)
- Frida
    - [Frida源码分析](https://mabin004.github.io/2018/07/31/Mac%E4%B8%8A%E7%BC%96%E8%AF%91Frida/)
    - [Xposed注入实现分析及免重启定制](https://bbs.pediy.com/thread-223713.htm)

对于Java层和Native层通过反射获取类以及对属性和方法的基础调用案例。

/root/Codes/Reflect/app/src/main/java/com/example/reflect/MainActivity.java

```
package com.example.reflect;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.util.Log;
import android.widget.TextView;

import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class MainActivity extends AppCompatActivity {

    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Example of a call to a native method
        TextView tv = findViewById(R.id.sample_text);
        tv.setText(stringFromJNI());

        Log.i("native-lib", "getStringLengthFromJNI(\"hello\") = " + String.valueOf(getStringLengthFromJNI("hello")));
        Log.i("native-lib", "changeString2HelloFromJNI(\"Hello World!\") = " + changeString2HelloFromJNI("Hello World!"));
        Log.i("native-lib", "staticStringFromJNI() = " + MainActivity.staticStringFromJNI());

        // test reflect
        testClazz();
        testField();
        testMethod();

        // invoke class Test from JNI
        String retFromJNI = invokeTestFromJNI();
        Log.i("retFromJNI", retFromJNI);
    }

    public void testClazz() {
        // 反射获取类的方法一
        Class testClazz1 = null;
        try {
            testClazz1 = MainActivity.class.getClassLoader().loadClass("com.example.reflect.Test");
            Log.i("testClazz", "MainActivity.class.getClassLoader().loadClass(\"com.example.reflect.Test\"): " + testClazz1.getName());
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

        // 反射获取类方法二
        Class testClazz2 = null;
        try {
            testClazz2 = Class.forName("com.example.reflect.Test");
            Log.i("testClazz", "Class.forName(\"com.example.reflect.Test\"): " + testClazz2.getName());
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

        // 反射获取类方法三
        Class testClazz3 = Test.class;
        Log.i("testClazz", "Test.class: " + testClazz3.getName());
    }

    public void testField() {
        Class testClazz = Test.class;

        try {
            // 获取任一public属性
            Field publicStaticField_field = testClazz.getField("publicStaticField");
            Log.i("testField", "testClazz.getField(\"publicStaticField\"): " + publicStaticField_field);

            // 获取任一属性
            Field privateStaticField_field = testClazz.getDeclaredField("privateStaticField");
            Log.i("testField", "testClazz.getDeclaredField(\"privateStaticField\"): " + privateStaticField_field);

            // 获取所有public属性
            Field[] fields = testClazz.getFields();
            for(Field i: fields){
                Log.i("testField", "testClazz.getFields(): " + i);
            }

            // 获取所有属性
            Field[] declaredFields = testClazz.getDeclaredFields();
            for(Field i: declaredFields){
                Log.i("testField", "testClazz.getDeclaredFields(): " + i);
            }

            try {
                // 获取属性的值: 静态传入null,动态传入对象
                String staticValue = (String) publicStaticField_field.get(null);
                Log.i("testField", "(String) publicStaticField_field.get(null): " + staticValue);
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }

            try {
                // 获取private属性需要设置读取权限
                // xposed原理分析 https://blog.csdn.net/zhangmiaoping23/article/details/52572447
                // setAccessible需要在get/set前设置
                privateStaticField_field.setAccessible(true);
                String staticPrivateValue = (String) privateStaticField_field.get(null);
                Log.i("testField", "(String) privateStaticField_field.get(null): " + staticPrivateValue);

                // 修改属性值,静态传入null,动态传入对象
                privateStaticField_field.set(null, "MODIFIED privateStaticField");
                staticPrivateValue = (String) privateStaticField_field.get(null);
                Log.i("testField", "(String) privateStaticField_field.get(null): " + staticPrivateValue);
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }

        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        }
    }

    public void testMethod() {
        Class testClazz = Test.class;

        // 获取public方法
        Method publicStaticFunc_method = null;
        try {
            publicStaticFunc_method = testClazz.getMethod("publicStaticFunc");
            Log.i("testMethod", "testClazz.getMethod(\"publicStaticFunc\"): " + publicStaticFunc_method);
            // 主动调用方法,静态传入null,动态传入对象
            publicStaticFunc_method.invoke(null);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }

        // 获取private方法
        Method privateStaticFunc_method = null;
        try {
            privateStaticFunc_method = testClazz.getDeclaredMethod("privateStaticFunc");
            Log.i("testMethod", "testClazz.getDeclaredMethod(\"privateStaticFunc\"): " + privateStaticFunc_method);
            privateStaticFunc_method.setAccessible(true);
            privateStaticFunc_method.invoke(null);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }

        // 只能打印public方法,并且基类的public方法也可以打印
        Method[] methods = testClazz.getMethods();
        for(Method i: methods){
            Log.i("testMethod", "testClazz.getMethods(): " + i);
        }

        // 可以打印自己的private方法,没有基类
        Method[] declaredMethods = testClazz.getDeclaredMethods();
        for(Method i: declaredMethods){
            Log.i("testMethod", "testClazz.getDeclaredMethods(): " + i);
        }
    }

    /**
     * A native method that is implemented by the 'native-lib' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI();

    public native int getStringLengthFromJNI(String j_str);

    public native String changeString2HelloFromJNI(String j_str);

    public static native String staticStringFromJNI();

    public static native String invokeTestFromJNI();
}

```

/root/Codes/Reflect/app/src/main/java/com/example/reflect/Test.java

```
package com.example.reflect;

import android.util.Log;

public class Test {
    private String identifier = null;
    public static String publicStaticField = "publicStaticField";
    public String publicField = "publicField";
    private static String privateStaticField = "privateStaticField";
    private String privateField = "privateField";

    public Test(){
        identifier = "Test()";
    }
    public Test(String arg1){
        identifier = "Test(String arg1)";
    }
    public Test(String arg1, int arg2){
        identifier = "Test(String arg1, int arg2)";
    }

    public static void publicStaticFunc(){
        Log.i("Test", "publicStaticFunc");
    }
    public void publicFunc(){
        Log.i("Test", "publicFunc");
    }
    private static void privateStaticFunc(){
        Log.i("Test", "privateStaticFunc");
    }
    private void privateFunc(){
        Log.i("Test", "privateFunc");
    }
}

```

/root/Codes/Reflect/app/src/main/cpp/native-lib.cpp

```
#include <jni.h>
#include <string>

#include <android/log.h>
#define TAG "JNI_LOG"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, TAG, __VA_ARGS__)
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG, TAG, __VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, TAG, __VA_ARGS__)


extern "C" JNIEXPORT jstring JNICALL
Java_com_example_reflect_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}

extern "C" JNIEXPORT jint JNICALL
Java_com_example_reflect_MainActivity_getStringLengthFromJNI(
        JNIEnv *env,
        jobject thiz,
        jstring j_str) {
    int length = env->GetStringUTFLength(j_str);
    return length;
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_reflect_MainActivity_changeString2HelloFromJNI(
        JNIEnv *env,
        jobject thiz,
        jstring j_str) {
    // C/C++方法收到的是JNI类型，返回值也是JNI类型(例如jstring, jintArray)，但是程序内部使用的是C/C++类型，所以中间需要JNI类型与C/C++类型的相互转换。
    const char * c_str = env->GetStringUTFChars(j_str, nullptr);
    LOGI("original j_str = %s", c_str);
    env->ReleaseStringUTFChars(j_str, c_str);

    jstring result = env->NewStringUTF("Hello");
    return result;
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_reflect_MainActivity_staticStringFromJNI(
        JNIEnv *env,
        jclass clazz) {
    // 实例方法第二个参数是jobject，static方法第二个参数是jclass
    std::string hello = "staticStringFromJNI";
    return env->NewStringUTF(hello.c_str());
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_reflect_MainActivity_invokeTestFromJNI(
        JNIEnv *env,
        jclass clazz) {
    // so中获取class,需要用/分隔
    jclass testClass = env->FindClass("com/example/reflect/Test");
    // so中获取field
    jfieldID publicStaticField = env->GetStaticFieldID(testClass, "publicStaticField", "Ljava/lang/String;");
    jstring publicStaticField_value = (jstring) env->GetStaticObjectField(testClass, publicStaticField);

    // 将字符串转换为C才能打印
    const char* value_publicStaticField = env->GetStringUTFChars(publicStaticField_value, nullptr);
    LOGI("value_publicStaticField: %s", value_publicStaticField);

    std::string str = "invokeTestFromJNI";
    return env->NewStringUTF(str.c_str());
}
```

## 3. IDA调试so基本思路：开发一个Frida反调试

- 几个Frida反调试案例
    - [frida-detection-demo](https://github.com/b-mueller/frida-detection-demo)
    - [AntiFrida](https://github.com/qtfreet00/AntiFrida)
    - [多种特征检测 Frida](https://bbs.pediy.com/thread-217482.htm)
    - [来自高维的对抗 - 逆向TinyTool自制](https://developer.aliyun.com/article/71120)
    - [Unicorn 在 Android 的应用](https://bbs.pediy.com/thread-253868.htm)
    - [爱奇艺 - 风险控制笔记](https://github.com/WalterInSH/risk-management-note/blob/master/README.md)

对于 Java层API、Native层API 都是可以被 hook 的，如果自定义函数调用 Syscall 就会增加难度，对于syscall也是可以hook的但是需要自己编译内核打印内核的系统调用(来自高维的对抗 - 逆向TinyTool自制)，Unicorn打印反调试(Unicorn 在 Android 的应用)，Nethunter可以直接打印syscall

风控的关键在于检测的项目够多，深层的混淆以及vmp(龙卷风)，只是靠自己修改字符串特征来编译 Xposed / Frida 试图过风控是不现实的。

### 3.1 使用 native 手写一个简易的 anti-frida

/root/Codes/Reflect/app/src/main/java/com/example/reflect/MainActivity.java

```
package com.example.antifrida;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.util.Log;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {

    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Example of a call to a native method
        TextView tv = findViewById(R.id.sample_text);
        tv.setText("ANTI-FRIDA");

        antiFrida();
    }

    // adb logcat 报错 read: unexpected EOF!，需要 设置 - 开发者模式 - 调试(日志记录器缓冲区大小) - 16M
    public native void antiFrida();
}

```

/root/Codes/Reflect/app/src/main/cpp/native-lib.cpp

```
#include <jni.h>
#include <string>
#include <stdio.h>
#include <jni.h>
#include <string.h>
#include <fcntl.h>
#include <pthread.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <android/log.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>


#include <android/log.h>
#define TAG "JNI_LOG"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, TAG, __VA_ARGS__)
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG, TAG, __VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, TAG, __VA_ARGS__)

// https://github.com/b-mueller/frida-detection-demo
// https://github.com/qtfreet00/AntiFrida
void *detect_frida_loop(void *) {
    struct sockaddr_in sa;
    memset(&sa, 0, sizeof(sa));
    sa.sin_family = AF_INET;
    inet_aton("0.0.0.0", &(sa.sin_addr));
    int sock;
    int i;
    int ret;
    char res[7];
    while (true) {
        LOGI("enter detect_frida_loop");

        // TCP扫描需要开网络权限
        // /root/Codes/AntiFrida/app/src/main/AndroidManifest.xml 添加 <uses-permission android:name="android.permission.INTERNET" />
        // 只检测开启在[8000, 9000)端口之间的Frida
        for(i = 8000; i < 9000; i++){
            sock = socket(AF_INET,SOCK_STREAM, 0);
            sa.sin_port = htons(i);
            LOGI("detect_frida_loop is scanning socket port %d", i);

            if (connect(sock , (struct sockaddr*)&sa , sizeof sa) != -1) {
                memset(res, 0 , 7);
                send(sock, "\x00", 1, NULL);
                send(sock, "AUTH\r\n", 6, NULL);
                // give it some time to answer
                usleep(500);
                if ((ret = recv(sock, res, 6, MSG_DONTWAIT)) != -1) {
                    if (strcmp(res, "REJECT") == 0) {
                        LOGI("VITAL: Frida was appeared in port %d", i);
                    } else {
                        LOGI("SAFE: Frida was not found");
                    }
                }
            }
            close(sock);
        }
    }
}

// 函数的返回值注意和声明要一致!!!否则代码自己会莫名其妙崩
extern "C" JNIEXPORT void JNICALL
Java_com_example_antifrida_MainActivity_antiFrida(
        JNIEnv *env,
        jobject thiz) {
    pthread_t t;
    pthread_create(&t,NULL, detect_frida_loop, (void*)NULL);
    LOGI("Java_com_example_antifrida_MainActivity_antiFrida start");
}

```

/root/Codes/AntiFrida/app/src/main/AndroidManifest.xml

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.antifrida">

    <!-- TCP扫描需要开网络权限 -->
    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

### 3.2 IDA 动静态调试

准备工作: 把IDA/dbgsrv下面的android_server64(android_server 32位手机) push 到 /data/local/tmp，添加执行权限后开启

```bash
root@Nick:~# adb push Downloads/android_server64 /data/local/tmp

sailfish:/data/local/tmp # chmod +x android_server64
sailfish:/data/local/tmp # ./android_server64
```

开启对应手机架构版本的IDA - Debugger - Attach - Remote ARMLinux/Android debugger - hostname写入手机ip - OK - 选择进程 - OK

### 3.2.1 动态调试

IDA 右侧 Modules 选 libnative-lib.so(可以Ctrl + F过滤) - 双击 - 进入Module: libnative-lib.so 可以看到 detect_frida_loop 和 Java_com_example_antifrida_MainActivity_antiFrida - 下断点调试

### 3.2.2 静态调试

直接IDA打开so即可

可以右键 代码 Synchronize with - Hex View / PC

#### 3.2.2.1 so硬编码修改

- 静态修改so
    - 1. 好像不行，这个办法没法生成新so
        - 找到REJECT(右键Synchronize with hex view, 到hex view 视图修改硬编码，右键edit，右键apply change) - 保存
    - 2. 硬编码替换so
        - 找到REJECT设置 Synchronize with hex view - IDA左上角工具栏Edit - Patch program - Change byte/words - 修改字符 - OK - Edit - Patch program - Apply patches to input files(额外勾选Create backup) OK

patch后的libnative-lib.so 的 md5 是 6a5615f7f6f717212c02164e82e58b9f
patch前的libnative-lib.so 的 md5 是 e8f3be9db5b6e55afeb19d37761d88b8

#### 3.2.2.2 objection找modules路径

```
root@Nick:~# objection -N -h 192.168.50.42 -p 8888 -g com.example.antifrida explore

com.example.antifrida on (google: 8.1.0) [net] # memory list modules

显示:
libnative-lib.so                                 0x726de9f000  12288 (12.0 KiB)      /data/app/com.example.antifrida-SwKcSsHgcvghJstE1gnqZg==/lib/arm64/libnativ...

```

#### patch so

```bash
root@Nick:~/Downloads/# adb push libnative-lib.so.anti_frida_patched /sdcard/Download/

sailfish:/ $ su

sailfish:/ # cd /data/app/com.example.antifrida-SwKcSsHgcvghJstE1gnqZg==/lib/arm64/

sailfish:/data/app/com.example.antifrida-SwKcSsHgcvghJstE1gnqZg==/lib/arm64/ # cp /sdcard/Download/libnative-lib.so.anti_frida_patched libnative-lib.so

127|sailfish:/data/app/com.example.antifrida-SwKcSsHgcvghJstE1gnqZg==/lib/arm64/ # chmod 755 libnative-lib.so.anti_frida_patched
```

对于加载的so需要做完整性校验否则硬编码就可以被修改

## x. 几个观点

1. 开发是核心，以开发的眼光看待问题
2. 学会hook大法之后是几乎不需要手动调的

