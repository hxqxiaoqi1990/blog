---
title: java学习之类和对象的概念
tags: [java]
categories: java
date: 2020-09-07 11:24:44
---

# 版本介绍

Java 有三个版本，分别为 J2SE、J2EE和J2ME，以下是详细介绍。

**J2SE(Java 2 Platform Standard Edition) 标准版**

J2SE是Java的标准版，主要用于开发客户端（桌面应用软件），例如常用的文本编辑器、下载软件、即时通讯工具等，都可以通过J2SE实现。

J2SE包含了Java的核心类库，例如数据库连接、接口定义、输入/输出、网络编程等。

学习Java编程就是从J2SE入手。

**J2EE(Java 2 Platform Enterprise Edition) 企业版**

J2EE是功能最丰富的一个版本，主要用于开发高访问量、大数据量、高并发量的网站，例如美团、去哪儿网的后台都是J2EE。通常所说的JSP开发就是J2EE的一部分。

J2EE包含J2SE中的类，还包含用于开发企业级应用的类，例如EJB、servlet、JSP、XML、事务控制等。

J2EE也可以用来开发技术比较庞杂的管理软件，例如ERP系统（Enterprise Resource Planning，企业资源计划系统）。

**J2ME(Java 2 Platform Micro Edition) 微型版**

J2ME 只包含J2SE中的一部分类，受平台影响比较大，主要用于嵌入式系统和移动平台的开发，例如呼机、智能卡、手机（功能机）、机顶盒等。

在智能手机还没有进入公众视野的时候，你是否还记得你的摩托罗拉、诺基亚手机上有很多Java小游戏吗？这就是用J2ME开发的。

Java的初衷就是做这一块的开发。

注意：Android手机有自己的开发组件，不使用J2ME进行开发。

Java5.0版本后，J2SE、J2EE、J2ME分别更名为Java SE、Java EE、Java ME，由于习惯的原因，我们依然称之为J2SE、J2EE、J2ME。

# 类和对象

示例：

```java
public class Demo {
    public static void main(String[] args){
        // 定义类Student
        class Student{  // 通过class关键字类定义类
            // 类包含的变量
            String name;
            int age;
            float score;
            // 类包含的函数
            void say(){
                System.out.println( name + "的年龄是 " + age + "，成绩是 " + score );
            }
        }
        // 通过类来定义变量，即创建对象
        Student stu1 = new Student();  // 必须使用new关键字
        // 操作类的成员
        stu1.name = "小明";
        stu1.age = 15;
        stu1.score = 92.5f;
        stu1.say();
    }
}
```



`类`：class定义一个类

`对象`：在Java中，使用new关键字，就可以通过类来创建对象，这个过程叫做类的实例化，因此也称对象是类的一个实例。

`方法`：类的函数

`属性`：类的变量，成员变量也叫

特点：面向对象编程在软件执行效率上绝对没有任何优势，它的主要目的是方便程序员组织和管理代码，快速梳理编程思路，带来编程思想上的革新。

**类中的成分研究：**

​	类中有且仅有五大成分（五大金刚）

1. 成员变量
2. 成员方法
3. 构造器
4. 代码块
5. 内部类



# 方法注意事项

1. 方法只能定义在类中，不能再定义在方法内
2. 方法定义的顺序无所谓
3. 方法定义之后不会执行，如果希望执行，一定要调用：单独调用，打印调用，赋值调用
4. 如果方法有返回值，那么必须写`return 返回值`
5. return后面的返回值必须与方法的返回值类型一致
6. 对于一个void没有返回值的方法，不能写return后面的返回值，只能写return自己
7. 对应void方法中的最后一行return可以省略
8. 一个方法当中可以有多个return语句，但必须保证同时只有一个会被执行，两个return不能连写

# 内存划分

1. 栈（stack）：存放方法中的局部变量，方法的运行一定要在栈中。
   1. 局部变量：方法的参数，或方法{}的内部变量。
   2. 作用域：一但超出作用域，立刻从栈内存中消失。
