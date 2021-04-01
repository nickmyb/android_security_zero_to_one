FRIDA hook Native基础
=====================

## 0. 致谢

以下内容全部来源于 @**r0ysue**

![肉丝的星球-大家一起来玩](../1_environment_preparation/static/r0ysue.jpg)

[相关内容全部上传百度网盘 提取码：ljt1](https://pan.baidu.com/s/1__B7dnZ2xKBdXTNE_sklZg)

[大黑客sakura的同一课程的学习笔记](https://eternalsakura13.com/2020/07/04/frida/)

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

完整项目在 3_AntiFrida.7z + 3_libnative-lib.so.anti_frida_patched + 3_libnative-lib.so.original

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

#### 3.2.2.3 patch so

```bash
root@Nick:~/Downloads/# adb push 3_libnative-lib.so.anti_frida_patched /sdcard/Download/

sailfish:/ $ su

sailfish:/ # cd /data/app/com.example.antifrida-SwKcSsHgcvghJstE1gnqZg==/lib/arm64/

sailfish:/data/app/com.example.antifrida-SwKcSsHgcvghJstE1gnqZg==/lib/arm64/ # cp /sdcard/Download/3_libnative-lib.so.anti_frida_patched libnative-lib.so

127|sailfish:/data/app/com.example.antifrida-SwKcSsHgcvghJstE1gnqZg==/lib/arm64/ # chmod 755 3_libnative-lib.so.anti_frida_patched
```

对于加载的so需要做完整性校验否则硬编码就可以被修改

## 4. java native / 符号 / JNI(art) / libc hook

完整项目在 4_Reflect.7z + 3_AntiFrida.7z + 4_anti_frida.js + 4_reflect.js

- 推荐几个链接
    - [教我兄弟学Android逆向系列课程](https://www.52pojie.cn/thread-742703-1-1.html)
    - [彻底搞清楚 GOT 和 PLT](https://www.jianshu.com/p/5092d6d5caa3)

ELF里面的GOT段/PLT段找到当前的符号，链接到偏移的地址，然后拿到符号进行跳转，ELF结构分节和区，里面就有GOT/PLT区表。

Frida的native hook主要分为符号hook(PLT/GOT hook) 和 inline hook

### 4.1 IDA导入jni.h

1. 复制出jni.h(~/Android/Sdk/ndk/21.0.6113669/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/include/jni.h，不同的apk只需要导入一次jni.h即可)

2. 导入 jni.h (左上角工具栏 File - Load file - Parse C header file - 打开)让IDA完整反编译出_JNIEnv *

3. 导入会报错， 删除jni报错对应的.h， 重新导入

4. 如果编译的函数参数不是_JNIEnv *，就右键修改一下  如果没有用到函数的参数,那IDA反编译不会显示这个参数的

### 4.2 so研究方式

主要就是通过trace来观察so层的行为

/root/Codes/Reflect/app/src/main/java/com/example/reflect/MainActivity.java

在onCreate添加了一下调用native函数的死循环用于后面的so层hook实验

```
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

        while (true) {
            try {
                Thread.sleep(2000);
                // Do some stuff
            } catch (Exception e) {
                e.getLocalizedMessage();
            }
            Log.i("while in onCreate", changeString2HelloFromJNI("string in Java"));
        }
    }
```

reflect.js

```
/*
 * frida -H 192.168.50.42:8888 com.example.reflect -l agent/reflect.js
 */


// 检查 以module_name_pattern开头 的 module(即so) 是否存在在当前apk
function if_module_existed(module_name_pattern) {
    var modules = Process.enumerateModules()
    var module = null
    var existed = false

    for (var i = 0; i < modules.length; i++) {
        module = modules[i]

        if (module.name.indexOf(module_name_pattern) != -1) {
            if (!existed) {
                console.log('search module_name_pattern[' + module_name_pattern + '] found:')
                console.log()
                existed = !existed
            }
            console.log('module.name = ' + module.name)
            console.log('module.base = ' + module.base)
            console.log()
        }
    }
    return existed
}


function get_symbol_offset() {
    var libnative_lib_so_name = "libnative-lib.so"
    var existed = if_module_existed(libnative_lib_so_name)
    if (existed) {
        console.log(libnative_lib_so_name, 'is existed!')
    } else {
        console.log(libnative_lib_so_name, 'is not existed!')
    }

    var libnative_lib_so_address  = Module.findBaseAddress(libnative_lib_so_name)
    console.log("libnative_lib_so_address =", libnative_lib_so_address)

    // 先找到module然后找到想要hook的符号
    if (libnative_lib_so_address) {
        var symbol_name = null
        var symbol_address = null

        var symbol_name = 'Java_com_example_reflect_MainActivity_changeString2HelloFromJNI'
        symbol_address = Module.findExportByName(libnative_lib_so_name, symbol_name)
        console.log(symbol_name + ' address = ' + symbol_address)
        console.log(symbol_name + ' offset =', '0x' + (symbol_address - libnative_lib_so_address).toString(16))
    }
}


// 需要定义async函数且frida使用v8 --runtime=v8
// async function sleep() {
//     await new Promise(r => setTimeout(r, 5000))
// }


// 符号hook
function symbol_hook() {
    // spawn 需要延迟执行否则libnative-lib.so还没有加载, 使用 setTimeout 或者 代码里加sleep
    // TODO 问: 对于只执行一次的native函数,spawn的时候so还没有加载怎么hook?
    var libnative_lib_so_name = "libnative-lib.so"
    var existed = if_module_existed(libnative_lib_so_name)
    if (existed) {
        console.log(libnative_lib_so_name, 'is existed!')
    } else {
        console.log(libnative_lib_so_name, 'is not existed!')
    }

    var libnative_lib_so_address  = Module.findBaseAddress(libnative_lib_so_name)
    console.log("libnative_lib_so_address =", libnative_lib_so_address)

    // 先找到module然后找到想要hook的符号
    if (libnative_lib_so_address) {
        var symbol_name = null
        var symbol_address = null

        var symbol_name = 'Java_com_example_reflect_MainActivity_changeString2HelloFromJNI'
        symbol_address = Module.findExportByName(libnative_lib_so_name, symbol_name)
        console.log(symbol_name + ' address = ' + symbol_address)
        console.log(symbol_name + ' offset =', '0x' + (symbol_address - libnative_lib_so_address).toString(16))
    }

    Interceptor.attach(symbol_address, {
        onEnter: function(args) {
            // args 是 jstring

            // Java_com_example_reflect_MainActivity_changeString2HelloFromJNI 三个参数
            console.log(symbol_address, "raw args[0], args[1], args[2]:", args[0], args[1], args[2])

            // 打印方式1
            // 星球jbytearray
            // 可能需要使用32位机器,sailfish这边是Error: access violation accessing 0x7133cabb8
            // var args2_address = args[2].readPointer()
            // console.log(hexdump(args2_address))

            // 打印方式2
            // 通过JNI接口打印jstring, 注意需要对应到frida的api, 不要直接使用java的
            // https://github.com/frida/frida-java-bridge/blob/062999b618451e5ac414d08d4f52bec05bd6adf6/lib/env.js
            console.log('args[2]:', Java.vm.getEnv().getStringUtfChars(args[2], null).readCString())
        },
        onLeave: function(ret) {
            // ret 是 jstring
            console.log("raw ret:", ret)
            console.log('ret:', Java.vm.getEnv().getStringUtfChars(ret, null).readCString())
            var new_ret = Java.vm.getEnv().newStringUtf("new_ret from " + symbol_name)

            // 替换返回值
            ret.replace(ptr(new_ret))
        }
    })
}


// https://github.com/chame1eon/jnitrace
function hook_libart_so() {
    var libart_so_name = "libart.so"
    var existed = if_module_existed(libart_so_name)
    if (existed) {
        console.log(libart_so_name, 'is existed!')
    } else {
        console.log(libart_so_name, 'is not existed!')
    }

    var libart_so_address  = Module.findBaseAddress(libart_so_name)
    console.log("libart_so_address =", libart_so_address)

    // 由于 name mangling 导致符号被混淆(name mangling也不是完全混淆,真实函数名还是包含在结果中的), 所以需要枚举所有符号
    // 不要用 Process.findModuleByName(module_name).enumerateSymbols() 符号可能枚举结果为空
    var symbols = Module.enumerateSymbolsSync(libart_so_name)
    var jni_function_name = 'GetStringUTFChars'
    var jni_function_address = null

    for (var i = 0; i < symbols.length; i++){
        var symbol = symbols[i]
        if((symbol.name.indexOf("CheckJNI") == -1) && (symbol.name.indexOf("JNI") >= 0)) {
            if (symbol.name.indexOf(jni_function_name) >= 0) {
                console.log('symbol.name =', symbol.name)
                console.log('symbol.address =', symbol.address)
                jni_function_address = symbol.address
            }
        }
    }
    console.log(jni_function_name + ' address =', jni_function_address)

    // JNI 的 GetStringUTFChars 竟然可以打印android的Log.i, 陈总建议是可以看native的log实现, 然而现在并看不懂
    Interceptor.attach(jni_function_address, {
        onEnter: function(args) {
            console.log(jni_function_name + " args[0] =", args[0])
            console.log(jni_function_name + " args[1] =", args[1])

            // args[0] 是 JNIEnv
            // JNIEnv 可以用 hexdump(args[0].readPointer()), jxxx参数不能用, 要调用JNI的方法打印
            console.log('hexdump args[0] =', hexdump(args[0].readPointer()))
            // args[1] 是 jstring, 不能用hexdump
            // console.log('hexdump args[1] =', hexdump(args[1].readPointer()))
            // 两个接口 getEnv / tryGetEnv
            console.log('args[1] =', Java.vm.getEnv().getStringUtfChars(args[1]).readCString())
            // console.log('args[1] =', Java.vm.tryGetEnv().getStringUtfChars(args[1]).readCString())

            // 打印调用栈: https://frida.re/docs/javascript-api/
            var trace_back_log = Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n')
            console.log(jni_function_name + ' traceback:\n' + trace_back_log + '\n')
        },
        onLeave: function(ret) {
            // char * 可以直接打印 ptr(ret).readCString()
            console.log(jni_function_name + " ret =", ptr(ret).readCString())
        }
    })
}


function main() {
    // get_symbol_offset()

    // symbol_hook()

    hook_libart_so()
}


setImmediate(main)

```

anti_frida.js

```
/*
 * frida -H 192.168.50.42:8888 -f com.example.antifrida -l agent/anti_frida.js
 */

// 1. 对native函数的hook，和普通java函数一致
function remove_antiFrida() {
    // 避免logcat日志过多无法看出来hook native函数是否生效，先清理一下logcat日志
    // 清除logcat日志: adb logcat -c

    var MainActivity = Java.use('com.example.antifrida.MainActivity')
    MainActivity.antiFrida.implementation = function() {
        console.log('enter remove_antiFrida')
        return
    }
}


// 检查 以module_name_pattern开头 的 module(即so) 是否存在在当前apk
function if_module_existed(module_name_pattern) {
    var modules = Process.enumerateModules()
    var module = null
    var existed = false

    for (var i = 0; i < modules.length; i++) {
        module = modules[i]

        if (module.name.indexOf(module_name_pattern) != -1) {
            if (!existed) {
                console.log('search module_name_pattern[' + module_name_pattern + '] found:')
                console.log()
                existed = !existed
            }
            console.log('module.name = ' + module.name)
            console.log('module.base = ' + module.base)
            console.log()
        }
    }
    return existed
}


function enumerate_modules_and_exports(module_name) {
    // 枚举符号和导出符号
    // symbols > exports
    // 但enumerateSymbols可能搜索结果为空，并且as打包生成的so会出现有导出符号没有符号的情况，不知道是做了什么配置，日常请使用sync版本
    // var symbols = Process.findModuleByName(libnative_lib_so_name).enumerateSymbols()
    // var symbols = Module.enumerateSymbolsSync(libnative_lib_so_name)

    // var symbols = Process.findModuleByName(module_name).enumerateSymbols()
    var symbols = Module.enumerateSymbolsSync(module_name)
    var exports = Process.findModuleByName(module_name).enumerateExports()
}


// 2. 找到偏移
function get_symbol_offset() {
    // root@Nick:~# objection -N -h 192.168.50.42 -p 8888 -g com.example.antifrida explore
    // com.example.antifrida on (google: 8.1.0) [net] # memory list modules

    // 找到:
    // libnative-lib.so                                 0x72b6e6c000  12288 (12.0 KiB)      /data/app/com.example.antifrida-cUCV8773Ge8uJPawmLlRPw==/lib/arm64/libnativ...

    var libnative_lib_so_name = "libnative-lib.so"
    var existed = if_module_existed('libnative')
    if (existed) {
        console.log(libnative_lib_so_name, 'is existed!')
    } else {
        console.log(libnative_lib_so_name, 'is not existed!')
    }

    // 找到基址, app重启后基址和导出函数地址是会变化的, 但偏移量是固定的
    var libnative_lib_so_address  = Module.findBaseAddress(libnative_lib_so_name)
    console.log("libnative_lib_so_address =", libnative_lib_so_address)

    // 先找到module然后找到想要hook的符号
    if (libnative_lib_so_address) {
        // 由于namemagling导致符号被混淆, detect_frida_loop 的导出符号是 _Z17detect_frida_loopPv
        // extern "C" JNIEXPORT void JNICALL Java_com_example_antifrida_MainActivity_antiFrida
        var symbol_name = null
        var symbol_address = null

        var symbol_name = 'Java_com_example_antifrida_MainActivity_antiFrida'
        symbol_address = Module.findExportByName(libnative_lib_so_name, symbol_name)
        console.log(symbol_name + ' address = ' + symbol_address)
        // 偏移量是固定的, 和Frida的IDA-View按空格显示的偏移一致
        console.log(symbol_name + ' offset =', '0x' + (symbol_address - libnative_lib_so_address).toString(16))

        symbol_name = '_Z17detect_frida_loopPv'
        symbol_address = Module.findExportByName(libnative_lib_so_name, symbol_name)
        console.log(symbol_name + ' address = ' + symbol_address)
        console.log(symbol_name + ' offset =', '0x' + (symbol_address - libnative_lib_so_address).toString(16))
    }
}


function hook_libc_so() {
    var libc_so_name = "libc.so"
    var existed = if_module_existed(libc_so_name)
    if (existed) {
        console.log(libc_so_name, 'is existed!')
    } else {
        console.log(libc_so_name, 'is not existed!')
    }

    var libc_so_address  = Module.findBaseAddress(libc_so_name)
    console.log("libc_so_address =", libc_so_address)

    var libc_function_name = 'pthread_create'
    var libc_function_address = null

    var symbols = null

    symbols = Module.enumerateSymbolsSync(libc_so_name)

    // C函数名字该是什么就是什么
    // objection: memory list exports libc.so --json libc_so.json
    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i]
        var name = symbol.name
        var address = symbol.address
        // 使用android native中的方法名
        if ((name.indexOf(libc_function_name) >= 0) && (name.indexOf(".cpp") == -1)) {
            console.log('name:', name)
            console.log('address:', address)

            libc_function_address = address
        }
    }

    Interceptor.attach(libc_function_address, {
        onEnter: function(args) {
            console.log(libc_function_name, 'arg[0] =', args[0])
            console.log(libc_function_name, 'arg[1] =', args[1])
            console.log(libc_function_name, 'arg[2] =', args[2])
            console.log(libc_function_name, 'arg[3] =', args[3])
        },
        onLeave: function(ret) {
            console.log('ret =', ret)
        }
    })
}


function main() {
    // Java.perform(remove_antiFrida)

    // get_symbol_offset()

    hook_libc_so()
}


setImmediate(main)

```

## 5. JNI_Onload / 主动调用 / inline_hook

完整项目在 5_Reflect.7z + 5_AntiFrida.7z + + 5_anti_frida.js + 5_reflect.js

- 推荐几个链接
    - [ndk-samples](https://github.com/android/ndk-samples.git)
    - [Android逆向新手答疑解惑篇——JNI与动态注册](https://bbs.pediy.com/thread-224672.htm)
    - [Java与Native相互调用](https://www.jianshu.com/p/b71aeb4ed13d)

### 5.1 动态注册

- 注册native函数的方式
    1. 静态注册: Java_com_example_demo_MainActivity_stringFromJNI
    2. 动态注册: RegisterNatives

#### 5.1.1 JNI_OnLoad动态注册

/root/Codes/Reflect/app/src/main/java/com/example/antifrida/MainActivity.java 修改onCreate 并添加 native方法 dynamicallyRegisteredStringFromJNI

```

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

        while (true) {
            try {
                Thread.sleep(2000);
                // Do some stuff
            } catch (Exception e) {
                e.getLocalizedMessage();
            }
            Log.i("while in onCreate", changeString2HelloFromJNI("string in Java"));
            Log.i("RegisterNatives", dynamicallyRegisteredStringFromJNI());
        }
    }

    // IDA中找不到动态注册的dynamicallyRegisteredStringFromJNI
    public native String dynamicallyRegisteredStringFromJNI();
```

/root/Codes/Reflect/app/src/main/cpp/native-lib.cpp 添加下面两个方法用于主动注册

```
// 去除了extern "C", 这样会被name mangling
JNIEXPORT jstring JNICALL
dynamicallyRegisteredString(
        JNIEnv *env,
        jobject thiz) {
    std::string str = "dynamicallyRegisteredStringFromJNI";
    return env->NewStringUTF(str.c_str());
}

// 在jni.h中定义但没有实现, 类似Activity的onCreate
JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved)
{
    JNIEnv *env;
    // 通过JavaVM获取JNIEnv
    vm->GetEnv((void**)&env, JNI_VERSION_1_6);
    JNINativeMethod methods[] = {
            // Java中native函数名, (参数)返回值, native函数
            // 当参数为引用类型的时候，参数类型的标识为"L包名", 其中包名的.(点)要换成"/"
            {"dynamicallyRegisteredStringFromJNI", "()Ljava/lang/String;", (void*)dynamicallyRegisteredString},
    };
    // 第一个参数: native函数所在的类，可通过FindClass获取(将.换成/)
    // 第二个参数: 一个数组，其中包含注册信息
    // 第三个参数: 数量
    env->RegisterNatives(env->FindClass("com/example/reflect/MainActivity"), methods, 1);
    return JNI_VERSION_1_6;
}

```

#### 5.1.2 trace libart

应该花时间多看大佬们写的代码,先抄后理解

- RegisterNatives / libart(JNI)
    - [frida_hook_libart](https://github.com/lasting-yang/frida_hook_libart)

注意hook时机,例如hook RegisterNatives就需要用spawn的方式启动

#### 5.1.3 frida反调试

- 修改AOSP源码过反调试,直接修改源码做到打印
    - [修改AOSP源码过反调试](https://bbs.pediy.com/thread-255653.htm)

#### 5.1.4 主动调用native函数

Interceptor.replace

reflect.js

```
// https://frida.re/docs/javascript-api/
// Interceptor.replace
function native_initiatively_invoke() {
    var module_name = 'libnative-lib.so'
    var function_name = 'Java_com_example_reflect_MainActivity_changeString2HelloFromJNI'
    var Java_com_example_reflect_MainActivity_changeString2HelloFromJNI_address = Module.getExportByName(module_name, function_name)
    var Java_com_example_reflect_MainActivity_changeString2HelloFromJNI = new NativeFunction(Java_com_example_reflect_MainActivity_changeString2HelloFromJNI_address, 'pointer', ['pointer', 'pointer', 'pointer'])

    // Interceptor.attach只能hook和修改返回值, Interceptor.replace可以主动调用
    Interceptor.replace(Java_com_example_reflect_MainActivity_changeString2HelloFromJNI_address, new NativeCallback(function(env, thiz, j_str) {
        var str = Java.vm.getEnv().getStringUtfChars(j_str).readCString()
        console.log('j_str =', str)

        // 主动调用
        // var ret = Java_com_example_reflect_MainActivity_changeString2HelloFromJNI(env, thiz, j_str)
        // 替换参数的主动调用
        // 创建一个 jstring: Java.vm.getEnv().newStringUtf(""), 创建一个 char*: Memory.allocUtf8String("")
        var new_j_str = Java.vm.getEnv().newStringUtf("HelloHello")
        var ret = Java_com_example_reflect_MainActivity_changeString2HelloFromJNI(env, thiz, new_j_str)

        // 但这里也没有做到完全的主动调用, replace是类似于Java.choose, native new参数解决就是真正的主动调用了
        return ret
    }, 'pointer', ['pointer', 'pointer', 'pointer']))
}

```

anti_frida.js

```
function anti_anti_frida() {
    var libnative_lib_so_name = "libnative-lib.so"
    var existed = if_module_existed(libnative_lib_so_name)
    if (existed) {
        console.log(libnative_lib_so_name, 'is existed!')
    } else {
        console.log(libnative_lib_so_name, 'is not existed!')
    }

    var libnative_lib_so_address  = Module.findBaseAddress(libnative_lib_so_name)
    console.log("libnative_lib_so_address =", libnative_lib_so_address)

    var detect_frida_loop_name = 'detect_frida_loop'
    var detect_frida_loop_address = null

    var symbols = null

    // 坑坑坑!!!
    // 找自己用as写的native函数要enumerateExportsSync, 不能enumerateSymbolsSync
    symbols = Module.enumerateExportsSync(libnative_lib_so_name)

    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i]
        var name = symbol.name
        var address = symbol.address
        if (name.indexOf(detect_frida_loop_name) >= 0) {
            console.log('name:', name)
            console.log('address:', address)

            detect_frida_loop_address = address
        }
    }

    // detect_frida_loop地址如下, 可以利用后三位地址偏移特征进行过滤
    // 0x72b6f34a7c
    // 0x72b6e97a7c


    var libc_so_name = "libc.so"
    var existed = if_module_existed(libc_so_name)
    if (existed) {
        console.log(libc_so_name, 'is existed!')
    } else {
        console.log(libc_so_name, 'is not existed!')
    }

    var libc_so_address  = Module.findBaseAddress(libc_so_name)
    console.log("libc_so_address =", libc_so_address)

    var libc_function_name = 'pthread_create'
    var libc_function_address = null

    var symbols = null

    symbols = Module.enumerateSymbolsSync(libc_so_name)

    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i]
        var name = symbol.name
        var address = symbol.address
        // 使用android native中的方法名
        if ((name.indexOf(libc_function_name) >= 0) && (name.indexOf(".cpp") == -1)) {
            console.log('name:', name)
            console.log('address:', address)

            libc_function_address = address
        }
    }

    // c的大函数的signature都可以直接搜到
    var pthread_create = new NativeFunction(libc_function_address, 'int', ['pointer', 'pointer', 'pointer', 'pointer'])

    Interceptor.replace(libc_function_address, new NativeCallback(function(arg0, arg1, arg2, arg3) {
        var ret = null
        if (String(arg2).endsWith('a7c')) {
            // 这里不能传NULL,会导致报错的(NULL就是空指针但为什么会导致报错呢),是因为NULL的address是0的原因吗? 可以传一个枚举出来的合理的函数地址, 如何传一个NULL pointer呢?
            // ret = pthread_create(arg0, arg1, NULL, arg3)
            ret = 0
            console.log(detect_frida_loop_name, 'is passed')
        } else {
            ret = pthread_create(arg0, arg1, arg2, arg3)
            console.log('normal pthread_create')
        }
        return ret
    }, 'int', ['pointer', 'pointer', 'pointer', 'pointer']))
}

```

#### 5.1.4 inline hook

- IDA(IDA View)判断是ARM指令还是thumb指令
    - 方法一
        1. Options - General - Number of opcodes byte 修改为8(或4)
        2. 如果地址显示是2个数字(可能需要按一下空格)就是thumb,地址需要+1,4个数字就不需要+1
    - 方法二
        1. 地址的奇偶
        2. 奇数是thumb指令

```reflect.js
function inline_hook() {
    var libnative_lib_so_name = "libnative-lib.so"
    var existed = if_module_existed(libnative_lib_so_name)
    if (existed) {
        console.log(libnative_lib_so_name, 'is existed!')
    } else {
        console.log(libnative_lib_so_name, 'is not existed!')
    }

    var libnative_lib_so_address  = Module.findBaseAddress(libnative_lib_so_name)
    console.log("libnative_lib_so_address =", libnative_lib_so_address)

    // 先找到module然后找到想要hook的符号
    if (libnative_lib_so_address) {
        var offset = 0xFC28
        var FC28_address = libnative_lib_so_address.add(offset)


        Interceptor.attach(FC28_address, {
            onEnter: function(args) {
                // hook地址没有参数,一般只能打印寄存器
                // 因为是64位所以是x
                // IDA找到dynamicallyRegisteredString,找偏移末尾是C28的前面一点下断点,可以发现断点到C28时候寄存器的值和inline_hook是一致的
                console.log('this.context.PC =', this.context.PC)
                console.log('this.context.x1 =', this.context.x1)
                console.log('this.context.x5 =', this.context.x5)
                console.log('this.context.x10 =', this.context.x10)
            },
            onLeave: function(ret) {
                console.log('ret =', ret)
            }
        })
    }
}
```

## 6. Frida hook native App实战

完整项目在 6_xman.js

如果so中的函数符号被strip掉的话IDA是搜不到的

### 6.1 反调试

- 相关文章
    - [Android反调试技术整理与实践](https://gtoad.github.io/2017/06/25/Android-Anti-Debug/)
    - [APK反逆向](https://github.com/gnaixx/anti-debug)

### 6.2 xman

1. jadx反编译发现内容主要在so中
2. IDA发现是动态注册
3. 用yang神的hook_RegisterNatives看一下
4. 找到偏移对应的native函数偏移,IDA直接跳转到对应函数(IDA可以尝试修改函数第一个参数类型为JNIEnv *)
5. 看native函数来猜一下整个流程,然后来trace验证一下
6. hook strcmp过验证 / 两种写入文件过验证

```xman.js
function hook_MyApp() {
    var MyApp = Java.use('com.gdufs.xman.MyApp')
    MyApp.m.value = 1

    var Toast = Java.use('android.widget.Toast')
    Toast.makeText.overload('android.content.Context', 'java.lang.CharSequence', 'int').implementation = function(arg0, arg1, arg2) {
        console.log('arg2:', arg1)
        var ret = this.makeText(arg0, arg1, arg2)
        return ret
    }
}


function if_module_existed(module_name_pattern) {
    var modules = Process.enumerateModules()
    var module = null
    var existed = false

    for (var i = 0; i < modules.length; i++) {
        module = modules[i]

        if (module.name.indexOf(module_name_pattern) != -1) {
            if (!existed) {
                console.log('search module_name_pattern[' + module_name_pattern + '] found:')
                console.log()
                existed = !existed
            }
            console.log('module.name = ' + module.name)
            console.log('module.base = ' + module.base)
            console.log()
        }
    }
    return existed
}


function hook_libc_so() {
    var libc_so_name = "libc.so"
    var existed = if_module_existed(libc_so_name)
    if (existed) {
        console.log(libc_so_name, 'is existed!')
    } else {
        console.log(libc_so_name, 'is not existed!')
    }

    var libc_so_address  = Module.findBaseAddress(libc_so_name)
    console.log("libc_so_address =", libc_so_address)

    var libc_function_name = 'strcmp'
    var libc_function_address = null

    var symbols = null

    symbols = Module.enumerateSymbolsSync(libc_so_name)

    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i]
        var name = symbol.name
        var address = symbol.address
        if ((name.indexOf(libc_function_name) >= 0) && (name.indexOf(".cpp") == -1)) {
            console.log('name:', name)
            console.log('address:', address)

            libc_function_address = address
        }
    }

    var libc_function = new NativeFunction(libc_function_address, 'int', ['pointer', 'pointer'])


    Interceptor.replace(libc_function_address, new NativeCallback(function(s1, s2) {
        var char_star_s1 = ptr(s1).readCString()
        var char_star_s2 = ptr(s2).readCString()

        if (char_star_s2 == 'EoPAoY62@ElRD') {
            console.log('char_star_s1 =', char_star_s1)
            console.log('char_star_s2 =', char_star_s2)
            return 0
        }

        var ret = libc_function(s1, s2)
        return ret
    }, 'int', ['pointer', 'pointer']))
}


function frida_write_reg(){
    var file = new File("/sdcard/reg.dat","w+")
    file.write("EoPAoY62@ElRD")
    file.flush()
    file.close()

    console.log('frida_write_reg')
}


// 主动调用so函数
function hook_libc_so_write_reg(){
    var fopen_address = Module.findExportByName("libc.so", "fopen")
    var fputs_address = Module.findExportByName("libc.so", "fputs")
    var fclose_address = Module.findExportByName("libc.so", "fclose")
    console.log('fopen_address =', fopen_address)
    console.log('fputs_address =', fputs_address)
    console.log('fclose_address =', fclose_address)

    var fopen = new NativeFunction(fopen_address, "pointer", ["pointer", "pointer"])
    var fputs = new NativeFunction(fputs_address, "int", ["pointer", "pointer"])
    var fclose = new NativeFunction(fclose_address, "int", ["pointer"])

    // 创建C的字符串,不能直接传入js的字符串
    var filename = Memory.allocUtf8String("/sdcard/reg.dat")
    var mode = Memory.allocUtf8String("w+")
    var file = fopen(filename, mode)
    var contents = Memory.allocUtf8String("EoPAoY62@ElRD")
    var fputs_ret = fputs(contents, file)
    fclose(file)

    console.log('fputs_ret =', fputs_ret)
    console.log('hook_libc_so_write_reg')
}


function main() {
    // Java.perform(hook_MyApp)

    // hook_libc_so()

    // frida_write_reg()

    hook_libc_so_write_reg()
}


setImmediate(main)

```

## 7. trace工具链

### 7.1 trace三件套

- trace三件套
    - [jnitrace: jni](https://github.com/chame1eon/jnitrace)
    - [frida-trace: libc & ...](https://frida.re/docs/frida-trace/)
    - strace: syscall

- 源码学习
    - [frida-hook-libart](https://github.com/lasting-yang/frida_hook_libart)

## 8. init_array开发和逆向自动化

完整项目在 8_Reflect.7z + hook_linker.js

- [init_array原理](https://www.cnblogs.com/bingghost/p/6297325.html)
- [Android逆向新手答疑解惑篇——JNI与动态注册](https://bbs.pediy.com/thread-224672.htm)
- [linker加载so(init_array) - hook linker](http://note.youdao.com/noteshare?id=38aa9a5ce7d03a9fd61a21d076eab219)
- [Android NDK中.init段和.init_array段函数的定义方式](https://www.dllhook.com/post/213.html)

### 8.1 init & init_array

/root/Codes/Reflect/app/src/main/cpp/native-lib.cpp添加

```
// 先init后init_array
// 编译生成后在.init段 [名字不可更改]
extern "C" void _init(void) {
    LOGI(".init");
}

// 编译生成后在.init_array段 [名字可以更改]
__attribute__((__constructor__)) static void dot_init_array() {
    LOGI(".init_array");
}
```

.init可以直接在IDA Function window / Exports 搜到

shift + F7 / IDA View中Ctrl + s: 双击函数可以找到init_array中定义的函数内容

- [32位so通过call_function加载init_array中的方法的流程#2228-2236](http://androidxref.com/6.0.1_r10/xref/bionic/linker/linker.cpp#2228)
- [IDA调试init_array,按文章下断点](https://www.cnblogs.com/bingghost/p/6297325.html)

/root/Codes/Reflect/app/build.gradle: android - defaultConfig - externalNativeBuild添加

```
// 编译32位库
ndk {
    // Specifies the ABI configurations of your native
    // libraries Gradle should build and package with your APK.
    abiFilters "armeabi-v7a"
}
```

- 32位(资料多)动态调试init_array(我这边没有debug成功)
    1. 找 Module - linker
    2. linker 中找 call_function(其中的参数就是.init和.init_array)
    3. __dl__ZL13call_functionPKcPFvvES0_ 中的 CODE XREF: __dl__ZL13call_functionPKcPFvvES0_+18↑j 后的 BLX R6(对应文章中的BLX R4)
    4. F7就是init_array

### 8.2 jnitrace 自吐 JNI_Onload & init_array

```
JNI_Onload:
jnitrace -m spawn -l libnative-lib.so com.example.reflect


获得如下打印:
    417 ms [+] JNIEnv->RegisterNatives
    417 ms |- JNIEnv*          : 0xf39312a0
    417 ms |- jclass           : 0x85    { com/example/reflect/MainActivity }
    417 ms |- JNINativeMethod* : 0xffae15c8
    417 ms |:     0xd8d62495 - dynamicallyRegisteredStringFromJNI()Ljava/lang/String;
    417 ms |- jint             : 1
    417 ms |= jint             : 0

    417 ms --------------------------------------------------Backtrace--------------------------------------------------
    417 ms |-> 0xd8d625e1: _ZN7_JNIEnv15RegisterNativesEP7_jclassPK15JNINativeMethodi+0x2c (libnative-lib.so:0xd8d59000)


dynamicallyRegisteredStringFromJNI对应的so中函数的偏移即: 0xd8d62495 - 0xd8d59000 = 0x9495


init_array(感谢hanbing老师的脚本):
没有支持arm64，可以在安装app的时候 adb install --abi armeabi-v7a 强制让app运行在32位模式

function LogPrint(log) {
    var theDate = new Date();
    var hour = theDate.getHours();
    var minute = theDate.getMinutes();
    var second = theDate.getSeconds();
    var mSecond = theDate.getMilliseconds()

    hour < 10 ? hour = "0" + hour : hour;
    minute < 10 ? minute = "0" + minute : minute;
    second < 10 ? second = "0" + second : second;
    mSecond < 10 ? mSecond = "00" + mSecond : mSecond < 100 ? mSecond = "0" + mSecond : mSecond;

    var time = hour + ":" + minute + ":" + second + ":" + mSecond;
    var threadid = Process.getCurrentThreadId();
    console.log("[" + time + "]" + "->threadid:" + threadid + "--" + log);

}

function hooklinker() {
    var linkername = "linker";
    var call_function_addr = null;
    var arch = Process.arch;
    LogPrint("Process run in:" + arch);
    if (arch.endsWith("arm")) {
        linkername = "linker";
    } else {
        linkername = "linker64";
        LogPrint("arm64 is not supported yet!");
    }

    var symbols = Module.enumerateSymbolsSync(linkername);
    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i];
        //LogPrint(linkername + "->" + symbol.name + "---" + symbol.address);
        if (symbol.name.indexOf("__dl__ZL13call_functionPKcPFviPPcS2_ES0_") != -1) {
            call_function_addr = symbol.address;
            LogPrint("linker->" + symbol.name + "---" + symbol.address)

        }
    }

    if (call_function_addr != null) {
        var func_call_function = new NativeFunction(call_function_addr, 'void', ['pointer', 'pointer', 'pointer']);
        Interceptor.replace(new NativeFunction(call_function_addr,
            'void', ['pointer', 'pointer', 'pointer']), new NativeCallback(function (arg0, arg1, arg2) {
            var functiontype = null;
            var functionaddr = null;
            var sopath = null;
            if (arg0 != null) {
                functiontype = Memory.readCString(arg0);
            }
            if (arg1 != null) {
                functionaddr = arg1;

            }
            if (arg2 != null) {
                sopath = Memory.readCString(arg2);
            }
            var modulebaseaddr = Module.findBaseAddress(sopath);
            LogPrint("after load:" + sopath + "--start call_function,type:" + functiontype + "--addr:" + functionaddr + "---baseaddr:" + modulebaseaddr);
            if (sopath.indexOf('libnative-lib.so') >= 0 && functiontype == "DT_INIT") {
                LogPrint("after load:" + sopath + "--ignore call_function,type:" + functiontype + "--addr:" + functionaddr + "---baseaddr:" + modulebaseaddr);

            } else {
                func_call_function(arg0, arg1, arg2);
                LogPrint("after load:" + sopath + "--end call_function,type:" + functiontype + "--addr:" + functionaddr + "---baseaddr:" + modulebaseaddr);

            }

        }, 'void', ['pointer', 'pointer', 'pointer']));
    }


}

setImmediate(hooklinker)
```

### 8.3 frida-trace 对 任意函数的 trace

```
# -f 启动有时候so没加载会hook不到
frida-trace -H 192.168.50.42:8888 -I libnative-lib.so com.example.reflect

# 基于单个函数地址的hook
frida-trace -H 192.168.50.42:8888 -a libnative-lib.so\!0x9495 com.example.reflect

# 几种trace工具的文档和源码去了解一下
```

## 9. C hook & CModule

完整项目在 9_libnative-lib.so + 9_invoke_so.js + 9_cmodule.js + 9_explore.js + 9_explore.c

### 9.1 参数打印和构造

- Frida JNI相关函数
    1. jstring是没法直接打印的,java vm中只能转化为char*后打印,和native编程的流程一致
    2. jni的基本类型,要通过调用jni相关api转化为c++对象才能打印和调用(参数构造可以java.vm.xx/直接获取hook的参数进行传递),因为Interceptor(相当于在libart.so中执行)没有在java.perform里,所以不能使用java的方法
- NativePointer + Memory

### 9.2 Frida调用C++编写的so

invoke_so.js

```
/*
 * frida -H 192.168.50.42:8888 -f com.android.settings -l agent/invoke_so.js --no-pause
 */
function invoke_so() {
    /**
     * https://github.com/lasting-yang/frida_hook_libart
     * 导入AntiFrida的libnative-lib.so在settings中执行
     * root@Nick:~# adb push Downloads/libnative-lib.so /data/local/tmp
     * root@Nick:~# adb shell su -c "cp /data/local/tmp/libnative-lib.so /data/app/libnative-lib.so"
     * root@Nick:~# adb shell su -c "chown 1000.1000 /data/app/libnative-lib.so"
     * root@Nick:~# adb shell su -c "chmod 777 /data/app/libnative-lib.so"
     * root@Nick:~# adb shell su -c "ls -al /data/app/libnative-lib.so"
     */

    var module_libnative_lib_so = Module.load("/data/app/libnative-lib.so")
    var export_function_name = "_Z17detect_frida_loopPv"
    var export_function_address = module_libnative_lib_so.findExportByName(export_function_name)
    console.log('export_function_address =', export_function_address)

    var detect_frida_loop = new NativeFunction(export_function_address, 'pointer', ['pointer'])
    var void_star = Memory.allocUtf8String("hello")
    detect_frida_loop(void_star)
    // adb logcat 可以看到开始Frida端口检测
}


function main() {
    invoke_so()
}


setImmediate(main)

```

### 9.3 CModule

- [CModule](https://frida.re/news/2019/09/18/frida-12-7-released/)

不依赖架构并且不用编译为so直接可以将C代码注入

```shell
root@Nick:~# frida -H 192.168.50.42:8888 com.android.settings
[Remote::com.android.settings]-> var cm = new CModule('int add(int a, int b) { return a + b; }')
[Remote::com.android.settings]-> cm
{
    "add": "0x779d1d5000"
}
[Remote::com.android.settings]-> var add = new NativeFunction(cm.add, 'int', ['int', 'int'])
[Remote::com.android.settings]-> add(1, 2)
3

```

cmodule.js

```
/**
 * 1.
 * frida -H 192.168.50.42:8888 com.android.settings -l agent/cmodule.js --runtime=v8
 * 
 * 2.
 * root@Nick:~/Downloads# ./frida-server-12.8.0-linux-x86_64
 * root@Nick:~# mousepad cmodule.md
 */
function open_1() {
    // 无法保存文件内容了!
    const m = new CModule(`
    #include <gum/guminterceptor.h>
    
    #define EPERM 1
    
    int
    open (const char * path,
          int oflag,
          ...)
    {
      GumInvocationContext * ic;
    
      ic = gum_interceptor_get_current_invocation ();
      ic->system_error = EPERM;
    
      return -1;
    }
    `)
    
    const openImpl = Module.getExportByName(null, 'open')
    console.log('openImpl = ', openImpl)
    
    Interceptor.replace(openImpl, m.open)
    
}


function open_2() {
    const openImpl = Module.getExportByName(null, 'open')

    Interceptor.attach(openImpl, new CModule(`
      #include <gum/guminterceptor.h>
      #include <stdio.h>
    
      void
      onEnter (GumInvocationContext * ic)
      {
        const char * path;
    
        path = gum_invocation_context_get_nth_argument (ic, 0);
    
        printf ("open() path=\\"%s\\"\\n", path);
      }
    
      void
      onLeave (GumInvocationContext * ic)
      {
        int fd;
    
        fd = (int) gum_invocation_context_get_return_value (ic);
    
        printf ("=> fd=%d\\n", fd);
      }
    `))
}


function open_3() {
    const openImpl = Module.getExportByName(null, 'open')

    Interceptor.attach(openImpl, new CModule(`
            #include <gum/guminterceptor.h>

            extern void onMessage (const gchar * message);

            static void log (const gchar * format, ...);

            void
            onEnter (GumInvocationContext * ic)
            {
                const char * path;

                path = gum_invocation_context_get_nth_argument (ic, 0);

                log ("open() path=\\"%s\\"", path);
            }

            void
            onLeave (GumInvocationContext * ic)
            {
                int fd;

                fd = (int) gum_invocation_context_get_return_value (ic);

                log ("=> fd=%d", fd);
            }

            static void
            log (const gchar * format,
                ...)
            {
                gchar * message;
                va_list args;

                va_start (args, format);
                message = g_strdup_vprintf (format, args);
                va_end (args);

                onMessage (message);

                g_free (message);
            }
            `,
            {
                onMessage: new NativeCallback(messagePtr => {
                    const message = messagePtr.readUtf8String()
                    console.log('onMessage:', message)
                }, 'void', ['pointer'])
            }
        )
    )
}


function open_4() {
    const calls = Memory.alloc(4)

    const openImpl = Module.getExportByName(null, 'open')

    Interceptor.attach(
        openImpl,
        new CModule(`
                #include <gum/guminterceptor.h>

                extern volatile gint calls;

                void
                onEnter (GumInvocationContext * ic)
                {
                    g_atomic_int_add (&calls, 1);
                }
            `,
            { calls }
        )
    )

    setInterval(() => {
        console.log('Calls so far:', calls.readInt())
    }, 1000)
}


function main() {
    // open_1()

    // open_2()

    // open_3()

    open_4()
}

setImmediate(main)

```

### 9.4 Frida C + JS 混合编程

- [Frida C + JS 混合编程](https://frida.re/docs/javascript-api/)

root@Nick:~/Codes/github/others/frida-agent-example# frida -p 0 -l agent/explore.js -C agent/explore.c

explore.js

```
console.log('Hello from Javascript')

var counter = Memory.alloc(4)
var bump = null
cs.counter = counter

rpc.exports.init = function () {
    bump = new NativeFunction(cm.bump, 'void', ['int'])
}

```

explore.c

```
extern int counter;

void init(void) {
    frida_log("Hello from C");
}

void bump(int n) {
    counter += n;
}

```

## 关于逆向学习的几个观点

1. 开发是核心，以开发的眼光看待问题
2. 学会hook大法之后trace是几乎不需要手动调的
3. 降维打击,我就是系统
4. 先抄大佬的代码学习

QA:
有时候-f so还没有加载hook不到,可以延迟hook吗?(但不能attach,有时候attach已经错过调用时机了)
