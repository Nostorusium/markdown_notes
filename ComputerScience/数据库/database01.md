# 数据库从入门到放弃

## 第一章-数据库系统概论

### DB和DBMS

众所周知,database,数据库,字面含义为承载着数据的基座
它只是承装数据的容器
当我们讨论数据库时,我们在讨论什么?
这一指代应当是:数据库这一容器本身与其内部的数据,而非Oracle,MySQL一类软件.
后者称之为DBMS,我们也通过DBMS来操纵管理数据库.
在应用程序使用数据库时,DBMS很自然的作为了连接两者之间的桥梁

### 从系统角度看DBMS

数据库语言可分为两类:
- 数据操纵语言
- 数据定义语言

语言分别经过对应的编译器解析,再交给数据查询或数据定义引擎并执行
所谓DBMS,只不过是一个解析数据库语言并最终执行的系统

- 数据库存放在硬盘上,则需要**存储管理器**控制读写
- 数据需要载入内存,则需要实现**缓冲区管理**
- 要把对数据的读写命令转化为对内存页的命令,则需要**索引/文件和记录管理**

其中,前两条通常交由OS管理,而第三条则属于DBMS的范畴.

### 数据与数据模式

- 数据Data,有时也叫视图View,指在某种表现形式下表现出来的数据库中的数据
- 模式Schema,数据库中数据的结构性的描述

对于不同用户所看到的数据描述,称为外模式(External Schema),当前用户所看到的,概念模式的一部分
从全局角度所理解的数据模式,称为概念模式(Conceptual Schema),数据的结构描述与关联约束,DBMS所关注的重点
而存储在介质上的数据模式,称之为内模式(Internal Schema),如存储路径,存储方式,索引方式

> 当谈及视图,View,默认认为是外部视图
当谈及模式,Schema,默认认为是概念模式

**三层模式,两层映射**,即是数据库系统的**标准结构**
外部视图经由E-C映射,映射到全局视图(概念模式)
而全局视图经由C-I映射,映射到内部视图
(E,external;C,conceptual;I,internal)

数据有两个独立性
- 逻辑数据独立性
- 物理数据独立性

应用程序基于外部视图开发
当概念模式变化,可以不改变外部模式,只需要改变EC映射.
这样一来应用程序就无需因外部视图改变而改变
基于同样的道理,当内部数据变化也可以不改变概念模式,而只需改变CI映射

### 数据模型

模式是对数据的结构抽象,而**模型**则是对**模式**的结构抽象
以**关系模型**为例:
所有模式都可以抽象为**表**,Table的形式 每一个具体的模式都是拥有不同列名的表
关系模型还要说明对这种表形式的数据有哪些操作和约束.

我们所看到的"表",即关系模型.
而关系模型的模式,则应是类似这样的东西:Student(sid char(8),sname char(10)...)

### 发展

- 文件系统与数据库
  在早期,用户直接使用操作系统来操纵磁盘上的数据集合.
  操作系统在管理信息时,为了让用户和硬件隔离开,提出了以**文件管理数据**的思想.
  以文件为数据的基本单位.
  数据库系统的一个重要发展便是从**文件系统**过渡到了**数据库**.

  文件系统向用户提供了操作接口,提供不同的存取方法,支持对文件的增删改查操作,数据存取以**记录**为单位.
  而这个过程中数据和程序紧密结合,数据的组织和语义紧密依赖处理该文件的应用程序.
  如果数据结构发生变化,则应用程序也必须修改.文件与文件之间没有联系,共享性差,冗余度大

  DBMS提供了一套语言给用户,数据库语言被DBMS编译并解析,由DBMS通过内核来执行
  至此,应用程序开发者不再需要直接与操作系统交互,这种隔离极大地方便了程序编写
  同时,数据存取也可以以**数据项**和**记录集合**为单位
- 关系数据库
  层次模型的树形结构与网状模型的图结构,都依赖复杂的指针系统来维系,结构描述复杂
  关系模型数据库中,数据之间的关联由Table属性值来表征,结构描述简单
  对数据的检索也不依赖于路径信息

