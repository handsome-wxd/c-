[toc]

# 使用STL库的55条建议

## 1.慎重选择容器的类型

根据你的业务需求和算法去选择不同的容器，将数据结构和算法结合起来。

c++容器类型：

- 序列容器：其中对象有序排列，根据整数值进行索引,有vector,list,deque,string
- 关联容器：其中对象的顺序不重要，根据键值进行索引,set,map,multiset,multimap
- 适配器：调正原有容器的行为，使对外产生新的接口，类型和返回元素。
- 生成器：构造元素序列

标准容器:指的是c++标准化后的容器，遵从c++的标准，不同的编译器都支持，代码移植能力强。vector,list,deque,string，set,map,multiset,multimap

非标准容器:指的是违背标准化后的容器，在不同版本的编译器上可能不支持,hash_set,hast_map等等

使用参考：

例如需要按顺序插入元素就需要序列容器，如果元素的顺序不重要就使用关联容器，如果就频繁删除容器内元素就避免使用内存连续的容器

## 2.不要试图编写独立于容器的代码

因为不同的容器所支持的函数和迭代器类型使不同的，可能会报错，所以不要编写独立于容器的代码。

## 3.确定容器中的对象拷贝正确且高效

当对vector,list,deque进行元素的插入或者删除操作时，现有的元素的位置通常会被移动（拷贝）。使用排序，反转也会被移动（拷贝）。拷贝是STL的工作方式。

当给容器中存入对象时存入的是对象的拷贝，当从容器中取出对象时，取出的容器中对象的拷贝。所以要确保拷贝的正确和高效。

当你创建的时基类对象却存着派生类对象时，拷贝就会剥离将派生的内容剥离。如果想要从基类容器使用派生类的虚方法，就要指针。

而且拷贝指针。

由于是拷贝的，因此在最后析构的时候容易出问题，因此最好采用智能指针。

## 4.调用empty判断是否为空，而不是size

首先empty一般是内联函数，而且empty的时间复杂度是常数。

size不是内联函数，而且size的时间复杂度是线性的。

## 5.区间成员函数优于与之对应单元素成员函数

给定v1和v2两个矢量，使v1的内容等于v2后半段内容相同。怎么实现。

```c++
//1.使用区间函数
v1.assign(v2.begin()+v2.size()/2,v2.end());
//2.使用copy函数，内部还是循环
v1.clear();
copy(v2.begin()+v2.size()/2,v2.end(),back_inserter(v1));
//3.使用insert,这样v1是反着的，因此每次都插到v1前面
insert(v1.begin(),v2.begin()+v2.size()/2,v2.end());
//使用单元素方法
//1.使用赋值
v1.clear()
for(vector<int>::const_iterator ci=v2.begin()+v2.size()/2;ci!=v2.end()；++ci){
    v2.push_back(ci);
}
```

从代码上

- 使用区间函数，可以少写一些代码
- 使用区间函数，意图更明确。

```c++
//将数组data内的元素添加到向量v中
//使用单元素成员函数
vector<int>::iterator insertloc(v.begin());
for(int i=0;i<numValues;i++){
	insertloc=v.insert(insertloc,data[i]);
	++insertloc;
}
```

从效率上，调用单元素成员函数

- 不必要的函数调用，将v.insert()函数调用numValues遍
- 将v中已有元素频繁的移动到插入元素的后面，每个元素移动numValues遍，如果插入元素为对象的化进行拷贝复制对象内的参数，需要的赋值次数更多。
- 如果容器为顺序存储的容器则可能会造成内存的多次扩充，则会把旧内存的元素拷贝进入新内存，并将旧内存的东西销毁然后插入新内存的东西。

对于vector 和string这三条都满足，对于deque第三条不满足。

对于list只有第一条满足，但是单个插入list会导致list需要不断变化pre 和next的指向，没有必要。

## 6.如果容器中包含了通过new操作创建的指针，切记在容器对象析构前，要将指针delete。

