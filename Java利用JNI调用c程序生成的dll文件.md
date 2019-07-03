# Java利用JNI调用c程序生成的dll文件

关键字：JAVA JNI  java调用C  VC++6.0编译C代码成32位dll文件 

## 1背景说明
JNI(Java Native Interface,Java本地接口)

## 2具体流程
### 2.1编写Java文件
安装32位jdk，以及32位的eclipse（这是考虑到VC++6.0只能编译生成32位的dll文件），打开eclipse，新建工程prj_javaCallC，接着建立类com.xd.javaCallC.test.Simple1，内容如下：
```Java
package com.xd.javaCallC.test;
/**
 * Java调用C代码测试
 * @author pengfei
 * @date 2015-04-02
 */
public class Simple1 {
	/*
	 * 编写本地方法，没有具体实现，是一个抽象方法
	 * 其具体实现在C语言中
	 */
	public native void add();
}
```
### 2.2利用Java文件生成.h文件
若使用Eclipse，那么需将编译好的class文件放置在同Java源文件相同的目录下，使用myeclipse则不需要。接下来在doc命令窗口进入到Java文件所在目录，比如：

再利用javah命令javah -jni com.xd.javaCallC.test.Simple1为JNI生成.h头文件，如下：

这样会在工程根目录下的src目录下生成.h头文件com_xd_javaCallC_test_Simple1.h

注意：在执行javah命令时要输入完整的包名+类名（即输入完全限定名），否则以后可能会出错。
### 2.3编写C代码
打开VC++6.0，新建一个Win32 Dynamic-Link library工程，再选择“一个空的DLL工程”，比如：javaCallAddNums，把刚才用javah命令生成的头文件copy到C工程的根目录下，如下：

然后用编辑器打开此头文件，里面内容如下：

注意到框起来的部分，对应的是Java文件中的本地方法public native void add();。
在该C工程中新建一个C++源文件javaCallCTestOfc1.cpp，内容参考如下：

函数体中书写add方法的具体实现过程，这就解决了我们在Java代码中只使用抽象方法而将实现放在C中，还要注意，一定要引入javah命令生成的.h头文件。
注意：记得在C代码中释放内存，比如：

### 2.4编译C代码，完善配置
C代码完成之后，进行编译，F7编译工程发现缺少jni.h这个头文件。这个头文件可以在%JAVA_HOME%/include目录下找到。把这个文件拷贝到C++工程目录，继续编译发现还是找不到。原来是因为在我们刚刚生成的那个头文件里，jni.h这个文件是被 #include <jni.h>引用进来的，因此我们把尖括号改成双引号#include "jni.h"，继续编译发现少了jni_md.h文件，接着在%JAVA_HOME%/include/win32下面找到那个头文件，放入到工程根目录，F7编译成功。在Debug目录里会发现生成了javaCallCaddNums.dll这个文件。

### 2.5配置环境变量Path
Dll文件编译成功之后，需要配置环境变量Path，将dll文件所在目录的路径配置到Path环境变量中，比如：H:\计算机语言\c语言\C语言程序\C语言--试验--程序\javaCallCOfCTest1\Debug，接下来重启eclipse，让eclipse在启动的时候重新读取Path。
### 2.6 Java代码加载库文件并进行测试
至此只剩下用Java代码调用C代码了，在Java代码中添加静态块加载C动态库文件，
package com.xd.javaCallC.test;
/**
 * Java调用C代码测试
 * @author pengfei
 * @date 2015-04-02
 */
public class Simple1 {
	/*
	 * 加载C动态库文件
     * System.loadLibrary("javaCallCaddNums")的参数与生成的dll文件的名称一
     * 致       
	 */
	static{
		System.loadLibrary("javaCallCaddNums");
	}
	/*
	 * 编写本地方法，没有具体实现，是一个抽象方法
	 * 其具体实现在C语言中
	 */
	public native void add();
    /*
	 * 测试
	 */
	public static void main(String[] args) {
		Simple1 simple =new Simple1();
		simple.add();
	}
}

