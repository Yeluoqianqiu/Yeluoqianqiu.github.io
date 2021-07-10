*假设你是一名有着相当经验的Java程序员（掌握Java的基本语法或者说J2SE），然后在大数据和人工智能大潮来临之际，你需要学习Python。这个时候可能你拿起了很多书，甚至购买了在线课程，可是总是感觉自己似懂非懂？*

*这是因为是Java和Python的设计思想有很大的不同！如果带着Java的思想去理解Python，这并不是一个好事，所以这篇文章试图对照两种语言的差异，以一个Java程序员的视角，希望把这个劣势转变为优势！*

1. **Java编译型静态语言 vs Python解释型动态语言**

Java运行前需要进行编译，成为.class文件，然后加载到虚拟机上运行。"编译“这个动作是程序员手工（使用javac编译器）或者IDE自动完成。Python在运行前也会编译成字节码（为了更高效的运行），然后在解释器（为了跨平台）上“逐行“运行的。而“编译”这个动作程序员是没有感知的，由Python解释器自动完成。

可能这个很多人说我都知道，可是这就是最重要的区别！Python解释器为了更有效的运行，做了很多优化，知道这些才能写出pythonic（地道而且优雅的）代码。

---

2. **Java纯面向对象编程 vs Python面向对象及面向过程编程**

Java无论处理多小的事情，你开始编写的代码总是要写在一个类里，然后实例化对象。而Python对于处理简单的事情，简单几句就能跑起来。如果在写大型复杂应用的时候，Python又提供了面向对象的编程。

---

3. **Java函数参数功能单一 vs Python多种函数参数功能多种**

Java仅提供了可变参数一种功能，具体实现看以下例子：

```
public class TestVarArgus {  
        public static void dealArray(int... intArray){  
            for (int i : intArray){  
                System.out.println(i +" ");
            }  
        }  
      
        public static void main(String args[]){   
            dealArray(1);  
            dealArray(1, 2, 3);  
        }  
    }
```

输出如下：

```
1
1 2 3
```

如果Java要实现参数默认值，只能通过“重载方法”的手段来间接实现，例子如下：

```
public void TestParameter(int level) {
        float money = 0.0f;
        boolean ratable = true;
        TestParameter(level, money, ratable);
    }
    public void TestParameter(int level, float money) {
        boolean ratable = true;
        TestParameter(level, money, ratable);
    }
    public void TestParameter(int level, float money , boolean ratable ) { 
                //最终实现在这里 
    }
```

而Python是在语言层面就支持了不同的参数类型，下面一一来说：

**默认参数** ，使用等于号（=）作为标记，如不提供默认值，则函数按照默认值计算：

```
def power(x, n=2):
    s = 1
    while n > 0:
        n = n - 1
        s = s * x
    return s
>>> power(5)
25
>>> power(5, 2)
25
>>> power(5, 3)
125
```

**可变参数** ，使用星号（*）作为标记，其实是在调用时传递了一个tuple（元组）:

```
def calc(*numbers):
    sum = 0
    for n in numbers:
        sum = sum + n * n
    return sum
>>> calc(1, 2)
5
>>> calc()
0
>>> nums = [1, 2, 3]
>>> calc(*nums)
14
```

**关键字参数** ，使用两个星号（**）作为标记，其实是在调用时传递一个dict（字典）：

```
def person(name, age, **kw):
    print('name:', name, 'age:', age, 'other:', kw)
>>> person('Michael', 30)
name: Michael age: 30 other: {}
>>> person('Bob', 35, city='Beijing')
name: Bob age: 35 other: {'city': 'Beijing'}
>>> person('Adam', 45, gender='M', job='Engineer')
name: Adam age: 45 other: {'gender': 'M', 'job': 'Engineer'}
>>> extra = {'city': 'Beijing', 'job': 'Engineer'}
>>> person('Jack', 24, **extra)
name: Jack age: 24 other: {'city': 'Beijing', 'job': 'Engineer'}
```

还有另外一种是命名关键字参数没有细说，这么多参数的时候，要注意使用顺序，必须是必选参数、默认参数、可变参数、命名关键字参数和关键字参数。（以上编程例子参考廖雪峰老师的函数的参数章节）

---

4. **Java的包package vs Python的package module**

Java中以目录(package)作为命名空间，单独的package下有自己的类和接口，而不同的package上虽然在文件系统中是有包含关系，但在Java的命名空间上没有从属关系。例如org.dominoo.action和org.dominoo.action.asl之间绝对没有包与子包的关系。

Python在这方面而要复杂多了。估计第一次看到这些语法的Java程序员会懵。

package，这也是对应文件系统的目录，每个目录下都需要有__*init.py* __描述文件。

module，对应的是一个以.py结尾的python文件。