```c++
void dosomething(){
vector<widget *>vmp;
for(int i=0;i<N;++i)
	vwp.push_back(new wideget);
}
//当vmp作用域结束后，它的全部元素会被析构，但是new出来的内存没有删除，导致内存泄漏

//第一版本
void dosomething(){
vector<widget *>vmp;
for(int i=0;i<N;++i)
	vwp.push_back(new wideget);
}
...
for(vector<widget*>::iterator i=vmp.begin();i!=vmp.end();++i)
    delete i;
//当时当new 过程中出现异常，还是会导致内存泄露

//第二版本
struct DeleteObject{
    template<typename T>
    void operator()(const T *ptr){
        delete ptr;
    }
};
void dosomething(){
 typedef boost::shared_ptr<widget>spw;
    
vector<spw>vmp;
for(int i=0;i<N;++i)
	vwp.push_back(spw(new wideget);
}
...
for_each(vmp.begin(),vmp.end(),DeleteObject());
 //将指针给智能指针，不怕发生异常内存泄漏，然后采用for_each，区间比单元素需要要高
```

## 7.切勿创建包含auto_ptr的容器对象

首先auto_ptr的对象是被c++标准所禁止的，说明可以移植性不强。

上面提到，STL中采用的是拷贝的方法，auto_ptr对象进行拷贝时会将原来的对象所有权设置交给新对象，然后将原来对象所有权设为为NULL

例如在排序时，会将基准元素拷贝给一个临时对象，使容器中一个元素为null,临时对象会在作用域结束后被销毁。导致容器可能会失去一个或者多个对象。

## 8.慎重选择删除元素的方法

```c++
container<int>c;
```

- 对于容器c,如果想删除值为1963的元素。对于不同容器应该采用不同的方法。

对于连续容器例如 vector,deque,string

```c++
c.earse(remove(c.begin(),c.end(),1963),c.end());
```

对于list

```c++
c.remove(1963);
```

对于关联容器采用

```c++
c.erase(1963);
```

- 当要删除判别式为 true的值

```
bool badValue(int);
```

对于连续容器例如 vector,deque,string

```
c.earse(remove_if(c.begin(),c.end(),badValue),c.end());
```

对于list

```
c.remove_if(badValue);
```

对于关联容器，两种方法

```c++
//方法一
AddocContainer<int>c;
...
AddocContainer<int>goodValues;
remove_copy_if(c.begin(),c.end(),inserter(goodValues,goodValues.end(),badValue);
//将不用删除的元素放入goodValues容器中
//将两容器元素交换
c.swap(goodVlues;)
//方法二
for(AddocContainer<int>::iterator i=c.begin();i!=c.end();){
	if(badValues(*i))
        c.earse(i++);  //确保i被删除后，有一个迭代器指向下一个元素。关联容器不反悔元素
    else
        ++i;
}
//对于 vector,deque,string，list,想要在为true时，写日志
  for(AddocContainer<int>::iterator i=c.begin();i!=c.end();){
	if(badValues(*i)){
        logFile<<"earse "<<*i<<"/n";
       i =c.earse(i);  
    }//确保i被删除后，有一个迭代器指向下一个元素。序列元素返回之下下一个元素的迭代器
    else
        ++i;
}             
               
       
               
```

## 9.了解分配子allocator的约定和限制

- 你的分配子是一个模板，模板参数T代表你为它分配内存对象的类型
- 提供类型定义pointer和reference,但始终让pointer为T* ,让reference 为T&
- 千万不要让你的分配子拥有随对象而不同的状态。通常分配子不应该有非静态的数据结构
- **传给分配子的allocate成员函数的是那些要求内存的对象的个数，new传递的是void* 需要的字节个数。同时记住，allocate返回的T*指针，既然没有T对象被构造。**
- 一定也要提供嵌套的rebind模板，因为标准容器依赖该模板。

## 10.切勿对STL容器的线程不安全性有不切实际的依赖

