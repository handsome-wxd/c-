[toc]

# STL源码分析

## STL概论与版本简介

### 1.STL六大组件 功能与运用

STL提供六大组件，彼此可以组合套用

容器：vector,deque,list,set,map

算法：sort,search,copy,erase

迭代器：扮演容器和算法之间的胶合剂，是所谓的“泛型指针”

仿函数：行为类似函数，可作为算法的某种策略（policy）。从实现角度来看，仿函数是一种重载operator（）的class或class template 。一般函数指针可视为狭义的仿函数。

配接器：一种用来修饰容器或仿函数或迭代器接口的东西。

配置器：负责空间配置和管理。从实现角度来看，配置器是一个实现动态空间配置、管理空间、空间释放的class template。

![image-20220920155957454](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220920155957454.png)

### 2.对一个类定义前置操作和后置操作

```c++
#incldue<iostream>
using namespce std;
//定义INT 类
class INT{
    friend ostream& operator<<(ostream &os,const INT * i);
    public:
    INT(int i):m_i(i);
    //定义前置++ 操作
    INT& operator++(){
        ++(this->m_i);
        return *this;
    }
    //定义后置++ 操作
    INT& operator++(int){
        INT temp=*this;
        ++(*this);
        return temp;
    }
     //定义前置-- 操作
    INT& operator++(){
        --(this->m_i);
        return *this;
    }
    //定义后置-- 操作
    INT& operator++(int){
        INT temp=*this;
        --(*this);
        return temp;
    }
   int& operator*() const{
       return (int &)m_i;
   }
   private:
    int m_i;
};
ostream& operator<<(ostream& os,const INT& i){
    os<<'{'<<i.m_i<<']';
    return os;
}
```



## 空间配置器

### 1.SGI 特殊的空间配置器，std::alloc

```c++
class Foo{...};
Foo *pf=new Foo;//调用 opeator new 配置内存 ，在调用Foo:Foo()构造对象内容
delete pf;//先析构，然后用::operator delete释放内存
```