### 关系数据库与更多

>**关系第一范式**
按行按列组织数据且数据项不可再分

在关系数据库中,数据项不可再分,这意味着不允许出现复合属性
即某一属性下不得有更细分的属性,一行元组内也不得出现多值属性

其他类型
- **对象-关系数据库**可以支持不满足关系第一范式的数据项,允许复合属性与多值
以对象来封装待分解的数据项
- 面向对象数据库支持复杂的数据类型,封装与抽象数据结构,支持面向对象的一些特性
- **XML数据库** 数据库的另一种形式,称半结构化数据库,在互联网世界得到广泛应用

多种多样的数据库可以开放互连.
应用程序可以通过接口**ODBC**(Open Database Connectivity)**开放数据库互连接口**(JAVA环境下有JDBC),由ODBC与不同DBMS的驱动程序交互
这样用户程序所写的数据库语句就经由ODBC转化为不同数据库中的数据库语言并向后执行

## 第二章-关系模型

一个关系(relation)就是一个Table.
关系模型就是处理Table的,他描述了三个部分:
- DB各种数据的基本结构形式:**Table/Relation**
- 表与表之间可以发生的操作:**关系运算**
- 操作应遵循的约束条件:**完整性约束**

即研究怎么建表,怎么操作表,与怎么约束表操作

关系模型三要素:
- 基本结构为Relation/Table
- 基本操作 Relation Operator
  并UNION,差DIFFERENCE,广义积PRODUCT,选择SELECT(ION)
  投影PROJECT(ION),交INTERSECT(ION),链接JOIN,除DIVISION
- 完整性约束
  实体,参照,用户自定义完整性约束

### 关系的严格定义

如何严格的定义**表**：
1. 首先定义列的域Domain,D
   即属性的"取值范围",如整数的集合,字符串的集合,全体学生的集合
   域是一组值的集合,他们应该有相同的数据类型
2. 其次定义行,即元组,与**笛卡尔积**
   当行域确定下来,那么所有元组的可能情况就确定下来
   一组域的笛卡尔积表示这几组域可能组合成的所有元组情况
   
   >一组域的笛卡尔积
   $D_1 \times D_2 \times ... \times D_n = \lbrace (d_1,d_2,...,d_n)| d_i \in D_i,i=1,...,n \rbrace$

   笛卡尔积的每个元素 $d_1,d_2,...,d_n$ 称一个n-元组(n-tuple)
   笛卡尔积就是**所有可能的n-元组的集合**
   如果 $m_i$ 表示分量 $d_i$ 的**基数**(域中取值个数),那么笛卡尔积的基数即 $m_1*...m_n$

在笛卡尔积中,并不是每一个n元组都是有意义的
我们从中抽出有意义的组合,称之为**关系**,他是笛卡尔积的**子集**