## 3 web中使用JNI
在web应用中，在系统环境变量Path中配置是不管用的，会报 java.lang.UnsatisfiedLinkError: no javaCallCOfCTest2 in java.library.path
这个异常，解决办法是先用
System.out.println(System.getProperty("java.library.path"));这条语句打印出java.library.path所代表的目录，比如：
F:\360Downloads\myeclipse.10\myeclipse32bit\install_myeclipse32bit\Common\binary\com.sun.java.jdk.win32.x86_1.6.0.013\bin;F:\360Downloads\Tomcat\tomcat32bit\install_tomcat32bit\apache-tomcat-6.0.39\bin
然后将编译好的dll文件copy至该目录下，比如：

这时，就没有必要将dll文件所在路径配置到Path环境变量中了。

## 4 使用JNI注意事项
### 4.1 内存释放
JNI基本数据类型不需要释放，例如：jint、jlong、jchar，需要释放的是引用类型的额数据，包括数组，比如：jstring、jobject、jobjectArray、jintArray、jclass、jmethodID等。
1)释放string
char * point=env->GetStringUTFChars(str,NULL); 
//用完释放内存
env->ReleaseStringUTFChars(str,point);
env->DeleteLocalRef(str);
2)   释放 类 、对象、方法
    env ->DeleteLocalRef(XXX);
	“XXX” 代表 引用对象
3)    释放 数组家族
	jobjectArray arrays = NULL;
	jclass jclsStr = NULL;
	jclsStr = env ->FindClass( "java/lang/String");
	arrays =env->NewObjectArray(len, jclsStr, 0);
 
	env ->DeleteLocalRef(jclsStr);  //释放String类
	env ->DeleteLocalRef(arrays); //释放jobjectArray数组
native method 调用 DeleteLocalRef() 释放某个 JNI Local Reference 时，首先通过指针 p 定位相应的 Local Reference 在 Local Ref 表中的位置，然后从Local Ref 表中删除该 Local Reference，也就取消了对相应 Java 对象的引用（Ref count 减 1）。
住：有可能其请检查调用的C代码是否进行了内存释放，比如：用malloc分配的内存要用free进行释放。
### 4.2 避免使用全局引用（global reference）
不要用static修饰变量，应使用局部引用。

## 5类型转换
在使用JNI调用C(C++)时会遇到类型转换的问题，先将本人所遇到的类型转换记录下来:
### 5.1 jobjectArray（JNI中的对象类型的数组） jstring（JNI中的字符串）
Java中传进来的是String[],比如：
public native void jniCallCEntranceFourteen(String strs[]);
那么，在JNI生成的头文件中是这样的，

将其放在.cpp文件中是这样的，

注意到，由java中传进来的字符串数组String []到JNI中变成了 对象数组jobjectArray，想要将其转成jstring，用JNI的API
env->GetObjectArrayElement(array,i);//array是数组jobjectArray，i表示数组下标。
jstring obja=(jstring)env->GetObjectArrayElement(array,i);
//这行代码的意思是将jobjectArray类型的数组array的第i号元素转换为jstring类型。

### 5.2 jstring（JNI中的字符串） char * （C语言中的字符串）
首先得明白，C语言中的定义字符串的一种方式为char * str=”abcd”;
用JNI的API
env->GetStringUTFChars(jstring,NULL);例如：
char* chars=(char*)env->GetStringUTFChars(obja,NULL);
//将jstring类型转换成char类型输出（char * chars代表C语言中的字符串）
一个详细代码：
JNIEXPORT void JNICALL Java_com_xtu_predict_base_JNICallCBase_jniCallCEntranceFourteen
  (JNIEnv * env, jobject obj, jobjectArray array)
{
	 int size=env->GetArrayLength(array);//得到数组的长度值
	 int i=0;
	 char * argvB[3];  //定义一个字符串数组（含3个字符串）
	 char * argvA[3];
	 for(i=0;i<size;i++)
	 {
		 jstring obja=(jstring)env->GetObjectArrayElement(array,i);
		 char* chars=(char*)env->GetStringUTFChars(obja,NULL);//将jstring类型转换成char类型输出
		 argvB[i]=chars;
	 }
	 char * point0="exe";
	 argvA[0]=point0;
	 argvA[1]=argvB[0];
	 argvA[2]=argvB[1];
	 printf("argvA[0]=%s\n",argvA[0]);
	 printf("argvA[1]=%s\n",argvA[1]);
	 printf("argvA[2]=%s\n",argvA[2]);
	 main_test(3,argvA);
	 printf("%s\n","xiaolin");
}

