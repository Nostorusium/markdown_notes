# 标准模板库 Standard template Library

STL从广义上分为容器container,算法algorithm,迭代器iterator.
STL几乎所有的代码都采用模板类或模板函数,提供了更好的代码重用机会.

STL是一种泛型编程(generic programming).
面向对象变成关心的是编程的数据部分
而泛型编程关注的是算法,旨在编写独立于数据类型的代码.
模板让这一切成为了可能,但必须对元素进行精心的设计.

STL六大组件如下

- 容器:存放数据
- 算法:操作数据
- 迭代器:遍历容器各元素
- 仿函数:为算法提供更多策略
- 适配器:为算法提供更多参数接口
- 空间配置器:为算法和容器动态分配,管理空间

## 迭代器

理解迭代器是理解STL的关键所在.
模板使算法独立于数据类型,而迭代器使算法独立于使用的容器类型.

考虑一个find()函数,他的使用场景可以是在一个数组中遍历所有元素找到特定元素,也可以是遍历某个链表找到想要的结点,而这一切都与类型强关联,你需要为不同的情况编写不同的find().
而泛型编程旨在使用同一个find()来处理数组,链表或者其他任何容器类型.

模板提供了存储在容器中的数据类型的通用表示,除此之外我们还需要遍历容器中各值的通用表示.
迭代器iterator就是这样的通用表示.对于某些无法线性遍历的非线性数据结构,下标是无意义的,就更需要迭代器了.
为了实现find(),他应该具有以下特征:

- 能够解引用,如果p是一个迭代器,则应该可以对*p进行定义
- 能够把一个迭代器赋给另一个
- 可以与另一个迭代器比较,查看是否相等
- 能够使用迭代器遍历容器中的所有元素,定义++p与p++

每个容器类如vector,list,deque等都实现了相应的迭代器类型.
对于其中的某个类,迭代器可以是指针,或者是对象.实现方式各异.
但不管实现方式如何,迭代器都会提供所需的操作,如*,++
每个容器类都有一个超尾标记,当迭代器递增到超越容器的最后一个值后,这个值会被赋给迭代器.
他的位置是最后一个元素再往后一个.
每个容器类都有begin()和end()方法,返回容器的第一个元素和超尾位置的迭代器.

使用容器类无需知道它是如何实现的,也无需知道超尾是如何实现的.
下面是一个打印vector<double>容器内元素的例子

```
vector<double>::iterator p = scores.begin();
for(;p!=scores.end();p++){
    cout<<*p<<endl;
}
```
rbegin()与rend()方法可以反向遍历.

### 用法

- begin(), rbegin()
- end(), rend()
- +int,向后移动,支持自增
- -int,向前移动,支持自减
- 迭代器-迭代器,得到两个迭代器之间的距离
- prev(it), 返回it的前一个迭代器
- next(it), 返回it的后一个迭代器

对于不同的容器,由于其结构特性,上面的用法不一定都存在.
比如set就不能相减求距离.

### 常见问题

- .end() 与 .rend()指向的位置通常无意义.
- 如无必要,不要用迭代器操作容器:遍历与访问没有关系
  一个常见的例子是删除vector元素会导致容器内迭代器的位置发生变化

## 常用容器

### Vector

#### 构造

```
vector<int> array;
vector<int> array(100); //长度为100
vector<int> array(100,114514); //初始值为114514
vector<vector<int>> dp(5,vector<int>(6,114514)); //五行六列初值为114514的二维数组
```

以下写法不推荐

```
// dont do this
vector<int> array[100]; //奇怪 别写
```

#### 类方法

- push_back() 推入
- pop_back() 删除
- size() 返回大小
- clear() 清空数组
- empty() 返回布尔值
- resize(length,value) 调整长度,可设初值
  延长会赋初值,缩短会抛弃后面元素
- reserve() 预留长度
- emplace_back() 以构造函数形式推入新元素,避免了复制

#### 注意事项

- vector的数据存储在堆空间,不会爆栈
- vector在堆内存
  如果被填满则需要调整堆重新分配,有性能损失.
  如果能提前确定长度,那么应当直接指定长度.
  可以用构造函数指定长度,或者使用reserve()