关系模式用 $R(A_1:D_1,A_2:D_2,...,A_n:D_n)$ 表示
例如:Student(S# char(8),Sname char(10),Ssex char(2)...)

### 码的概念

- **候选码**/候选键(Candidate Key)是关系中的一个**属性组**,它的值能**唯一标识**一个元组
  可以类比唯一的id号,但在这里他可以成组出现.
  > 候选码举例
  > 学生(S#,sname,sage,sclass) 关系中,S#能唯一标识一个学生元组
  > 选课(S#,C#,sname,cname,grade) 关系中,仅靠S#不足以唯一标识一个选课元组
  > 一个学生可以选多门课,而不能重复选同一门课
  > 则其候选码是(S#,C#)的联合
  候选码可以是{S#},{S#,sname},{S#,sname,sage}...
  只要满足唯一标识的条件就认为是候选码.
  其中,包含在候选码中的属性是主属性

- **主码**/主键(Primary Key)
  当有**多个候选码**时,可以选定一个作为**主码**,DBMS以主码为主要线索管理关系中的元组;如选择{S#}作为学生表的主码

- **外码**/外键(Foreign Key)
  ~~若一个属性组不是当前关系得到候选码,而是另一个关系的候选码,称**外码**~~
  如果一个属性同时是其他关系的候选码,称**外码**.
  两个关系通常是通过**外码**链接的
  一个属性可以既是**主码**又是**外码**

### 完整性

- 实体完整性
  主码属性值不能为**空值**,我们认为null,非法值,无意义的值均是空值.
- 参照完整性
  关系中的外码取值**可以为空**,认为未分配
  但必须**存在**
- 用户自定义完整性
  用户针对具体应用所定义的约束条件,如姓名在5个汉字以内

实体完整性和参照完整性由DBMS自动支持

## 第三章-关系代数

使用对关系的运算来表达查询.
关系模型有5个基本运算动作:**并,差,积,选择,投影**
SQL语言的本质是一个复杂的关系运算组合,**DBMS**解释这种组合并按照次序调动基本动作

### 并相容性

**并相容性**指参与运算的两个关系与相关属性之间要有一定的对应性,可比性或关联性
**并,差,交**操作需要满足并相容性

定义如下:
- 关系R与关系S的**属性个数**相同
- 对于第i个属性,关系R中的Ri与关系S中的Si的**域相同**

下面给出需要并相容性的关系运算
- **并Union**
  R∪S指把两个关系合并成一个关系,并去掉重复的元组
  >R为吃薯条的人 S为吃薯饼的人
  >则R∪S为吃薯条或吃薯饼的所有人
- **差Difference**
  R-S表示出现在R中但不出现在S中的元组
  R-S与S-R是不同的
  >R为吃薯条的人 S为吃薯饼的人
  >则R-S为吃薯条但不吃薯饼的人
  >S-R为吃薯饼但不吃薯条的人
  >*注:建议R-S与S-R反省一下自己*
- **交Intersection**
  R∩S表示同时出现在R与S中的元组

### 其他关系运算

- **广义笛卡尔积CartesianProduct**
  RxS表示R与S中的元组进行所有可能的拼接
  >R为住在下北泽的人 S为整天睡大觉的人
  >RxS则表示一个混乱的大表
  >广义积通常没有直接意义
  >进行广义积通常是因为一个检索涉及到多个表
  >我们将这些表拼接起来再进行检索
  RxS与SxR是相同的(行列无关性)
- **选择Select**
  给定一个关系R与一个条件cnd
- **投影Project**
  在给定关系中选出几个属性列,重复元组应去掉
- **交Intersection**
  需要并相容性;
  R∩S表示同时出现在R与S中的元组
- **θ连接Theta Join**
  表示在R于S的笛卡尔积中取出满足条件的元组
  其中条件为R中属性与S中属性的比较关系
  这要求属性应有可比性
  我们可以认为这是在笛卡尔积的基础上做属性上的条件选择
- **等值连接**
  当连接条件中运算符为"="时就是**等值连接**
- **自然连接**
  当连接没有条件时为**自然连接**
  自然连接的行为是:在笛卡尔积中选取**属性相同且值相等**的元组,并**去掉重复列**
  > 查询所有学生选课的成绩
  > 则可以 SC 自然连接 Course
  > 两个表中学号相同的元组自然的合并

### 除Division

常用于求解"查询...全部的,所有的"
R÷S,则要求S的属性集是R的属性集的真子集
S的属性要少于R
除法最为复杂,我们可以尝试逆向思考:
R÷S x S所得到的笛卡尔积,应都能在R中找到
那么要做除法运算 我们可以做以下步骤:
1. 保留R中与S属性列对应元组一致的行,其他删去
2. 保留R中存在于 S属性列与R中剩余属性列的笛卡尔积 的行
3. 删除S所对应的被剔除属性列
4. 删除重复行
这样可以保证除剩下的R÷S与S的广义积都在R中

但这非常的不直观,我们给出以下理解:
R÷S, S我们认为是需要**筛选**的属性,R÷S是在对R中一部分属性做筛选,只有满足S条件的才能留下来
且被筛选者必须满足S中的所有条件,只要有一条不符合就被筛去,这对应了理论中"不存在于笛卡尔积者剔除"

### 外连接Outer-Join

有个印象就行

### 关系演算

没见到考题,略
见到了再看

##  第四章-SQL

### 概述


SQL是集DDL(definition),DML(management),DCL(control)于一体的数据库语言

- 数据定义DDL
  - create 建立
  - alter  修改
  - drop 撤销
- 数据管理DML
  - insert 插入
  - delete 删除
  - update 更新
  - select 选择
- 数据控制DCL
  - grant 授权
  - revoke 撤销授权

**定义语句**

```
create database 数据库名
create table 表名(列名 数据类型 约束,...)
```

SQL-92数据类型:
- char(n): 固定长度字符串
- varchar(n): 可变长度字符串
- int: 有时也写作integer
- real: 有时也写作float(n)
- data: 如2024-1-1
- time: 如20:30:114
- 其他

主要有以下约束:
- primary key: 主键约束 每个表只能有一个
- unique: 唯一性约束,可以有多个
- not null: 非空约束,不允许为空值

```
create table Student(S# char(8) not null,sname char(10),ssex char(2),sage int,D# char(2),sclass char(6));
create table Course(C# char(3),cname char(12),chours int,credit float(1),T# char(3));
```

**管理语句**

- insert
  ```
  insert into 表名(可省略列名) values(值,值,...)
  insert into Student values('1919810','田所浩二','男',24,'01','114514')
  //若不省略列名,则修改顺序与给出的列名顺序保持一致
  ```
- select
  ```
  select 列名
  from 表名
  where 条件
  ```
  举例:
  ```
  select * from Student;
  select S#,sname,sage from Student where sname='田所浩二';
  select S# from Student
  where sname = '田所浩二' and (sage = 24 or sclass = '114514');
  ```
  DINTINCT可以保证无重复元组
  ```
  select DISTINCT S# from SC where score>80;
  ```
- order by
  ```
  order by  列名 asc/desc
  ```
- like
  模糊查询可以使用运算符like
  ```
  列名 like/not like "字符串"
   % 匹配任意多个字符
   _ 匹配单个字符
   \ 转义字符
  ```
  举例如下:
  ```
  select S#,sname from Student
  where sname like '田所%';
  select S#,sname from Student
  where sname not like '田_浩_';
  ```

### 多表联合查询

```
//表1与表2作笛卡尔积
select 列名
from 表1,表2,...
where 条件;
```
举例
```
// 按001号课程由高到低显示学生姓名
select sname from Student,SC
where Student.S# = SC.S# and SC.C# = '001'
order by score DESC;
// 按数据库课成绩从高到低显示学生姓名
select sname from Student,SC,Course
where Student.S# = SC.S#
and SC.C# = Course.C#
and cname = '数据库'
order by score DESC;
```
如果有重名,如两个表的属性重名/两个表重名,则需要别名(尤其是自己与自己时)
```
select 列名
from 表名1 as 表别名1,表名2 as 表别名2,...
where 条件
// as可以省略
```
举例
```
//薪水有差额的任意两个老师
select T1.tname as teacher1,T2.tname as Teacher2
from Teacher T1,Teacher T2
where T1.salary > T2.salary;
//既学过001号课又学过002号课的所有学生的学号
select S.S#
from SC S1,SC S2
where S1.S# = S2.S# and S1.C# = '001' and S2.C#='002';
//001课比002课成绩高的所有学生的学号
select S1.S#
from SC S1,SC S2
where S1.S# = S2.S# and S1.C#='001' and S2.C#='002'
and S1.score>S2.score;
```
注意同表的连接条件.
```
//求上题所有学生的姓名
select Student.sname
from SC S1,SC S2,Student
where S1.S# = S2.S# and Student.S# = S1.S#
and S1.C#='001' and S2.C#='002'
and S1.score>S2.score;

//求没学过田所浩二讲授课程的所有人
select sname from Student,SC,Course,Teacher
where
Teacher.tname = '田所浩二' and
Course.T# = Teacher.T# and
Course.C# = SC.C# and
SC.S# = Student.S#
```

关于多表查询的抽象性,我们可以这样认为:
1. 首先明确from确定了一个笛卡尔积的巨大表
2. 我们使用where来过滤不符合项,留下符合项
3. 使用恰当的where过滤条件使正确的项留下来

### 元组操作

- 插入insert
  ```
  //新增单一元组
  insert into Teacher values("114514","田所浩二","03","1250");
  //批量加入到某个表
  insert into St(S#,Sname)
  select S#,sname from Student
  where sname like '田所%';
  ```

- 删除delete
  ```
  delete from 表 where 条件

  //删除114514号同学所选的所有课程
  delete from SC
  where SC.sno = 114514;
  ```

- 更新update
  ```
  //将所有计算机系教师工资上调5%
  update Teacher
  set Salary = Salary*1.05
  where D# in (select D# from Dept where Dname = 'CS');
  ```

### 表/数据库操作

修正数据库操作alter
- add 新增列
- drop 删除完整性约束
- modify 修改列定义

```
//在学生表里加两列
alter table Student add status char[10],PID char[10];
//给sname数据类型char加点长度
alter table Student Modify sname char(114514);
//删除姓名取唯一值的约束
alter table Student drop Unique(sname);
```

其他
```
//撤除表/数据库
drop table 表名
drop database 数据库名
//指定当前使用的数据库
use 数据库名
//关闭当前数据库
close 数据库名
```

## 第五章-复杂查询

### 子查询

表达式 (not) in 子查询

```
//列出田所浩二和林檎的所有信息
select * from Student
where sname in ("田所浩二","林檎");
//等效于
select * from Student
where sname = "田所浩二" or sname = "林檎";

//列出选修了114514号课程多的学生的学号和姓名
select S#,sname from student
where S#  in (select S# from SC where C#=114514);

//列出没学过田所浩二讲授课程的所有学生的姓名
select sname from Studednt
where S# not in
(select S# from SC,Course,Teacher
where Teacher.tname = '田所浩二' and
SC.C# = Course.C# and
Teacher.T# = Course.T#);
```

内层子查询需要依靠外层查询的情况下称为**相关子查询**
```
//学过114514号课程的学生姓名
select sname from Student
where S# in (select S# from SC where S# = Student.S# and C#='114514');
//等价于 
select sname from Student,SC
where Student.S# = SC.S# and SC.C# = '114514';
```

### 谓词子查询some,all

some查询结果只要有一个满足就为真
all查询结果的所有值都满足才为真

```
//工资最低的教师
select tname from teacher
where salary <= all(select salary from teacher);

//找出114514号课成绩不是最高的所有学生的学号 取等号最高也会留下
select S# from SC
where C#=114514 and 
score < some(select score from SC where C#=114514);

//1919810号同学成绩最低的课程号
select C# from SC
where S# = 1919810 and
score <= all(select score from SC where S#=1919810);
```

### exists子查询

exists用于判断查询子句是否有记录
如果有则返回,并抛弃没有的
```
//选修了田所浩二主讲课程的所有学生的姓名
//exists可以不用,没有什么特别的功能

select sname from Student
where exists(
  select * from SC,Course,Teacher
  where Student.S# = SC.S# and
  SC.C#=Course.C# and
  Teacher.tname = '田所浩二' and
  Teacher.T# = Course.T# 
);
```

而not exists花样很多,他可以帮助查询"所有的","全部的"
```
//学过114514号教师讲过的所有课程的学生姓名
//等价于 不存在与{114514号教师讲的课 没学过}的学生
select sname from Student where not exists( //把没学的踢出去
  select * from Course where Course.T# = 114514 and 
  not exists ( //不存在,说明有人没学
    select * from SC where SC.S#=Student.S# and SC.C#=Cource.C#) //学了的人
  )
);
```