STL不提供线程安全，需要自己写互斥锁

```c++
//定义lock类
template<tyname Container>
class Lock{
public:
Lock(const Container& container):c(container){
	getMutexFor(c);//创造互斥体
}
~Lock(){
	releaseMutexFor(c);//析构互斥体
}
private:
const Container& c;
}

//
vector<int>v;
...
{
	Lock<vector<int>>lock(v);
	vector<int>::iteator first5(find(v.begin(),v.end(),5));
	if(first5 !=v.begin())
		*first5=0;
}//代码块结束，互斥体被释放

```

## 11.vector和string优于动态分配的数组

vector和string可以实现自己释放内存，并且可以使用STL算法

但是在多线程情况下使用string，可能会造成性能下跌，不如使用vector<char>，因为string采用的是引用计数。

## 12.使用reserve避免不必要的重新分配

![image-20220911205652519](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220911205652519.png)

因为当容器容量不满足需求时，会进行容量扩充，重新申请一个是当前容量数倍的内存，把当前内存中元素移入然后将原来的内存进行析构。这个过程很浪费时间，因此两种方法去避免。

- 当知道需要预留的空间大小使，用reserve函数提前去开辟。
- 当不知道时，开辟足够大的，到最后在去除多余容量。

## 13.注意string实现的多样性

STL对string的实现有不同的方式，每个方式都有不同的特点，详情看第十五条

它们的区别主要是：

string的值可能会被引用，也可能不会被引用

string对象的大小范围是char*的1倍到7倍

创建一个新字符串可能需要0次，1，2次动态分配内存

string对象可能被共享，也可能不共享其大小和内存信息

string可能支持，也可能不支持对单个对象的分配子

不同实现对内存的最小分配有不同策略。

## 14.了解如何将string和vector 传给旧的API

对于vector

```c++
//对于这个旧的API
void dosomething(const int* pInts,size_t numInts);
//vector 应该传递 
vector<int>v;
dosomething(&v[0], numInts);
//也有人说传递 dosomething(v.begin(), numInts);
//begin()返回是一个迭代器，事实上对于vector迭代器就是指针，但是事实上迭代器不是一个指针
```

对于string

```c++
//对于旧的API
void dosomething(const char *pstring)
//strng 应该传递
string str;
dosomething(str.c_str())
//因为string不是连续存储的，而且string不一定是以空字符结尾的。所以string内部有个c_str，指向字符串值的指针
```

## 15.使用‘swap技巧’去除多余的内存

```c++
vector<int>n(100,0);
//删除一部分元素，只留下十个元素，但是容器的容量还是100
n.erase(b.begin()+10,n.end());
//为了去除多余的容量，使用swap
vector<int>(n).swap(n);
//通过vector<int>(n)创建一个隐式对象，这个对象的容量大小就是n实际包含元素的大小，然后与n进行交换，再将隐式对象析构掉。消除对于容量。
//swap调用拷贝构造函数一次，赋值函数两次，将两个对象指针进行调换。不仅容器的内容就交换了，指针，引用都被交换了（string除外），但是相对来说还是在原来的容器中。

//使用swap使内存最小
vector<int>().swap(n);
string str="dada";
string ().swap(str);
```

## 16.避免使用vector<bool>

因为vector<bool>不满足c++标准：c如果是包含对象T的容器，则 T* p=&c[0]是可以通过编译的。但是对于vector<bool>是不行的。因为，vector<bool>在内部是一位一位存的，而不是按自己存储，因为没办法创建指向单个位的指针

可以用deque<bool>替代和bitset替代（大小固定，不能插入元素）。

## 17，理解相等和等价的区别

等价不一定是相等的，相等肯定是等价的。

对于类的等价，取决于operator = =函数中的函数体。在关联容器中，例如map,set这种顺序存储的容器，需要定义一个比较类去确定它们的顺序。