2. 堆（heap）：凡是new出来的东西，都在堆当中。
3. 方法区（method area）：存储.class相关信息，包含方法信息。



# 数据类型

## 数据类型定义

| 数据类型 | 说明         | 占用内存 | 取值范围                                                     | 优先级 | 示例                       |
| -------- | ------------ | -------- | ------------------------------------------------------------ | ------ | -------------------------- |
| byte     | 字节型       | 1 bytes  | -128（-2^7）到 127（2^7-1）                                  | 1      | 3, 127                     |
| short    | 短整型       | 2 bytes  | -32768（-2^15）到 32767（2^15 - 1）                          | 1      | 3, 32767                   |
| int      | 整型         | 4 bytes  | -2,147,483,648（-2^31）到 2,147,483,647（2^31 - 1）          | 2      | 3, 21474836                |
| long     | 长整型       | 16       | -9,223,372,036,854,775,808（-2^63）到 9,223,372,036,854,775,807（2^63 -1） | 3      | 3L, 92233720368L           |
| float    | 单精度浮点型 | 4 bytes  | 1.4E-45 到 3.4028235E38                                      | 4      | 1.2F, 223.56F              |
| double   | 双精度浮点型 | 8 bytes  | 4.9E-324 到 1.7976931348623157E308                           | 5      | 1.2, 1.2D, 223.56, 223.56D |
| char     | 字符型       | 2 bytes" | \u0000（即为0）到 \uffff（即为65,535）                       | 1      | 'a', ‘A’                   |
| boolean  | 布尔型       | 1 bit    | 只有两个取值：true 和 false                                  | 无     | true, false                |

注：

- 后缀L，D，F不区分大小写
- E后的数字代表10的n次方
- 布尔型：true 等同于1，false 等同于0
- 字符型：数据只能是一个字符，由单引号包围
- 优先级：是数据类型转换时使用

## 数据类型转换

- 1. 不能对boolean类型进行类型转换。
- 2. 不能把对象类型转换成不相关类的对象。
- 3. 在把容量大的类型转换为容量小的类型时必须使用强制类型转换。
- 4. 转换过程中可能导致溢出或损失精度。

### 自动转换

```java
public class ZiDongLeiZhuan{
        public static void main(String[] args){
            char c1='a';//定义一个char类型
            int i1 = c1;//char自动类型转换为int
            System.out.println("char自动类型转换为int后的值等于"+i1);
            char c2 = 'A';//定义一个char类型
            int i2 = c2+1;//char 类型和 int 类型计算
            System.out.println("char类型和int计算后的值等于"+i2);
        }
}
```



### 强制转换

```java
public class QiangZhiZhuanHuan{
    public static void main(String[] args){
        int i1 = 123;
        byte b = (byte)i1;//强制类型转换为byte
        System.out.println("int强制类型转换为byte后的值等于"+b);
    }
}
```

# 访问控制符

**Java支持四种不同的访问权限：**

| 控制符    | 说明                                     |
| --------- | ---------------------------------------- |
| public    | 共有的，对所有类可见。                   |
| protected | 受保护的，对同一包内的类和所有子类可见。 |
| private   | 私有的，在同一类内可见。                 |
| 默认的    | 在同一包内可见。默认不使用任何修饰符。   |

**应用范围**

Java中，可以使用访问控制符来保护对类、变量、方法和构造方法的访问。Java 支持 4 种不同的访问权限。