那么看看以下几个语句，如果不仔细研究一番，会感觉混乱。

```
import module1 #导入module1.py文件的所有内容，在以后使用中需要用module1.name
from module1 import name #复制module1.py文件中的name变量
from module1 import * #复制module1.py文件中的所有变量
from dir1.dir2.module1 import name #复制dir1/dir2 package下的module1.py文件中的name变量
```

---

5. **Python中的yield关键字，生成器(generator) 迭代器(Iterator) 可迭代对象(Iterable) 的联系和区别**

先看一幅直观的图了解它们之间的关系：

![image](https://cdn.nlark.com/yuque/0/2020/jpeg/938149/1596088826725-e6183b88-9fbb-4fe2-929e-f4f880e27dd5.jpeg "image")

在英文中able结尾的，一般表示一种能力，而or结尾的一般表示一个名词。那么iterable和iterator之间的区别是：

1）对象拥有__iter__方法，是可迭代对象；对象拥有next方法，是迭代器。

2）定义可迭代对象，必须实现__iter__方法；定义迭代器，必须实现__iter__和next方法。

3）迭代器保存的是迭代算法，表示的是一个数据流，不断的通过next方法返回下一个数据，直到没有数据时抛出`StopIteration`错误。可以把这个数据流看做是一个有序序列，但我们却不能提前知道序列的长度，只能不断通过`next()`函数实现按需计算下一个数据，所以`Iterator`的计算是***惰性*** 的，只有在需要返回下一个数据时它才会计算。

（具体参考廖雪峰老师的[高级特性](https://link.zhihu.com/?target=https%3A//www.liaoxuefeng.com/wiki/1016959663602400/1017269809315232)章节）

---

6. **Java的单继承和接口 vs Python的多继承和Mixin类**

可能你已经知道，Java是通过接口来实现多继承的。例如这样的一种关系：

```
public class Person {
     protected name;
     protected age;
}
//老师继承了人，所以有了姓名和年龄属性
public class Teacher extends Person {
}
```

如果这个时候，老师具备了教学的能力？那么Java只需要再Teacher中实现teachable这个接口就可以：

```
public interface Teachable {
     void teachSomeBody();
}
public class Teacher extends Person implements Teachable{
}
```

如果教师还有其它的能力，那就多继承接口，解决问题。

可是Python中是没有接口的，怎么办呢？于是引用了一种叫Mixin的类（不是“迷信”啊，是mix-in，混入功能的意思）来代替接口。来看例子：

```
class Vehicle(object):
    pass
 
class PlaneMixin(object):
    def fly(self):
        print 'I am flying'
 
class Airplane(Vehicle, PlaneMixin):
    pass
```

飞机是交通工具的子类并且有飞行能力，因此Airplane继承了Vehicle和PlaneMixin类，其中PlaneMixin类说明的是一种能力，这个类是作为功能添加到子类中，而不是作为父类。当然，如果有多种功能，也可以继承多个Mixin的类。

好吧，我也觉得，其实这是一个折中的处理办法，而且并不优雅。（或者我对python的理解还不够）对于Minxi需要注意的地方可参考这篇文章：[一个例子走近 Python 的 Mixin 类](https://link.zhihu.com/?target=https%3A//blog.csdn.net/u012814856/article/details/81355935)

---

7. **Java的类成员访问控制 vs Python的约定**

想必你已经清楚Java的关键字private / protected / public的区别，联系和作用。在Python中，这些都没有啊，怎么办？通过约定和自觉（对，Python里有很多这样的约定，例如缩进，还有函数命名，参数命名，至于为啥是自觉，后面会说到）

在Python类内部，如果属性不被外部访问，可以把属性的名称前加上“双下划线”（__），就是私有变量（private），如以下例子：

```
class Student(object):
    def __init__(self, name, score):
        self.__name = name
        self.__score = score
    def print_score(self):
        print('%s: %s' % (self.__name, self.__score))
```

那么在外部就访问不到name和score属性了。

```
>>> bart = Student('Bart Simpson', 59)
>>> bart.__name
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'Student' object has no attribute '__name'
```

双下划线开头的实例变量是不是一定不能从外部访问呢？其实也不是。不能直接访问`__name`是因为Python解释器对外把`__name`变量改成了`_Student__name`，所以，仍然可以通过`_Student__name`来访问`__name`变量。（不过自觉不要这么做！）

那么还有一种是“单下划线“（_），这样的实例变量外部是可以访问的，当你看到这样的变量时，意思就是，“虽然我可以被访问，但是，请把我视为私有变量，不要随意访问”。

需要注意的是，以双下划线开头，双下划线结尾的，是特殊变量，特殊变量是可以直接访问的，不是私有（private）变量。