```c++
struct CIstringCompare{
public:
binary_function<string,string,bool>
bool operator()(const string&lhs,const string&rhs )const{
return ciStringCompare(lhs,rhs);//忽略string 大小写的比较
}
};
set<string,CIstringCompare)ciss;
```

## 18.为包含指针的关联函数指定比较类型

```c++
set<string *>ssp;
ssp.insert(new string("Anteater"));
ssp.insert(new string("Wombat"));
ssp.insert(new string("Lemur"));
ssp.insert(new string("Penguin"));
for(set<string *>::iterator it;it!=ssp.end();it++){
cout<<*it<<endl; //打印的是地址 ，而且不一定是按照字符串顺序打印的，因为set是根据地址大小进行排序的
}

//因此需要写一个比较类去进行比较
struct DereferenceLess{
    template<typename,PtrType>
    bool operator()(PtrType lhs,PtrType rhs){
        return *lhs<*rhs;
    }
}
set<string*,DereferenceLess>ssp;//这样就按照字符串顺序进行打印
```

## 19.总让比较函数在相等的情况下返回false

```c++
//例如给set<int,less_equal<int>> s;
s.insert(10);
s.insert(10);
//这会导致容器中出现两个10，违背了set的定义，因为在比较两个10是否等价时采用
!(10<=10)&&!(10<=10) //false 不等价 所以导致存了两个10

```

## 20.切勿直接修改set 或者 multiset的键

所谓的键就是进行比较等价的内容，因为set是按顺序存储的所以不能直接修改。而map是修改不了，因为map存储类型是pair<const k,v>。

```c++
//但是有的sTL不支持修改set 它返回迭代器类型为 const T&,想要修改就要类型转化,但是存在风险
//const_cast 指向原来的引用
const_cast<employee>(*it).setTitle("dasd");
//使用static_cast则不行，它会生成一个临时隐匿对象，修改的是这个对象
static_cast<employee>(*it).setTitle("dasd");
//相当于
(employee(*it)).setTitle("dasd");
```

安全的方法就是拷贝一份元素将其修改，然后删除原有的，再将其添加。

## 21.考虑用排序vector代替关联容器

当你的数据类型满足三个阶段时，可以考虑用排序vector代替关联容器

- 插入删除阶段，几乎不查询
- 查找阶段，几乎不查找
- 重组阶段，将元素全部删除再添加。

因为关联容器底层为平衡二叉树，一个节点保活root,left,right三个地址，相比vector太空间。

## 22.当效率至关重要，谨慎选择opertor[] 和insert

```c++
map<int,widget>m
//在插入时使用operator[]，相当于先要构造临时对象然后析构然后赋值
m[1]=1.50;
//相当于
typedef map<int,widget>IntWidgetMap;
pair<IntWidgetMap::iterator,bool>result=m.insert(IntWidgetMap::value_type(1,widget()));
result.first->second=1.5;
//不如使用insert
m.insert(IntWidgetMap::value_type(1,50);

//当更新数据时
m[1]=51;
//用insert需要构造，然后析构然后赋值
m.insert(IntWidgetMap::value_type(k,v).first->second=v;
```

当插入时用insert效率高，当更新时用operator[]效率高。

## 23.使用distance 和advance将容器的const_iterator转换为iterator

当有的容器类的成员函数仅接受iterator作为参数,const_iterator不能作为他们都参数。必须进行强制类型转换。

```c++
typedef deque<int> IntDeque;
typedef IntDeque:: iterator Iter;
typedef IntDeque:: const_iterator ConstIter;
ConstIter ci;
Iter i(ci);
Iter i(const_cast<Iter>(ci));//const_iterator不能强制转为iterator
//iterator 与const_iterator是两个不同的类，它们之间的差距可能比string 与 double都远
//但是对于vector 与string是可以通过的，因为它们的迭代器底层是char*

//使用advance 与distance实现安全转换 将ci 从const_iterator 转为iterator
Iter i(d.begin());
advance(i,distance<ConstIter>(i,ci));//要加ConstIter 进行强制类型转换
```

