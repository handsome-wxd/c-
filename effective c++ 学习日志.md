# 准则

## 1.少使用define

- define所定义的常量会在预处理的时候被替代，出错编译器不容易找到错误。而且还没有作用范围限制，推荐使用const
- define宏定义的函数，容易出错，而且参数需要加上小括号，推荐使用inline
- 有的类中例如数组初始化需要添加元素个数，如果define定义的常量没有作用范围限制，推荐使用enums

## 2.确定对象使用前先初始化

- 为内置型对象进行手动初始化
- 构造函数最好使用成员初始化列，如果在构造函数中进行赋值的话相当于先初始化默认值，然后有赋给值，导致浪费时间。
- 为了免除跨编译单元之初始化次序，将非本地静态变量替换成本地静态变量。因为静态变量在程序编译时就赋值存在，不会导致引用时未构造。

## 3.为多态基类声明析构函数

- 带多态性质的基类应该声明一个virtual的析构函数。如果class带有任何virtual函数，它就应该拥有一个virtual析构函数
- Classs的设计目的如果不是作为base  classes使用，或不是为了具备多态性，就不该声明virtual析构函数，因为设置virtual会使派生类带上virtual 函数表，导致浪费空间。

```c++
class A{
  virtual ~A(){
      
  }
}
class B:pulic A{

}
A *b=new (B);
//当未定义virtual 基类析构函数时，会调用A的析构函数，可能导致未释放B新增内成员的的空间
delete b;
```

## 4.不要让异常逃离析构函数

```c++
class DBconn{
public:
void close(){
db.close();
closed=true;
}
~DBconn(){
if(!closed)
try{
db.close()
}catch(...){
std:abort();//结束程序，不要让异常传出去，造成不确定后果
}
}
}
private:
DBConnection db;
bool closed;
};
//在对元素析构时，当两个及以上元素出现异常时，程序就会停止或者造成不明确的行为，造成内存呢泄露
```

- 析构函数绝对不要吐出异常。如果一个析构函数调用的函数可能出现异常，析构函数应该捕捉任何异常，然后吞下他们或者结束程序。

- 如果客户需要对某个操作在运行期间抛出的异常做出反应，class 应该提供一个普通函数执行该操作。

  ## 5.不要在构造函数或者析构函数里面调用virtual函数

  对于virtual 一般是多态定义的，但是当构造函数构造子类使会先构造父类，当在构造器中调用virtual会导致调用的是父类版本的virtual，因为在构造父类时，此时编译器还不知道子类有什么成员，所以用当前版本的。

  ## 6.在operator中处理自我赋值

  ```c++
  //当rhs与this是同一个对象时，delete pb会导致低下rhs的pb也delete 导致报错
  Widget::operator=(const Widget& rhs){
  	delete pb;
  	pb=new Bitmap(*rhs.pb);
  	return *this;
  }
  //进行验同测试,当时new出现异常时，会导致this.pb被释放
  Widget::operator=(const Widget& rhs){
      if(&rhs==this)
          	return *this;
  	delete pb;
  	pb=new Bitmap(*rhs.pb);
  	return *this;
  }
  //这样既不怕是统一对象，也不怕new出错
  Widget::operator=(const Widget& rhs){
      Bitmap *pOrig=pb;
  	pb=new Bitmap(*rhs.pb);
      delete pOrig;
  	return *this;
  }
  
  
  ```

  ## 7.以独立的语句将newed对象置于智能指针

```c++
processWidget(std:trl::shared_ptrM<Widget> pr(new Widget),priority())
```

对于c++执行这句话，以什么样的次序进行执行弹性很大，与java与c#不同1。如果以 new widget,priority(),shared_ptrM<Widget>(),顺序则可能出现内存泄露的风险。当priority出错()时，将无法将new出来的内存进行删除，因此最好以单独语句执行。

```
std:trl::shared_ptrM<Widget> pr(new Widget);
processWidget(std:trl::shared_ptrM<Widget> pr,priority())
```

## 8.以引用传递代替值传递

- 按值传递可能会使特化的信息别切割

```c++
class window{
public:
	virtual :display();
}

class myWindow:public window{
    private:
    int size;
    public:
    virtual:display();
}
void useDisPlay1(window w){
    w.display();
}
void useDisplay2(window& w){
    w.display();
}
mywidow w;
useDisPlay1(w);//按值传递会使 mywindow所得有特化的信息被切割
userIdsPlay2(w);//按引用传递则不会使切割
```

- 按值传递会让编译器去构造副本，对于一般自定义的类来说浪费时间和空间
- 按值传递适合内置类型，STL迭代器和函数对象。因为传递引用的实质使传递指针。

## 9.inline的使用

- 将大多数inline限制在小型、被频繁调用的函数身上，这可使以后的调试过程和二进制升级更容易，也可使潜在的代码膨胀问题最小化，使程序提升速度最大化。
- 不要只因为function template 出现在头文件就将他们设置为inline。

inline一定被放在头文件是因为编译器为了将函数调用代码替换为函数本体 要知道函数本体长什么样子

template 一定放在头文件里是 因为一旦被使用，编译器会将它具体化，得知道它长什么样子。

### 10.将文件间的编译依存关系降到最低

- 如果使用object reference 或者obejct point 可以实现就不要用 object了。
- 如果能够，尽量以class声明替代函数。
- 为声明式和定义式提供不同的头文件
- 或者将声明类定义为abstrate 类，实现类继承进行继承。

就是将类的实现和申明写成两个类，然后在声明类中引用实现类的指针。这样当实现类中的成员进行变化时，声明类不用重新编译。而且声明类中也无法看出方法的具体实现。

## 11.避免遮蔽继承而来的名称

```c++
class base{
private: 
int x;
public :
void fun()
}

class drived :public base{
   public:
    using base:fun()
    void fun();
}
//子类fun()会遮蔽父类fun()，想用父类fun就要用 using base:fun()

```

derived的作用域

![image-20220830185337010](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220830185337010.png)

drived classes内的名称会遮蔽base class内的名称，为了让遮蔽的名称重用，用using 或者转交函数

## 12.private继承

- private继承意味is-implemented-in-terms of（根据某物实现）。它通常比复合成员的级别低。当derived class 需要访问protected base class的成员，或需要重新定义继承而来的virtual函数时，这么设计是合理的。
- 和复合不同，private继承可以造成empty base 最优化。这对致力于”对象尺寸最小化“的程序开发者而言，可能很重要。

## 13.多重继承

- 多重继承比单一继承复杂。它可能导致新的歧义性，以及对virtual继承的需要。

- virtual继承会导致速度大小，初始化等等成本。如果virtual base classes 不带任何数据，将是最具有实用价值的情况。

- 多重继承的确有正当途径，当其中一个情节涉及”public继承某个Interface class“ 和private 继承某个协助实现的class的两相组合。例如public 继承的接口在private 继承的类中有方法去实现。

  ![image-20220830215346169](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220830215346169.png)

## 14.将与参数无关的代码抽离template

- templates生成多个classes和多个函数，所以任何template代码都不该与某个造成膨胀的template参数产生依赖关系。

- 因非类型模板参数而造成的代码膨胀，往往可以消除，做法是以函数参数或者class成员替换template函数。类如去定义类中一些参数，这样的参数可以写在类中。
- 因类型参数造成的代码膨胀，往往可以降低。做法是让带有完全相同的二进制表述的具体表述共享实现代码。类如int与long可能公用一个模板。