- **default** (即默认，什么也不写）: 在同一包内可见，不使用任何修饰符。使用对象：类、接口、变量、方法。
- **private** : 在同一类内可见。使用对象：变量、方法。 **注意：不能修饰类（外部类）**
- **public** : 对所有类可见。使用对象：类、接口、变量、方法
- **protected** : 对同一包内的类和所有子类可见。使用对象：变量、方法。 **注意：不能修饰类（外部类）**。

**请注意以下方法继承的规则：**

- 父类中声明为public的方法在子类中也必须为public。
- 父类中声明为protected的方法在子类中要么声明为protected，要么声明为public。不能声明为private。
- 父类中默认修饰符声明的方法，能够在子类中声明为private。
- 父类中声明为private的方法，不能够被继承。

# final关键字

修饰的类：不能被继承，但可以继承其它的类

修饰的方法：不能被重写

修饰的变量：是一个常量，值只能设置一次

修饰引用类型的变量：是地址值不能改变，但属性值可以改变

# static关键字

修饰的成员变量：静态方法（类变量），可以被该类下所有的对象所共享，调用方式：”类名.类变量名“

修饰的成员方法：静态方法中没有对象this，所有不能访问非静态成员，调用方式：”类名.类方法“

# 方法重载

在同一个类中的多个方法，它们方法名相同，参数列表不同，方法重载与返回值类型无关：

```java
public class test {
    //数据类型不同
    public static int sum(int a,int b){
        return a + b;
    }
	//数据类型不同
    public static long sum(long a,long b){
        return a + b;
    }
    //数据个数不同
    public static double sum(double a,float b,int c){
        return a + b + c;
    }
}
```



# 继承

java中，子类只能继承父类的非私有成员（成员变量，成员方法）:

**主程序入口**

```java
package com.huang;

public class test {
    //主程序入口
    public static void main(String[] args) {
        //创建一个对象
        Child c = new Child();

        //调用set方法
        c.setName("huang");

        //调用get方法
        System.out.println(c.getName());
    }
}
```

**定义子类**

```java
package com.huang;

//定义子类
public class Child extends Parent {
}
```

**定义父类**

```java
package com.huang;

//定义父类
public class Parent {
    private String name;
    private int age;
    //无参构造
    public Parent() {
    }
    //有参构造
    public Parent(String name, int age) {
        this.name = name;
        this.age = age;
    }
    //get和set方法
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```



# 抽象类

抽象类概念：包含抽象方法的类，用abstract修饰。

抽象方法概念：只有方法声明，没有方法体的方法，用abstract修饰。

抽象方法的由来：当需要定义一个方法，却不明确方法的具体实现，可以将方法定义为abstract，具体实现延迟到子类。

**主程序入口**

```java
package com.huang;

public class test {
    //主程序入口
    public static void main(String[] args) {
        //创建一个对象
        Dog c = new Dog();
        c.eat();
    }
}
```

**子类**

```java
package com.huang;

public class Dog extends Animal {

    public void eat() {
        System.out.println("吃骨头");
    }
}
```

**父类（抽象类）**

```java
package com.huang;

//父类，动物类（抽象类）
public abstract class Animal {
    //抽象方法（要求子类必须重写）
    public abstract void eat();
}
```



# 接口

**概念：**接口技术用于描述类具有什么功能，但并不给出具体实现，类要遵从接口描述的统一规则定义，对外提供的一组规则、标准。

**接口定义：**

- 定义接口格式：interface 接口名 {}
- 类和接口是实现关系，用implements表示：class 类名 implements 接口名

**接口创建对象的特点：**

- 接口不能实例化：通过多头的方法实例化子类对象
- 接口的子类（实现类）：可以是抽象类（无效重写父类方法），也可以是普通类

**接口继承关系的特点：**

- 接口与接口之间的关系：可以是单继承，也可以是多继承，格式：接口 extens 接口1，接口2，接口3...
- 继承和实现的区别：
  - 继承体现的是 “is a” 的关系，父类中定义共性内容
  - 实现体现的是 “like a” 的关系，接口中定义扩展功能

**接口成员变量特点：**

- 接口没有成员变量，只有共有的、静态的常量，格式：**public static final** 常量名 = 常量值

**接口成员方法的特点：**

- JDK7以前，共有的、抽象方法：public abstract 返回值类型 方法名（）；
- JDK8以后，可以有默认方法和静态方法：public default 返回值类型 方法名（）；static 返回值类型  方法名（）；
- JDK9以后，可以有私有方法：private 返回值类型  方法名（）；

主程序入口

```java
package com.huang;

public class MyTest {
    public static void main(String[] args) {
        //创建厨师对象
        FoodMeus cc = new ChinaCoker();
        //创建顾客对象
        Comuser comuser = new Comuser(cc);
        //下单
        comuser.order();
    }
}
```

客户调用菜单接口类

```java
package com.huang;

//调用菜单接口的客户类
public class Comuser {
    //封装数据类型
    private FoodMeus foodMeus;

    //无参构造
    public Comuser() {
    }
    //有参构造
    public Comuser(FoodMeus foodMeus) {
        this.foodMeus = foodMeus;
    }
    //get和set方法
    public FoodMeus getFoodMeus() {
        return foodMeus;
    }

    public void setFoodMeus(FoodMeus foodMeus) {
        this.foodMeus = foodMeus;
    }
    //下单实例方法
    public void order(){
        foodMeus.tangchupaigu();
        foodMeus.xihongshi();
    }
}
```

菜单接口类

```java
package com.huang;

//菜单接口，定义抽象方法
public interface FoodMeus {
    void xihongshi();
    void tangchupaigu();
}
```

厨师实现类（两个实现）

```java
package com.huang;

//菜单接口的实现类，定义接口方法的具体实现功能
public class ChinaCoker implements FoodMeus {

    public void xihongshi() {
        System.out.println("中餐，西红柿");
    }

    public void tangchupaigu() {
        System.out.println("中餐，糖醋排骨");
    }
}

```

```java
package com.huang;

public class UsbCoker implements FoodMeus{
    public void xihongshi() {
        System.out.println("西餐，西红柿");
    }

    public void tangchupaigu() {
        System.out.println("西餐，糖醋排骨");

    }
}
```

# java API工具

文档地址：https://docs.oracle.com/javase/8/docs/api/

## Object类

**简介**

类层次结构最顶层的基类，所有类都直接或间接的继承自Object类，所有类都是一个Object对象。

**构造方法**

Object()：构造一个对象，所有子类对象初始化时都会优先调用该方法。

**成员方法**

- int hashCode()：返回对象的哈希码值，该方法通过对象的地址值进行计算，不同对象的返回值一般不同。
- Class<?> getClass()：返回调用此方法对象的运行时类对象（调用者的字节码文件对象）。
- String toString()：返回该对象的字符串表示。
- boolean equals()：返回其它某个对象是否与此对象”相等“，默认情况下比较两个对象的引用（地址值），建议重写。

## Scanner类

**简介**

扫码器，能够解析字符串和基本数据类型的数据。

**构造方法**

Scanner(InputStream)：构造一个扫描器对象，从指定输入流中获取数据参数System.in，对应键盘录入。

**成员方法**

- hasNetXxx()：判断是否还有下一个输入项，其中Xxx可能是任意基本数据类型，返回结果为布尔类型。
- nextXxx()：获取下一个输入项，其中Xxx可能是任意基本数据类型，返回对应类型的数据。
- String nextLine()：获取下一行数据，以换行符作为分隔符。
- String next()：获取下一个输入项，以空白字符作为分隔符：空格、tab、回车等。

## String类

**简介**

字符串，每个字符串对象都是常量。

**构造方法**

String(byte[])：构造一个String对象，将指定字节数组中的数据转化成字符串。

String(char[])：构造一个String对象，将指定字符数组中的数据转化成字符串。

**成员方法（判断）**

- boolean equals(String)：判断当前字符串与给定字符串是否相同，区分大小写。
- boolean equalsIgnoreCase(String)：判断当前字符串与给定字符串是否相同，不区分大小写。
- boolean startsWith(String)：判断是否以给定字符串开头。
- boolean isEmpty()：判断字符串是否为空。

**成员方法（获取）**

- int Length()：获取当前字符串的长度。
- char charAt(int index)：获取指定索引位置的字符
- int indexOf(String)：获取指定字符（串）第一次出现的索引
- int lastIndexOf(String)：获取指定字符（串）最后一次出现的索引
- String substring(int)：获取指定索引位置（含）之后的字符串
- String substring(int,int)：获取从索引start位置（含）起至索引end位置（不含）的字符串

# 集合和数组

**集合**

- 只能存储引用类型
- 长度不固定，可以任意扩容
- 单列集合
  - Collection是**单列集合**的根接口，主要用于存储一系列符合某种规则的元素，它有两个重要的子接口List和Set。
  - List接口的特点是**元素有序可重复**
  - Set接口的特点是**元素无序，不可重复**
  - ArrayList和LinkedList是List接口的**实现类**
  - HashSet和TreeSet是Set接口的**实现类**

- 双列集合
  - Map是**双列集合**的根接口，特用蓝色标注，用于存储具有键（Key），值（Value）映射关系的元素，每个元素都包含一个键-值对，在使用该集合时可以通过指定的键找的对应的值。
  - **Map为独立接口**
  - HashMap和TreeMap是Map接口的实现类

**数组**

- 存储多种数据类型
- 固定长度，不可改变容量

## 集合使用案例

```java
package com.huang;

import java.util.*;

public class DouDiZu {
    public static void main(String[] args) {
        //存放牌编号和牌
        Map<Integer,String> pokers = new HashMap<>();

        //存放牌编号
        List<Integer> list = new ArrayList<>();

        //定义牌
        String[] munbers = {"3","4","5","6","7","8","9","10","J","Q","K","A","2"};
        String[] colors = {"方块","红桃","梅花","黑桃"};

        //定义牌编号
        int num = 0;

        //遍历，组合成牌
        for (String munber : munbers) {
            for (String color : colors) {
                String poker = color + munber;
                pokers.put(num,poker);
                list.add(num);
                num++;
            }
        }
        //添加大小王
        pokers.put(num,"小王");
        list.add(num++);
        pokers.put(num,"大王");
        list.add(num++);

        //洗牌
        Collections.shuffle(list);

        //创建玩家集合和底牌集合
        List<Integer> liuyifei = new ArrayList<>();
        List<Integer> zhaoliying = new ArrayList<>();
        List<Integer> xiaohei = new ArrayList<>();
        List<Integer> dipai = new ArrayList<>();

        //发牌
        for (int i = 0; i < list.size(); i++) {
            //获取牌编号，i是索引
            Integer pokerNum = list.get(i);

            //取模
            if (i >= list.size() - 3) {
                dipai.add(pokerNum);
            } else if (i % 3 == 0) {
                liuyifei.add(pokerNum);
            } else if (i % 3 == 1) {
                zhaoliying.add(pokerNum);
            } else if (i % 3 == 2) {
                xiaohei.add(pokerNum);
            }
        }

        //查看牌
        System.out.println("liuyifei：" + showPokers(liuyifei,pokers));
        System.out.println("zhaoliying：" + showPokers(zhaoliying,pokers));
        System.out.println("xiaohei：" + showPokers(xiaohei,pokers));
        System.out.println("dipai：" + showPokers(dipai,pokers));

    }

    //看牌方法
    public static String showPokers(List<Integer> list, Map<Integer,String> pokers){

        //排序牌，从小到大
        Collections.sort(list);

        //合并牌组
        StringBuilder showPoker = new StringBuilder();

        //使用牌编号获取牌内容
        for (Integer l : list) {
            String s = pokers.get(l);
            showPoker.append(s + " ");
        }

        //转成字符串
        String str = showPoker.toString();

        return str.trim();

    }

    }

```

# 异常处理

## 捕获

```java
package com.huang;

/**
 * 捕获异常
 * 先执行try{}中的内容，看是否有异常
 * 没有：直接执行finally语句中的内容
 * 有：先执行catch（）{}语句，在执行finally{}语句
 */
public class CatchTry {
    public static void main(String[] args) {

        try {
            //尝试要执行的代码
            int num = 10;
            System.out.println(num);
            //return;
        } catch (Exception e) {
            //出现问题后的代码
            System.out.println("不能被0除");
        }finally {
            //都要执行的代码
            System.out.println("都要执行");
        }
    }
}
```



## 抛出

```java
package com.huang;

/**
 * 抛出异常（throws Exception），把异常抛给调用者
 * 调用者两种处理方式
 * 1。直接再抛出异常
 * 2.使用try{}、catch（）{}捕获异常
 */
public class Throws {
    public static void main(String[] args) throws Exception {

        //show();

        try{
            show();
        }catch (Exception e){
            System.out.println("代码出错");
        }

        System.out.println("是否执行");
    }

    public static void show () throws Exception {
        int a = 10 / 0;
        System.out.println(a);
    }
}

```

# IO流

概述：I（input）/O（output），是java数据传输方式。

划分：

- 按流向分
  - 输入流：读数据
  - 输出流：写数据
- 按操作分
  - 字节流：以字节为单位来操作数据
    - InputStream：字节输入流的顶层抽象类
      - FileInputStream：普通的字节输入流
      - BufferedInputStream：高效的字节输入流
    - OutputStream：字节输出流的顶层抽象类
      - FileOutputStream：普通的字节输出流
      - BufferedOutputStream：高效的字节输出流
  - 字符流：以字符为单位来操作数据
    - Reader：字符输入流的顶层抽象类
      - FileReader：普通的字节输入流
      - BufferedReader：高效的字节输入流
    - OutputStream：字符输出流的顶层抽象类
      - FileWriter：普通的字符输出流
      - BufferedWriter：高效的字符输出流

## 文件处理

```java
package com.huang;

import java.io.File;
import java.io.IOException;

/**
 * 文件处理
 */
public class FileCreate {
    public static void main(String[] args) throws IOException {

        //创建文件操作对象，方式一
        File file1 = new File("D:/hexo/1.txt");
        System.out.println(file1);

        //创建文件操作对象，方式二
        File file2 = new File("D:/hexo/","1.txt");
        System.out.println(file2);

        //创建文件操作对象，方式三
        File file3 = new File("D:/hexo/");
        File file4 = new File(file3,"1.txt");
        System.out.println(file4);

        //创建文件，如果存在，则不创建，返回false
        File file5 = new File("d:/hexo/2.txt");
        boolean flag1 = file5.createNewFile();
        System.out.println(flag1);

        //创建单级目录
        File file6 = new File("d:/hexo/huang");
        boolean flag2 = file6.mkdir();
        System.out.println(flag2);

        //创建多级目录
        File file7 = new File("d:/hexo/huang/a/b/c");
        boolean flag3 = file7.mkdirs();
        System.out.println(flag3);

        //判断文件是否存在
        File file8 = new File("d:/hexo/huang/a/b/c");
        System.out.println("是否是文件夹：" + file8.isDirectory());
        System.out.println("是否是文件：" + file8.isFile());
        System.out.println("是否是存在：" + file8.exists());

        //获取绝对路径
        File file9 = new File("common/fastjson-1.2.53.jar");
        String absolutePath = file9.getAbsolutePath();
        System.out.println(absolutePath);

        //获取相对路径，必须在项目内
        String  absolutePath1= file9.getPath();
        System.out.println(absolutePath1);

        //获取文件名
        String  absolutePath2= file9.getName();
        System.out.println(absolutePath2);

        //获取目录下的所有文件名,返回字符数组
        File file = new File("d:/hexo");
        String[] list = file.list();
        for (String s : list) {
            System.out.println(s);
        }

        //获取目录下的路径+文件名，返回File[]数组，可以使用file方法
        System.out.println("----------------------");
        File[] files = file.listFiles();
        for (File file10 : files) {
            System.out.println(file10);
            System.out.println(file10.isDirectory());
        }

    }
}
```

## 文件读取



## 文件写入