## 24.正确理解由reverse_iterator的base()成员函数所产生的iterator的用法

```c++
vector<int>v;
v.reserve(5);
for(int i=1;i<=5;++i){
    v.push_back(i);
}
vector<int>::reverse_iterator ri=find(v.rbegin(),v.rend(),3);
vector<int>::iterator i(ri.base());
```

执行完上述代码之后，该vector和相应迭代器的状态如下图所示：

![image-20220915221408689](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220915221408689.png)

如果在reverse_iterator ri指向的位置插入新元素，只需要在ri.base()位置处插入元素就行了。

但是reverse_iterator处删除元素的话，则需要执行v.((++ri).base())；

## 25.对于逐字符的输入请考虑使用istreambuf_iterator

```c++
//将一个文本文件拷贝到string中
ifstream inputFile("interestingData.txt");
//这段代码没有将空白字符加载进去，因为istream_iterator使用operator>>函数完成实际的读操作，默认operator>>函数会跳过空白字符
//如果没有消除skipws标志，将默认忽略空白字符
inputFile.unsetf(ios::skipws);//消除sKipws标志位
string fileData((istream_iterator<char>(inputFile)),istream_iterator<char>());

//但是拷贝速度不快，因为每一个operator>>操作 需要执行许多额外的操作
//推荐使用istreambuf_iterator，速度快40%，它是直接从缓冲区中读取下一个字符
ifstream inputFile("intersetingData.txt");
string fileData((istreambuf_iterator<char>(inputFile)),istreambuf_iterator<char>());
```

```c++
#include<iostream>
#include<random>
#include<ctime>
#include<fstream>
#include<iterator>
using namespace std;

int main(){
    ofstream outfile;
    outfile.open("file.txt",ios::app);
    srand(time(nullptr));
    for(int i=0;i<1000000;++i){
        outfile<<rand()%26+'a';
    }
    clock_t start,stop;

    ifstream inputFile("file.txt");
    inputFile.unsetf(ios::skipws);
    start=clock();
    string fileData1((istream_iterator<char>(inputFile)),istream_iterator<char>());
    stop=clock();
    cout<<"istream花费了"<<(stop-start)<<endl;
     start=clock();
    string fileData2((istreambuf_iterator<char>(inputFile)),istreambuf_iterator<char>());
    stop=clock();
    cout<<"istreambuf花费了"<<(stop-start)<<endl;

    return 0;
}
```

![image-20220916223026949](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220916223026949.png)

## 26.确保目标区间足够大

```c++
//给一个容器后面插入元素
int transmogrify(int x);//该函数根据x生成一个新值
vector<int> values;
...//给values 里面插入新值
vector<int>results;
//给results.end()插入元素是错误的，因为*resultes.end()里面没有元素
transform(values.begin(),values.end(),results.end(),transmogrify);
//应该用back_inserter生成迭代器进行插入，返回的迭代器将被push_back调用
transform(values.begin(),values.end(),back_inserter(results),transmogrify);
//插在前面,被front_back调用 ，没有front_back的容器不能使用，比如vector
transform(values.begin(),values.end(),front_inserter(results),transmogrify);
//如果担心，需要重新分配空间，就需要使用 reserver
vector<int>results;
results.reverse(results.size()+values.size());
transform(values.begin(),values.end(),front_inserter(results),transmogrify);
```

## 27.了解各种和排序有关的选择

- 如果需要对vector,string,deque或者数组中的元素执行一次完全排序，那么可以使用sort或者stable_sort进行
- 如果有一个vector,string,deque或者数组，并且只需要对等价性最前面的n个元素进行排序，那么可以使用partial_sort

```c++
//将质量最好的前n个放在widgets的最前面
partial_sort(widgets.begin(),widgets.begin()+n,widgets.end(),quealityCompare);
```

- 如果有一个vector,string,deque或者数组，并且只需要只需要找个第n个位置的元素，或者需要找到等价性最前面的n个元素但有不对这n个元素进行排序，那么可以使用nth_element