- size_t溢出
  .size()的返回值类型是size_t,注意32位编译器和64位编译器下size_t取值范围不同导致溢出

#### 为什么不用Array

>std::array与C风格的array几乎一模一样,但它提供了.size(),并且支持越界检查,越界的语句不会有任何作用.
其性能和原生数组差不多.
尽管.size()会返回其长度,但实际上std数组并不实际存储数组长度,size()会把泛型中的"5"直接返回,所以并没有占据额外的空间.
当你想使用一个函数打印数组时,若使用普通数组则把数组长度作为参数写入;
而使用std数组,但仍需要在类型中的<>写入其长度.在这方面std数组没有优势;
但std数组仍有很大的优势,它支持迭代器,支持sort排序,同时维持了size.
所以尽可能的使用std数组,它的性能与原生数组相差无几

std::array和原生数组类似,在堆上分配内存,也不支持动态增长.
考虑到vector在堆上分配,std::array的性能通常优于vector,因为在栈上申请内存的开销几乎为0.

### Stack

#### 构造与用法

```
stack<int> stack;
stack.push(114);
stack.push(514);
stack.pop();
int value = stack.top();  //取栈顶
int size = stack.size();  //大小
boolen b = stack.empty(); //判空
```
#### 注意事项

使用std::stack就不需要手写栈了.
另外Vector也可以当栈用.
vector::back()取尾部元素,就相当于取栈顶.
vector::push_back()就相当于进栈,pop_back()就相当于出栈.

要注意stack不可访问内部元素

```
// WRONG!
for(auto value : stack){
  cout<<stack[i]<<endl;
}
```

### Queue

和栈基本一样,只不过先进先出,同样不能访问内部

#### 构造与用法

```
queue<int> queue;
queue.push(114);
queue.push(514);
queue.pop();
int front = queue.front();
int back = queue.back();
```

### 优先队列 priority_queue

换一个名字:堆,heap
提供常数时间的最大元素查找,对数时间的插入和提取.
底层原理是二叉堆.
进出队复杂度为 $O(\log(n))$
取堆顶复杂度为 $O(1)$

#### 构造与用法

priority_queue<类型,容器,比较器> pque;
比较器默认为less<类型>,是一个仿函数,可自定义.
容器指priority_queue底层用什么容器储存,默认是vector
需要自定义比较器的情况涉及重载,lambda表达式,此处不展开

```
//储存int的大顶堆
priority_queue<int> pque1;

//储存int的小顶堆
priority_queue<int,vector<int>,greater<int>> pque2; 

pque1.push(114514);
pque1.pop();
int a = pque1.top();
```

#### 注意事项

priority_queue仅堆顶可读且所有元素不可写.
如果要维护元素的有序性,每次向队列插入大小不定的元素或者从队列里取出大小最大/最小的元素k个,堆大小为n,
则使用快速排序有复杂度 $k·n\log(n)$
使用优先队列维护则 $k·\log(n)$

### 集合 Set

提供对数时间的插入删除查找.
底层是红黑树.

#### 集合三要素

- 确定性:一个元素要么在集合中,要么不在
- 互异性:一个元素只能出现一次
- 无序性:元素无序

Set满足确定性和互异性,元素有序(从小到大).
需要无序的情况可以使用unordered_set
multiset不满足互异性(任意次)和无序性(从小到大)

#### 构造与用法

```
set<int> set;
set.insert(114);
set.insert(514);
for(auto & ele:set){
  cout<<ele<<endl;
}

set.erase(114);

// 查找 返回一个迭代器
if(set.find(514)!=set.end()){
  cout<<"yes\n";
}


// 查询个数 由于互异性count返回值只能是1或0
int count = set.count(514);
int size = set.size();
set.clear();

// 遍历
for(set<int>::iterator it = st.begin();it!=set.end();++it){
  cout << *it << endl;
}
```

#### 注意事项

set虽然可以遍历,但只能使用迭代器遍历,不存在下标的概念.
且set的迭代器取到的元素是只读的(因为是const迭代器).
如果要修改则需要先erase再insert.

