## java笔记-->类的声明和对象的创建

对象创建的步骤为：

1. 申请空间，并且清空此块内存空间的值（在c语言不会清空，会留着上个程序所用的值）
2. 运行构造函数，初始化成员变量为源码内设置的值
3. 建立声明的标识符与对象的连接关系

java源码如下

```java
class T{
    int m = 8;
}
T t = new T(); //这条语句对应的jvm汇编码如下
```

jvm汇编码如下

```java
0 new #2 <T>  //1
3 dup
4 invokespecial #3 <T.<init>>  //2
7 astore_1 //3
8 return
```