```c++
//找出质量最好的前n个元素，不用对n个元素进行排序
nth_element(widgets.begin(),widgets.begin()+n-1,widgets.end(),quealityCompare);
//找到质量中间的元素
vector<widget>::iterator it=nth_element(widgets.begin(),widgets.begin()+widgets.size()/2,widgets.end(),quealityCompare);
```

- 如果需要将一个标准序列容器的元素按照是否满足某个特定的条件区分开，那么partition和stable_partition很合适

```c++
//找到第一个质量比二级还差元素的位置
vector<widget>::iterator goodEnd=partition(widgets.begin(),widgets.end(),hasAcceptableQulity);
//返回第一个不满足大于二级质量条件的widget
```

- 如果你的数据在一个list中，仍然可以使用partition和stable_partition

- 排序算法的效率排序

partition,stable_partition,nth_element,partial_sort,sort,stable_sort.

## 28.如果确实需要删除元素，则需要在remove这一类算法之后调用erase

```c++
//remove的参数是迭代器，并不知道容器的类型，无法调用容器的成员函数 earse,所以用remove 容器中元素的数目并不会减少
template<<class ForwardIterator,class T>
ForwardIterator remove(ForwardIterator first,ForwardIterator last,const T&value);
//remove只是把不用删除的元素移到区间的前部，它返回迭代器指向最后一个“不用删除”的元素之后的元素。
remove(v.begin(),v.end(),99);
```

事实上并没有将99的放在后面，后面还保留着旧值，用保留的值覆盖掉要删除的值

![image-20220917190200302](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220917190200302.png)

![image-20220917190127261](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220917190127261.png)

```c++
//为了真的删除空间中值，使用erase
v.erase(remove(v.begin(),v.end(),99),v.end());
//list的remove是和erase融合的
list<int>li;
li.remove(99);
```

remove对于包含指针的容器，再用在和erase进行连用时，记得先要释放内存，防止内存泄漏。对于智能指针不用这样做。

## 29.使用accumulate进行区间统计

```c++
//统计list中所有值的和
list<double> ld;
...//添加元素
double sum=accumulate(ld.begin(),id.end(),0.0);
//统计容器中所有字符串长度之和
string::size_type
StringLengthSum(string::size_type sumSoFar,const string& s){
 return sumFoFar+s.size();
}
set<string>ss;
string::size_type lengthSum=accumlate(ss.begin(),ss.end(),static_cast<string::size_type(0),stringLengthSum);

//统计区间内所有点的平均值
struct Point{
    Point(double initX,double initY):x(initX),y(initY){}
    double x,y;
}
class PointAverage:public binary_function<Point,Point,Point>{
    public:
    PointAverage():xSum(0),ySum(0),numPoints(0){}
    const Point operator()(const Point& avgSoFar,const Point &p){
        ++numPoints;
        xSum+=p.x;
        ySUm+=p.y;
        return Point(xSum/numPoints,ySum/numPoints);
    }
    private:
    size_t numPoints;
    double xSum;
    double ySum;
}
Point avg=acculate(lp.begin(),lp.end(),Point(0,0),PointAverage());
```

## 30.遵循按值传递的原则来设计函数子类

对于多态的函数对象，不能使用虚函数，因为参数类型是基类，而实参类型是派生类的，在传递的过程中会产生剥离问题：在对象拷贝的过程中，派生部分可能会被去掉，而仅保留基类部分。

函数对象在STL中作为参数或者返回时总是值传递，意味着第一：函数对象小巧，第二，函数对象是单态。

## 31.确保判别式是纯函数

- 判别式：是一个返回值为bool的函数。在STL中，判别式有着广泛的用途。标准关联容器的比较函数就是判别式，对于find_if以及各种排序算法，判别式往往也被作为参数来传递。
- 纯函数：是指返回值仅仅依赖其参数的函数。
- 判别类：是一个函数的子类，它的operator()函数是一个判别式。

