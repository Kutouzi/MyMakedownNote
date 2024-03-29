## c++笔记-->STL

STL是c++语言提供的标准模板库，内含有算法、容器、迭代器实用工具。头文件有<algorithm>、<deque>、<functional>、<iterator>、<list>、<map>、<memory>、<numeric>、<queue>、<set>、<stack>、<utility>

### 容器

头文件有<list>、<map>、<deque>、<queue>、<set>、<stack>

这里提到的容器都在堆上分配内存（即使它叫stack，也是在堆上）

这里提到的容器都是浅拷贝，所以提供的对象不能涉及到指针。如果对象内涉及到指针，会发生只拷贝了指针，没拷贝指向的内容。为了解决这个问题，所有涉及指针的对象必须重载=号和拷贝构造函数，把这两个写为深拷贝

这里提到的容器不会抛出异常，没有正确使用会直接崩溃

这里提到的容器除了queue和stack之外都有迭代器，除了vector和deque之外都不可以随机存储

这里提到的容器都有clear函数，用来将容器内的内容置空，但不释放占用的内存

|              | vector | list |  deque   | queue | set  | map  | stack |
| ------------ | :----: | :--: | :------: | :---: | :--: | :--: | :---: |
| **数据结构** |  列表  | 链表 | 双向列表 | 队列  | 集合 | 字典 |  栈   |
| **增删速度** |   +    | +++  |    +     |   +   |  -   |  -   |   +   |
| **搜索速度** |   ++   |  --  |    +     |   +   |  ++  | +++  |  ---  |

#### vector

- 简介：vector是一个列表（是java中的arraylist），本质是一个动态的数组
- 优点：尾部存取快，访问元素极快，可以下标访问
- 缺点：中间插入效率低
- 注意：
  1. 插入元素是复制值插入
  2. 触发动态扩容时会发生拷贝（因此构造容器尽量指定长度，否则默认只有4字节）
  3. 删除元素不会有动态缩减
  4. 插入删除操作时迭代器失效，之前获取的迭代器要重新获取一遍

技巧1：收缩空间的方法，原理是用旧的对象重新构造一个新的对象，也会不可避免发生复制

```c++
原对象.resize(收缩后的大小);  //resize后不会释放删减空间的内存
vector<类型>(原对象).swap(原对象);  //交换匿名对象和原对象指针
```

技巧2：如果知道大概需要多少，可以预留空间，减少拷贝复制

```c++
原对象.reserve(预留的大小);
```

#### string

- 简介：string是一个封装了char*的类
- 优点：不用管理内存，提供字符串的处理函数
- 缺点：只支持ascii码，在堆分配会降低效率
- 注意：
  1. 短字符串和长字符串储存位置不同，短字符串储存在栈，长字符串在堆，加了const在常量区
  2. 当性能要求极高时，要使用char*来手动管理分配空间

技巧1：如果是c++17以上，字符串分割、拷贝可以用string_view提升性能，见后来的特性篇

#### deque

- 简介：deque是一个双向列表，本质是一个多段连续空间拼接成的，它有个节点控制器来连接
- 优点：可以前后插入删除，可以下标访问
- 缺点：中间插入效率低，排序效率低

#### stack

- 简介：stack是一个先进后出的容器即栈
- 优点：特殊情况才使用
- 缺点：不可以遍历，不可以从栈顶外的地方存取，不可以随机访问
- 注意：
  1. 如果要获取元素需要top后再pop，否则只是弹出去而已
  2. 在pop前应检查栈是否为空

#### queue

- 简介：queue是一个先进先出的容器即队列
- 优点：特殊情况才使用
- 缺点：不可以从队尾出，不可以从队头进，不可以遍历，不可以从中间存取，不可以随机访问

#### list

- 简介：list是一个链表（是java中的linklist）
- 优点：添加删除元素极快
- 缺点：不可下标访问，访问元素慢，不可以随机访问
- 注意：
  1. 节点除了数据还包含一个指针，这个指针指向下一个节点，最后节点指针nullptr

#### set