为了紧密分工,STL allocator将这两阶段操作区开，内存配置操作由alloc::allocate()负责，内存释放由alloc::deallocate(）负责；对象构造操作由::construct()负责，对象析构操作由::destory()负责。

SGI<memory>内容包含一下两个文件：

```c++
#include<stl_alloc.h>//负责内存空间的配置和释放
#include<stl_construct.h>//负责对象内容的构造和析构
```

![image-20220920183554633](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220920183554633.png)

**对象的构造和析构基本工具：construct()和destory()**

```c++
#include<new.h>
template<class T1,class T2>
inline void construct(T1* p,const T2& value){
new (p) T1(value);
}
//以下是destory()第一版本，接受一个指针
template<class T>
inline void destory(T* pointer){
    pointer->~T;
}
//destory 第二版本，接受两个迭代器
template<class ForwardIterator>
inline void destory(ForwardIterator first,ForwardIterator last){
    _destory(first,last,value_type(first));
}
//判断元素数值类别（value_type）是否由trivial destructor
template<class ForwardIterator,class T>
inline void _destory(ForwardIterator first,ForwardIterator last,T*){
    typedef typename _type_traits<T>::has_trivial_destructor trivial_destructor;
    _destory_aux(first,last,trival_destructor());
}
//数据类型为non_trivial destructor,对所有元素进行析构，
template<class ForwardIterator>
inline void _destory_aux(ForwardIterator first,ForwardIterator last,_false_type){
    for(;first<last;+first)
        destory(&*first);
}
//数据类型为trivial destructor,对不进行析
template<class ForwardIterator>
inline void _destory_aux(ForwardIterator first,ForwardIterator last,_true_type){
}
```

**空间的配置和释放，std::alloc**

SGI是以malloc()和free()完成内存的配置和释放

当配置区块超过128bytes，视为足够大，直接调用第一级配置器，直接采用malloc()和free()

当配置区块小于128bytes，视为过小

![image-20220921194538433](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220921194538433.png)

无论alloc被定义为第一级配置器还是第二级配置器，SGI还可以为它再包装一个接口，使配置器接口能够复合STL规格

```c++
template<class T,class Alloc>
class simple_alloc{
public:
    static T *allocate(size_t n){
        return 0==n? 0:(T*) Alloc::allocate(n*sizeof(T));}
    static T *allocate(void){
        return (T*) Alloc::allocate(sizeof(T));}
    static T * deallocate(T *p ,size_t n){
       if(0!==n) Alloc::deallocate(p,n*sizeof(T));}
    static T *deallocate(T *p){
        Alloc::deallocate(p,sizeof(T));}
    }
    }
}
```

第一级配置器和第二级配置器的包装接口和运行方式

![image-20220921194629548](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220921194629548.png)



**第一级配置器  __malloc_alloc_template 剖析**

```c++
template<int inst>
class _malloc_alloc_template{
private:
    //以下都是函数指针，所代表的函数将用来解决内存不足的情况
    //oom: out of memory
    static void  *oom_malloc(size_t);
    static void *oom_realloc(void *,size_t);
    static void (*_malloc_allioc_oom_handler)();

public:
    static void * allocate(size_t n){
        void *result=malloc(n);//第一级配置器直接用malloc()
        //如果无法满足需求时，采用 oom_malloc()
        if(0==result) result=oom_malloc(n);
        return result;
    }
    static void * deallocate(void * p,size_t /* n */){
        free(p);//第一级适配器直接用free
    }
    static * void reallocate(void *p,size_t /* old_sz */,size_t new_sz){
        //realloc:更改已经配置的内存空间，即更改由malloc()函数分配的内存空间的大小
        void * result=realloc(p,new_sz);
         //如果无法满足需求时，采用 oom_malloc()
        if(0==result) result=oom_malloc(p,new_sz);
        return result;
    }
    
    //以下是仿真c++的set_new_handler()，用它指定自己的out-of-memory handler
    static void (*set_malloc_handler(void (*f)())) (){
        void (* old)()=_malloc_alloc_oom_handler;
        _malloc_alloc_oom_handler=f;
        return (old);
    }
    //malloc_alloc out-of-memory handing,初始值为0，有待客端设定
    template<int inst>
    void (*_malloc_alloc_template<inst>::_malloc_alloc_oom_handler)()=0;
    
    tempalte<int inst>
    void *_malloc_alloc_template<inst>::oom_malloc(size_t n){
        void (* my_malloc_handler)();
        void *result;
        for(;;){//不断尝试释放配置，再释放，再配置...
            my_malloc_handler=_malloc_alloc_oom_handler;
            if(0==my_alloc_handler){_THROW_BAD_ALLOC;}//当内存不足例程未被客端设定，将抛出异常
            {*my_alloc_handler}();//调用处理例程，企图释放内存
            return =malloc(n);  //再次尝试配置内存
            if(result) return (result);
        }
    }
    
    tempalte<int inst>
    void *_malloc_alloc_template<inst>::oom_remalloc(void *p,size_t n){
        void (* my_malloc_handler)();
        void *result;
        for(;;){//不断尝试释放配置，再释放，再配置...
            my_malloc_handler=_malloc_alloc_oom_handler;
            if(0==my_alloc_handler){_THROW_BAD_ALLOC;}
            {*my_alloc_handler}();//调用处理例程，企图释放内存
            return =remalloc(p,n);  //再次尝试配置内存
            if(result) return (result);
        }
    }    
}
```

**第二级配置器 _default_alloc_template 剖析**

第二级配置器多一些机制，避免太多小额区间造成内存的碎片。小额区级带来的不仅是内存碎片，配置时额外负担也是大问题。区间越小，额外负担所占比例就越大。

![image-20220921202431077](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220921202431077.png)

当区块够大，超过128bytes就移交给第一级配置器，当小于128bytes则以内存池（memory pool）管理。称为次层配置：每次配置一大块内存，并维护对应之自由链表。为了方便将内存需量求调整至**8**的倍数，（客端需要30bytes，自动调整为32bytes），并维护16个free-lists。各自管理大小为8-128bytes。当需要内存时，就用free-lists拔出，释放了由配置器归还到free-lists。

![image-20220921203215244](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220921203215244.png)

```c++
//第二级配置器部分实现内容
enum{_ALIGN =8};//小型区块的上调边界
enum{_MAX_BYTES=128}; //小型区块的上限
enum{_NFREELISTS =_MAX_BYTES/_ALIGH}//free_list个数
template<bool threads,int inst>
class _default_alloc_template{
private:
//ROUND_UP 将bytes上调至8的倍数
static size_t ROUND_UP（size_t bytes）{
	return (((bytes)+_ALIGN-1)& ~(_ALIGN-1));
}
private :
	union obj{ //free_list的节点构造
	union obk * free_list_link;
	char client_data[1];
	}
private:
//16个free_listes
static obj *volatile free_list[_NFREELISTS];
//以下函数根据区块大小，决定使用第n号free_list.n从1起算
static size_t FREELIST_INDEX(size_t bytes){
return (((bytes)+_ALIGN-1)/_ALIGN-1);
}
//返回大小为n的对象，并可能加入大小为n的其他区块到free list
static void *refill(size_t n);
//配置一大块空间，容纳nobjs个大小为“size"的区块
//如果配置nobjs个区块有所不便，nobjs 可能会降低
static char *chunk_alloc(size_t size,int *nobjs);

//chunk allocation state
static char *start_free; //内存池起始位置，只在chunk_alloc()中变化
static char *end_free; //内存池结束位置，只在chunK_alloc()中变化
static size_t heap_size;

public:
static void * allocate(size_t n){....}
static void deallocate(void *p,size_t)
static size_t heap_size;
}
```

**空间配置函数allocate()**

判断区块大小，大于128bytes就调用第一级配置器，小于128bytes就检查对应的free  list。如果free list之内由可用的区块，就直接拿来用，如果没有可用的区块，就直接将区块大小上调至8倍数边界，然后调用refill()，准备为free list 重新填充空间

```c++
static void * allocate(size_t n){
 obj * volatile * my_free_list;
 obj * result;
 //如果大于128就调用第一级配置器
 if(n>(size_t)_MAX_BYTES){
   return (malloc_alloc::allocate(n));
 }
 //寻找16个free lists中适当的一个
 my_free_list =free_list+FREELIST_INDEX(n);
 result-*my_free_list;
 if(result ==0){
 //没找到可用的free list ，准备填充free list
 void * r=refill(ROUND_UP(n));
 return r;
 }
 //调整free list
 *my_free_list=result->free_list_link;//指向下一个
 return (result); 
}
```

![image-20220921213459043](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220921213459043.png)



**空间释放函数dealllocate()**

```c++
static void * deallocate(void *p,size_t n ){
 obj * q =(obj *)p;
 obj * volatile *my_free_list;
 //如果大于128就调用第一级配置器
 if(n>(size_t)_MAX_BYTES){
   return (malloc_alloc::deallocate(p,n));
 }
 //寻找16个free lists中适当的一个
 my_free_list =free_list+FREELIST_INDEX(n);
 result-*my_free_list;
 //调整free list
 q->free_list_link=*my_free_list;//指向下一个
 *my_free_list=q;
}
```

![image-20220921214556723](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220921214556723.png)

**重新填充free lists**

 	当free list中没有可用的区块，调用refill()，准备为free list 重新填充空间。新的空间将取自内存池（经由chunk_alloc()完成）。

```c++
//返回一个大小为n的对象，并且有时候会为适当的free list增加节点
//假设n已经适当上调至8的倍数
template<bool threads,int inst>
void *_default_alloc_template<threads,inst>::refill(size_t n){
     //n 是指要的内存字节大小 可能为8 16 24 
    int nobjs=20; // 指取几块
    //调用chunk_alloc()，尝试取得nobjs 个区块作为free list的新节点
    //注意参数 nobjs 是pass by reference
    char *chunk=chunk_alloc(n,nobjs);
    objs * volatile *my_free_list;
    obj * result;
    obj * current_obj,*next_obj;
    int i;
    //如果只获得一个区块，这个区块就分配调用，free list 无新节点
    if(1==nobjs) return （chunk）;
    my_free_list=free_list+PREELIST_INDEX(n);
    //以下再chunk 空间内创建 free list
    result=(obj *)chunk; //这一块准备返回给客端
    //以下导引free list 指向新配置的空间
    *my_free_list=next_obj
    //以下将free list 的个节点串接起来
    for(i=1;;i++){
       
        current_obj=next_obj;
        next_obj=(obj *)((char *)next_obj+n);
        if(nobjs-1==i){
            current_obj->free_list_link=0;
            break;
        }else{
            current_obj->free_list_link=next_obj;
        }
        }
    }
    return (result);
}
```

**内存池（memory pool）**

从内存池中取空间给free list使用，是chunk_alloc()的工作：

```c++
//假设size 已经适当上调至8的倍数
template<bool threads,int inst>
char * _default_alloc_template<thread ,inst>::chunk_alloc(size_t size,int &nojs){
char * result;
size_t total_bytes=size*nobjs;
size_t bytes_left=end_free_start_free;//内存池剩余空间
if(byetes_left>=toal_bytes){
    //内存池剩余空间完全满足需求量
    result=start_free;
    start_free+=total_bytes;
    return result;
}
else if(bytes_left>=size){
	//内存池剩余空间不能完全满足需求量，但是足够供应一个含以上的区块
	nobjs=bytes_left/size;
	total_bytes=size*nobjs;
	result=start_free;
	start_free+=total_bytes;
	return (result);
}else{
	//内存池剩余空间连一个区块的大小都无法提供
    //从heap中配置内存，要求的内存需求是要配置的2的倍
	size_t bytes_to_get=2*total_bytes+ROUND_UP(heap_size>>4);
	//以下试着让内存池中的残存零头还有剩余价值
	if(bytes_left>0){
		//内存池内还有一些零头，先配给适当的free list
		//首先先寻找适当的free list
		obj * volatile 8 my_free_list=free_list+FREELIST_INDEX[bytes_left];
		//调整free list 将内存池中残余空间插入
		((* obj)start_free)->free_list_link=*my_free_list;
		*my_free_list=(obj *)start_free;
	}
	//配置heap空间，用来补充内存池
	start_free=(char *)malloc(bytest_to_get);
	if(0==statrt_free){
		//heap 空间不足，malloc()失败
		int i;
		obj 8 volatile * my_free_list *p;
		//试着检视我们手上的东西，这不会造成伤害，我们不打算尝试配置较小的区块，因为多进程机器上容易造成灾难
		//以下搜寻适当的free list
		//所谓适当就是 ”尚有未用区块，且区块足够大“知free list
		for(i=size;<_MAX_BYTES;I+=_ALIGN){
			my_free_list=free_list+FREELIST_INDEX(I);
			p=*my_free_list;
			if(0!=p){//free list 内尚有未用的区块
				*my_free_list=p->free_list_link;
				statrt_free=(char *)p;
				end_free=start_free+i;
				return (chunk_alloc(size,nobjs));
			}
		}
		end_free=0;//没有一点内存可以用
		//调用第一级配置器，来看看out-of-momory机制是否可以尽点力
		start_free=(char*)malloc_alloc::allocate()bytest_to_get);
		//这会抛出异常，或者内存不足会情况获得改善
	}
	heap_size+=bytes_to_get;
	end_free=start_free+bytes_to_get;
	return (chunk_alloc(size,nobjs));
}
}
```

具体流程是，当内存池内空间足够时就将这个调用20个区块返回给free_list(一个交给客端，19个维护)。当内存池空间不足二十个但是大于1个就将不足20的空间交给free_list。如果内存池的空间一个区块的空间都没有，需要利用malloc()从heap中重新配置内存，新配置的内存为需求量的二倍，就是40个区块空间。当整个system heap空间不够时，就去找是否有大于区块但是未用的free list，找到就挖一块交出去，找不到就调用一级配置器，一级配置器有out-of-memory机制，有机会释放其他内存拿来给自己用。如果失败就报异常。

### 2.内存基本处理工具

uninitialized_copy(),uninitialized_fill(),uninitialized_fill_n(),分别对应高层次函数copy(),fill(),fill_n()。

**uninitilaized_copy**

```c++
template<class InputIterator,class ForwardIterator>
FrowardIterator uninitialized_copy((InputIterator first,InputIterator last,ForwardIterator result);
```

uninitialized_copy()使将内存配置和对象的构造行为分离开。输入目的地[result,result+(last-first)]范围内每个迭代器都指向未初始化的区域，则uninitialized_copy（）会使用**copy constructor给身为输入来源[first,last）范围内每个对象产生一份复制品**，放入输出范围中。

容器的全局构造函数range constructor以两个步骤完成：

- 配置内存区块，足以包含范围内所有元素
- 使用uninitialized_copy()，再该内存上构造元素

**uninitialized_fill**

```c++
template<class InputIterator,class ForwardIterator>
void uninitialized_fill((InputIterator first,InputIterator last,const T&x);
```

uninitialized_fill()使将内存配置和对象的构造行为分离开。输入目的地**[First,last)**]范围内每个迭代器都指向未初始化的区域，则uninitialized_fill（）会使用copy constructor给**[first,last）**范围内每个对象产生一份**x复制品**，放入输出范围中。

**uninitialized_fill_n**

```c++
template<class ForwardIterator,class Size,class T>
ForwardIterator uninitialized_fill_n((ForwardIterator first,Size n,const T&x);
```

uninitialized_fill()使将内存配置和对象的构造行为分离开。输入目的地[**first,first+n)**范围内每个迭代器都指向未初始化的区域，则uninitialized_fill（）会使用copy constructor给**[first,first+n）**范围内每个对象产生一份**x复制品**，放入输出范围中。

这三个都是支持回滚的，要不产生所有必要元素，要不不产生任何元素。

这三个函数对于元素类别采用不通的方法创造复制品，对于POD（传统C struct）类型，直接采用复制方法。对于非POD采用构造方法一个一个构造。

```c++
//例如**uninitilaized_copy**
template<class InputIterator ,class ForwardIterator,class T>
inline ForwardIterator _uninitialized_copy(InputIterator first,InputIterator last,ForwardIteratro result,T*){
typedef typename _type_trait<T>::is_PO_type is_POD;
return _uninitalized_copy_aux(first,last,result,is_POD());
}
//POD类型
template<class InputIterator ,class ForwardIterator>
inline ForwardIterator uninitalized_copy_aux(InputIterator first,InputIterator last,ForwardIteratro result,_true_type){
return copy(first,last,result,result);
}
//非POD类型
template<class InputIterator ,class ForwardIterator>
inline ForwardIterator uninitalized_copy_aux(InputIterator first,InputIterator last,ForwardIteratro result,_false_type){
ForwardIterator cur=reslut;
for(;first!=last;++first;++cur)
    construct(&*cur,&first);
    return cur;
}
```

三个内存基本函数的泛型版本与特化版本

![image-20220922153942962](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220922153942962.png)



## 迭代器

STL的中心思想是将数据容器和算法分开，彼此独立设计，最后再以一贴粘合剂将它们撮合在一起。迭代器就是粘合剂。

### 1.迭代器相应型别

如果想在算法中申明一个变量，以”迭代器所指对象的型别“

直接用function template 参数推导机制

```c++
template <class I,class T>
void func_impl(I iter,T t){
	T temp;// 定义参数
	//..这里做func()应该做的事
}；
template<class I>
inline void func(I iter){
func_impl(iter,*iter);
}
int main(){
    int i;
    func(& i);
}
```

但是对于指定返回值类型就不行了，因为template 参数推导不能推导返回值类型

采用声明嵌套类的方法

```c++
template<class T>
struct MyIter{
    typedef T value_type;//内嵌申明
    T* ptr;
    MyIter(T* p=0):ptr(p){}
    T& operator*() const {return *ptr;}
    //...
};
template<class T>
typename I::valye_type func(I iter){
    return *iter;
}
//...
MyIter<int> ite(new int(8));
cout<<func(ite);
```

但是不是所有的迭代器都是 class type，对于非class 无法在内部定义

采用template partial specialization

```c++
//将这个类型的定义写在外面
//traits 扮演特性萃取机的角色，这个特性s就是迭代器相应类别
template<class I>
struct iterator_traits<I *>{
typedef I value_type;
}
//针对指向常数对象的指针
template<class T>
strcut iteraotr_traits<const T*>{
    typedef T value_type;
};
```

![image-20220922170357965](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220922170357965.png)

常用迭代器相应类别有五种:**value type,difference type,pointer,reference,iterator catagoly**

**iterator_traits负责萃取迭代器的特性**

```
template<class I>
struct iterator_traits{
	typedef typename I::iterator_category iterator_category;
	typedef typename I::value_type value_type;
	typedef typename I::difference_type difference_type;
	typedef typename I::pointer pointer;
	typedef typename I::refernce;
}
```

***value type***

value type 是指迭代器所指对象的类型

**difference type**

difference type 用来表示两个迭代器之间的距离，用来表达一个容器的最大容量

**reference type**

```c++
int *pi=new int(5);
const int 8pci=new int(90);
*pi=7; //对mutable iterator 进行解引用，获得是一个左值，允许赋值
*pci=1;//不容许 ，pci是一个constant iterator
```

p是mutable  iterator其value type是T，但是*P是T&，p如果是一个constant iterator其value type是T，但是*P是 const T&;这就是 reference type

**point type**

```c++
Item& operator*() const{return *ptr;}
Item* operator->() const{return ptr;}
//Item& 是reference type，Item* 是point type
```

**iterator_category**

定义的是iterator的类别。

`Input Iterator,Output Iterator,Forward Iterator,Bidirectional Iterator,Random Access Iterator`

直线和箭头并非c++的继承关系 而是concept(概念)与refinement(强化)的关系。

![image-20220922191320119](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220922191320119.png)

```c++
//对于迭代器函数例如 distance()不同的迭代器有不同的实现方式，因此需要用标明迭代器的类型去针对不同迭代器 选择不同实现
struct input_iterator_tag{};
struct output_iterator_tag{};
struct forward_iterator_tag:public input_iterator_tag{};
struct bidirectional_iterator_tag:public forward_iterator_tag{};
struct random_access_iterator_tag:public bidirectional_iterator_tag{};
//不同迭代器实现advance
//Inputator
template<class InputIterator ,class Distance>
inline void _advance(InputIterator& i,Distance n,input_iterator_tag){
    while(n--) ++i;
}
//RandomAccessIterator
template<class RandomAccessIterator ,class Distance>
inline void _advance(RandomAccessIterator& i,Distance n,random_access_iterator_tag){
    i=+n;
}
//advnace 对外上层接口,以Iterator 为模板参数是STL算法的一个命名规则，算法所能接受之最低阶迭代器类型来为迭代器型别参数命名。
template<class InputIterator,class Distance>
inline void advance(InputIterator& i,Distance n){
    _advance(i,n,iterator_traits<InputIterator>::iterator_category());
}
```

### 2.__type_traits

_type_traits负责萃取型别特性

对于**uninitilaized_copy**,**uninitialized_fill**,**uninitialized_fill_n**针对不同的类型采用不同的构造方法。

```c++
struct _true_type{};//定义成结构体是为可以直接从名字看出 它表达的意思
struct _flase_type{};
//因此在type_traits中定义一些typedefs,其类型不是_true_type就是_false_type
template <class type>
struct _type_traits{
    typedef _true_type this_dummy_member_must_be_first;//它是通知有能力自动将_type_traits特化的编译器说，我们现在看到这个_type_traits template是特殊的
    //默认全是_flase_type
    typedef _false_type has_trivial_default_constructor;
    typedef _false_type has_trivial_copy_constructor;
    typedef _false_type has_trivial_assignment_operator;
    typedef _false_type has_trivial_destructor;
    typedef _false_type is_POD_type;
}
```

对于一些基本类型提出特化_type_traits

```c++
//例如char
_STL_TEMPLATE_NULL struct _type_traits<char>{
	typedef _true_type has_trivial_default_constructor;
    typedef _true_type has_trivial_copy_constructor;
    typedef _true_type has_trivial_assignment_operator;
    typedef _true_type has_trivial_destructor;
    typedef _true_type is_POD_type;
}
```

实际使用

```c++
template<class InputIterator ,class ForwardIterator,class T>
inline ForwardIterator _uninitialized_copy(InputIterator first,InputIterator last,ForwardIteratro result,T*){
typedef typename _type_trait<T>::is_PO_type is_POD;
return _uninitalized_copy_aux(first,last,result,is_POD());
}
```



## 序列式容器

![image-20220925201623294](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220925201623294.png)

### 1.vector

vecot源码摘录

```c++
template<class T,class Alloc * alloc>
class vecotr{
	public:
	//vector的嵌套型别定义
	typedef T value_type;
	typedef value_type* pointer;
	typedef value_type* iterator;
	typedef value_type& reference;
	typedef size_t size_type;
	typedef ptrdiff_t difference_type;
	protected:
	typedef simple_alloc<value_type,Alloc> data_allocator;
	iterator start; //表示目前使用空间的头
	iterator finish; //表示目前使用空间的尾
	iterator end_of_storage; //表示目前可用空间的尾;
	
	void insert_aux(iterator poisition,const T& x);
	void deallocate(){
	if(start)
		deallocator::deallocate(start,end_of_storage-start);
	}
	
	void fill_initalize(size_type n,const T& value){
		start=allocate_end_fill(n,value);
		finish=start+n;
		end_of_storage=finish;
	}
	public:
	iterator begin() {return start;}
	ierator end() {return finish;}
	size_type size() const {return size_type(end()-begin());}
	size_type capacity() const{
		return size_type(end_of_storage-begin());}
	bool empty() const {return begin()==end();}
	reference operator[](size_type n){reutrn *(begin()+n);}
    
    vector():start(0),finish(0),end_of_storage(0){}
    vector(size_type n,const T& value){fill_initialzie(n,value);}
    vector(int n,const T&value){fill_initilaize(n,value);}
    vector(long n,const T&value){fill_initilaize(n,value);}
    explicit vector(size_type n){fill_initialize(n,T());}
    
    ~vector(){
        destory(start,finish);
        deallocate();
    }
    reference front() {return *begin();}
    reference back() {return *(end()-1);}
    void push_back(const T&x){
        if(finish!=end_of_storage){
            construct(finish,x);
            ++finish;
        }
        else
            insert_aux(end(),x);
    }
   void pop_back(){
       --finish;
       destroy(finish);
   }
   iterator erase(iterator position){
       if(position+1!=end())
           	copy(position+,finish,position);
       --finish;
       destory(finish);
       return position;
   }
   void resize(size_type new_size,const T&x){
       if(new_size<size())
           erase(negin()+new_size,end());
       else
           insert(end(),new_size-size(),x);
   }
    void resize(size_type new_size){resize(new_size,t());}
    void clear() {erase{begin(),end()};}
	}
	protected:
	iterator allocate_and_fill(size_type n,const T& x){
        ierator result =data_allocator::allocate(n);
        uninitalizeed_fiil_n(result,n,x);
        return result;
    }
}
```

![image-20220925213451685](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220925213451685.png)

**vecotr 内存扩张代码**

```c++
//vecotr 内存扩张代码
template<class T,class Alloc>
void vecotr<T,Alloc>::insert_aux(iterator position,const T& x){
	if(finish!=end_of_storage){//还有备用空间
	//在备用空间起始位置构造一个元素，并以vector最后一个元素指为初始值
	construct(finish,*(finish-1));
	++finish;
	T x_copy =x;
	copy_backward(position,finish-2,finish-1);
	*position =x_copy;
	}
	else{//无备用位置
		//并不是在原来的位置接续新空间，而是找一块大的空间，将原来的元素复制过来，
		//因此对vector的任何操作，一旦引起空间重新配置，指向原vector的所有迭代器就都失效了。
		const size_type old_size=size();
		const size_type len=lod_size!=0? 2*len:1;
		iterator new_start =data_allocator::allocate(len);//实际配置
		iterator new_finish=new_start;
		try{
			new_finish=uninitilaized_copy(start,position,new_start);
			construct(new_finish,x);
			++new_finish;
			new_finish=uninitialized_copy(position,finish,new_finish);
		}
		catch(...){
			//支持回滚
			destory(new_start,new_finish)
			data_allocator::deallocate(new_start,len);
            throw;
		}
		//析构并并释放原来vecotr
		destory(begin(),end());
		deallocate();
		//调整迭代器，指向新vector
		start=new_start;
		finsih=new_finish;
		end_of_stroage=new_start_len;
	}
}
```

**vector::insert()实现内容**

```c++
template<class T,class Alloc>
void vector<T,Alloc>::insert(iterator position,size_type n,const T&x){
    if(n!=0){
        if(size_type(end_of_storage-finish)>=n){
            T x_copy=x;
            const size_type elems_after=finish-position;
            iterator old_finish=finish;
            if(elems_after>n){
                //插入节点之后的现元素的个数大于新增元素个数
                uninitalized_copy(finish-m,finish,finish);
                finish+=n;
                copy_backward(position,old_finish-n,old_finish);
                fill(position,position+x,x_copy);
            }else{//备用空间小于新增元素个数
                //首先决定新长度，旧长度的两倍，或旧长度+新增元素个数
                const size_type old_size=size();
                const size_type len=old_size+max(old_size,n);
                iterator new_start=data_allocator::allocate(len);
                iterator new_finish=new_start;
                _STL_TRY{
                    new_finsih=uninitialized_copy(start,position,new_start);
                    new_finish=uninitialized_fill_n(new_finish,n,x);
                    new_finish=uninitialized_copy(position,finish,new_finish);
                }
                catch(...){//回滚
                    destory(new_start,new_finish)
					data_allocator::deallocate(new_start,len);
           			 throw;
                }
                destory(start,finish);
                deallocate();
                start=new_start;
                finish=new_finish;
                end_of_storage=new_start+len;
            }
        }
    }
}
```



## 关联式容器

## 算法

## 仿函数

## 配接器