判别式是纯函数的理由：STL传递是按值传递，所以如果依赖判别式依赖外界参数可能会导致出错。

## 32.若一个类是函数子，应该让它可配接

```c++
list<widget*>widgetPtrs;//一个包含Widget对象指针的list容器
bool isInteresting(const Widget* pw);//判断某个Widget指针所指对象是否足够有趣
//找第一个有趣的widget
list<Widget *>::iterator  i=find_if(widgetPtrs.begin().widgetPtrs.end(),isInteresting);
//如果想第一个不有趣的widget
list<Widget *>::iterator  i=find_if(widgetPtrs.begin().widgetPtrs.end(),not1(isInteresting);//错误
list<Widget *>::iterator  i=find_if(widgetPtrs.begin().widgetPtrs.end(),not1(ptr_fun(isInteresting));//正确
 //not1是函数配接器，ptr_fun的作用是完成类型定义
//只有提供必要类型定义的函数对象被称为可被配接的函数对象
//提供这些类型定义最简单的方式就是让函数子从特定的基类继承。
//函数子operator只有一个参数就从std::unary_function继承
//函数子两个参数就从std::binary_function继承
//unary_function和binary_function是模板，不能直接继承，需要提供类型实参
```

```c++
//operator一个参数继承std::unary_function
template<typename T>
//提供operator第一个参数的类型和返回值类型
class MeetsThreshold:public std::unary_function<widget,bool>{
private:
	const T threshold;
public:
	bool operator()(const Widget&) const;
}

//operator两个参数的继承
//当函数子不需要任何私有成员
//提供operator第一个参数的类型,第二个参数的类型和返回值类型
//operator是带引用和const而模板没有，传递给unary_function,binary_function非指针类型需要去掉const 和 &
struct WidgetNameComapre:public std::binary_function<Widget,Widget,bool>{
    bool operator()(const Widgets& lhs,const widget&rhs)const;
}
//对于指针类型 则是可以
struct WidgetNameComapre:public std::binary_function<const Widget *,const Widget*,bool>{
    bool operator()(const Widgets* lhs,const widget* rhs)const;
}

//类型定义后的子类可以直接只有函数配接器
list<widget>::reverse_iterator i=find_if(widgets.begin(),widgets.end(),not1(MeetsThreshold<int>()))s
```

## 33.理解ptr_fun,mem_fun和mem_fun_ref的由来

```c++
f(x);//f是一个非成员函数
x.f();//f是成员函数，并且x是一个对象或者对象的引用
p->f();//f是成员函数，并且p是一个指向对象x的指针
void test(widget &w);
vector<widget>vw;
for_each(vw.begin(),vw.end(),test);//可以通过编译
class widget{
    public:
    ...
    void test();
    ...
}
for_each(vw.begin(),vm.end(),&widget::test);//不能通过编译
list<widget *>lpw;
for_each(lpw.begin(),plw.end(),&widget::test);//不能通过编译
//原因 for_each算法是基于非成员函数实现的
//for_each算法的实现
template<typename InputIterator ,typename Function>
Function for_each(InputIterator begin,InputIterator end,Function f){
    while(begin!=end)
        f(*begin++);
}
//mem_fun,mem_fun_ref用来调整使能够被成员函数使用
//mem_fun的定义
template<typename R,typename C>
mem_fun<R,C>;//C是类，R是指向成员函数的返回类型
mem_fun(R(C::*pmf)());
//mem_fun带一个指向某个成员函数的指针参数pmf，并且返回一个mem_fun_t类型的对象。mem_fun_t是一个函数子类，它拥有该成员函数的指针，并提供operator函数，在operator函数中调用通过参数传递进来的对象上的成员函数
for_each(lpw.begin(),lpw.end(),mem_fun(&widget::test));//可以通过编译
```

## 34.算法调用优于手写的循环