- 简介：set是红黑树实现的一个不允许重复元素的集合，multiset是红黑树实现的一个可以重复元素的集合，unordered_set才是数据结构中的简单set
- 优点：查找效率比较高，set和multiset会自动排序，特殊情况使用
- 缺点：不可以直接修改元素
- 注意：
  1. 存储对象需要指定排序方法
  2. 要修改元素需要先删掉后添加，让程序整理一次红黑树结构

技巧1：对象指定排序

``` c++
/*要一个比较的类*/
class Compare{
    public:
    bool operator()(对象类型 对象,对象类型 对象){
        return 对象.属性 > 对象.属性;
    }
}
set<对象类型,Compare> s; //再把对象类型和比较类都传进去就可以
```

#### pair

- 简介：pair是两个数据结合成一对的结构
- 优点：比结构体简单
- 缺点：需要以位置进行访问

#### map

- 简介：map是红黑树实现的一个不可重复键的键值对结构，multimap是红黑树实现的一个可重复键的键值对结构，unordered_map才是数据结构中的简单map
- 优点：查找效率比较高，map和multimap会自动排序，特殊情况使用
- 缺点：不可直接修改键
- 注意：
  1. 要修改键需要先删掉后添加，让程序整理一次红黑树结构

技巧1：指定排序

``` c++
/*要一个比较的类*/
class Compare{
    public:
    bool operator()(对象类型 对象,对象类型 对象){
        return 对象.属性 > 对象.属性;
    }
}
map<对象类型,对象类型,Compare> s; //再把对象类型和比较类都传进去就可以
```



### 函数对象（谓词）

头文件有<functional>

函数对象把对象的()重载，使得对象可以像函数调用一样使用

里面的适配器都不建议使用，不符合设计原理

### 函数转函数对象

```c++
int 函数名(参数,参数,...){
    
}
auto a = bind(函数名,值,值,...);
//因为绑定了，所以不需要参数就可以调用
a();
```

#### 绑定适配器

将给函数增加参数，但不改变函数本身

```c++
bind2nd(函数名,新参数);
```

#### 取反适配器

将函数的结果取反

```c++
not1(函数调用);
```

#### 函数对象适配器

将函数转化为函数对象

```c++
ptr_fun(函数名);
```

获取函数对象的调用

```c++
mem_fun(&类名::函数名);  //如果是对象指针用这个
mem_fun_ref(&类名::函数名);  //如果是对象用这个
```

#### 引用

在函数对象中传入参数会值传递，而不是引用传递，所以如果需要传递引用要使用ref，比如线程和bind就需要用

如果是常量就用cref

```c++
void task(int&& value,int& ret){
    ret = value;
}
int main(){
    int ret = 0;
    //函数对象，要用ref传递引用，否则将值传递
    std::thread mythread(task,1,std::ref(ret));
}
```

### 算法

头文件有<algorithm>

注意：

1. 对容器增删迭代器失效的问题
2. 如果要查找存放对象的容器，要重载对象的==运算符

### sort

对容器进行排序

### partial sort

返回容器中最大或最小的N个元素

#### find

普通查找，需要提供开始结束迭代器和一个回调函数，返回相邻元素的第一个位置的迭代器。没找到会返回结束迭代器

```c++
find(容器.begin,容器.end,找的值,回调函数);
```

#### adjacent find

查找相邻重复元素，需要提供开始结束迭代器和一个回调函数，返回相邻元素的第一个位置的迭代器。没找到会返回结束迭代器

```c++
adjacent_find(容器.begin,容器.end,找的值,回调函数);
```

#### binary search

二分查找，需要提供开始结束迭代器和一个回调函数，返回相邻元素的第一个位置的迭代器。没找到会返回结束迭代器

```c++
binary_search(容器.begin,容器.end,找的值,回调函数);
```

#### count

统计个数

#### count if

按条件统计个数

### 迭代器

#### transform

将一个容器内容搬运到另一个容器内，回调函数写搬运到新容器时要做的事情

```c++
transform(容器.begin,容器.end,新容器.begin,回调函数);
```

#### foreach

一个普通的迭代器

```c++
for_each(容器.begin,容器.end,回调函数);
```