### 映射 Map

提供对数时间的有序键值对结构,底层是红黑树
增删改查均为 $\log(n)$

映射就好比是一个数组:

```
//int到int的映射
int map[114514];
map[0] = 1;
map[1] = 1;
map[2] = 4;
...
```

默认value是0

map提供了任意类型向任意类型的映射.

#### 映射性质

- 互异性:一个key只出现一次
- 无序性:key无序

map满足互异性,但不满足无序性(从小到大).
**unordered_map** 则都满足,底层是哈希表.
**multimap** 则都不满足,底层是红黑树

#### 构造与用法

```
map<int,int> mp1; // int -> int
map<string,int> mp2;

//map支持方括号运算符
mp1[2] = 1;
mp1[1] = 2;
mp1.erase(2);

if(mp1.find(2) != mp1.end()){ //find查询的是key
  cout << "yes" << endl;
}else{
  cout << "no" << endl;
}

// 迭代器遍历
for(map<int,int>::iterator it = mp.begin();it!=mp.end();++it){
  cout<< it->first << ' ' << it->second <<> endl; //用pair存的 打印
}
// 范围for遍历
for(auto&pr : mp){
  cout<< pr.first << ' ' << pr.second <<> endl;
}
```

count,clear,empty,size同其他容器

#### 注意事项

使用场景:维护特殊的映射.
```
// 统计每一个单词出现的次数.

map<string,int> times;
vector<string> words;
words.push_back("ruarua");
words.push_back("ruarua");
words.push_back("114514");

for(int i=0;i<words.size();i++){
  mp[words[i]]++;
}
```

当使用中括号访问一个不存在的key,会默认创建这个key并赋初值为0.

find()若找不到key,则返回尾迭代器.

### String

#### 使用

略

#### 注意事项

尾接字符串使用+=

```
string1 = string1 + "aaa";
string2 += "bbb";
```

先加再相等,由于复制会造成性能开销.而字符串长度可以很长,该开销可能很大.
而string的+=运算符经过优化会直接尾接字符串.

### 二元组 Pair

#### 构造与用法

```
pair<int,int> p1 = {1,2};          //C++11以后才有
pair<int,int> p2 = make_pair(1,2); //老式
cout<< p1.first << ' ' << p1.second << endl;  //访问
```



```
struct pair{
  int a;
  int b;
};
```

它与这样一个结构没什么区别.如果需要三元组,你可以pair套娃曲线救国.但最好别这么做.

## 常用算法

### swap

```
int a=1,b=2;
swap(a,b);
```

### sort

```
vector<int> array{1,9,1,9,8,1,0};
sort(array.begin(),array.end(),greater<int>());
```

比较器默认从小到大.
自定义比较器：

```
vector<pair<int,int>> array{{1,1},{4,5},{1,4}};
sort(array.begin(),array.end(),cmp);

bool cmp(pair<int,int> a,pair<int,int> b){
  //第二位从小到大
  //返回true表示 a b 顺序正确
  if(a.second != b.second)
    return a.second < b.second;
  //若第二位相同则第一位从大到小
  return a.first > b.first;
}
```

### lower_bound,upper_bound

二分查找对应元素,返回迭代器.
减去头迭代器转化成下标.

lower_bound(起点,终点,查找值);
lower_bound寻找小于等于给定值的第一个元素的迭代器
upper_bound寻找大于
若找不到返回尾迭代器

```
vector<int> array{1,1,4,5,1,4};
int pos = lower_bound(array.begin(),array.end(),5) - array.begin();
```

### reverse 翻转

```
vector<int> array{4,1,5,4,1,1};
reverse(array.begin(),array.end());

vector<int> array2{1,9,1,9,8,1,0};
reverse(array2.begin()+1,array2.begin()+5);
```

### 数学函数

```
cout << min(1,2);
// C++11
cout << max({1,4,5});

int x = abs(-5);  //绝对值
x = exp(2);       //e^x
x = log(3);       //lnx
x = pow(2,3);     //x^y
x = sqrt(2);      //√x
```