- 效率：算法通常比程序员自己写的循环效率高
- 正确性：自己写的循环比使用算法更容易出错
- 可维护性：使用算法的代码通常比手写循环的代码更加简洁明了。

如果要做的工作有算法实现就用算法。但是如果使用循环很简单，而使用算法实现的话却要求使用绑定器和配接器或者要求一个单独的函数子类，就使用循环。

## 35.容器的成员函数优于同名算法

- 成员函数往往速度很快
- 成员函数通常和容器结合的更加紧密

```
set<int>s;
...  //插入一百万个值
set<int>::iterator i=s.find(727);//以对数时间运行
if(i!+s.end())...
set<int>::iterator i=find(s.begin(),s.end(),727);//使用find算法 以线性时间运行

```

## 35.正确区分count,find,binary_search,lower_bound,upper_bound和equal_range



- 对于区间没有排序的，应当使用count和find，它们是线性时间，使用**相等性**查找

  ```
  vector<int>s;
  ```

  count是找区间中存在某个特定值的个数

  ```
  int num=count(s.begin,s.end(),5);//查找5在区间存在的个数
  ```

  find 是找区间中第一个特定值的位置

  ```
  int index=count(s.begin,s.end(),5);//查找第一个5的位置
  ```

- 对于区间排序的，可以使用binary_search,equal_range,lower_bound，对数时间,使用**等价性**查找

  ```
  vector<widget>vm;
  ...
  wdiget w;
  ```

  binary_search，只回答是否存在的问题

  ```
  bool exist=binary_search(vm.begin(),vm.end(),w);//查看5是否存在
  ```

  lower_bound，返回一个迭代器，如果存在返回第一份的拷贝，如果不存在返回适合于插入该值的位置

  ```
  vector<widget>::iterator it=lower_bound(vm.begin(),vm.end(),w);
  ```

  equal_range,返回一对迭代器，第一个迭代器等于lower_bound返回的迭代器，第二个迭代器等于upper_bound（指向区间内最后一个等价元素的下一个位置）返回的迭代器

  ```c++
  typedef vector<widget>::iterator VMIter;
  typedef pare<VMIter,VMIter> VMIterPair;
  VMIterPair p=equal_range(vm.begin(),vm.end(),w);
  //两个迭代器之间的距离，即使原始区间查找等价对象的数目
  if(p.first!=p.second){
  //说明存在
  }
  {
  //说明不存在，两个迭代器都指向w的插入位置
  }
  ```

  ![image-20220918171324230](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220918171324230.png)

## 36.避免直写型的代码

假如有一个vector<int>，现在想删除其中所有其值小于x的元素，但是，在最后一个其值不小于y的元素之前的所有元素应该保留下来。

```c++
vector<int> v;
int x,y;
...
v.erase(remove_if(v.rbegin(),v.rend(),bind2nd(greater_equal<int>(),y)).base(),v.end(),bind2nd<less<int>>(),x)),v.end());
//可读性太差,不利于后来者维护
//进行分解
typedef vector<int>::iterator VecIntIter;
//初始化rangeBegin,使它指向v中大于等于y的最后一个元素之后的元素
//如果不存在这样的元素，则rangeBegin被初始化为v.bgein()
//如果这样的元素正好是V最后一个元素，则rangeBegin被初始化为v.end()
VecIntIter rangeBegin=find_if(v.rbegin(),v.rend(),bind2nd(greater_equal<int>(),y)).base();
//从rangeBegin到v.end()的区间中删除所有小于x的值
v.erase(remove_if(rangeBegin,v.end(),bin2nd(less<int>(),x)),v.end());
//bind2nd 将二元函数转为一元函数，因为函数内一个参数已经被确定
```

## 37.总是包含（#include）正确的头文件

因为不同的平台头文件之间包含关系不通，因此任何时间如果你使用某个头文件中的一个STL组件，一定要提供对应的#include 指令

**在一个声明为const的成员函数内部，该类所有的非静态数据成员都自动被转换为相应的const类型。**