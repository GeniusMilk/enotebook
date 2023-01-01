#STLCard
#八股文
#Anki

#C++
#C++的标准库


#myanki

```ActivityHistory
/
```

```toc 
	style: bullet | number | inline (default: bullet) 
	min_depth: number (default: 2) max_depth: number (default: 6) 
	title: string (default: undefined) 
	allow_inconsistent_headings: boolean (default: false) 
	delimiter: string (default: |) 
	varied_style: boolean (default: false) 
```

参考链接
[STL详解及常见面试题_~青萍之末~的博客-CSDN博客_stl面试题](https://blog.csdn.net/daaikuaichuan/article/details/80717222)

### 一、STL的介绍
Standard Template Library，标准模板库，是C++的标准库之一，一套基于模板的容器类库，还包括许多常用的算法，提高了程序开发效率和复用性。

STL包含6大部件：
==容器==、==迭代器==、==算法==、==[[C++ Lambda 表达式 详解#4 2 C 仿函数|仿函数]]、适配器==、==空间配置器==。 <!--SR:!2022-08-30,8,166!2022-09-10,13,204!2022-09-01,4,164!2022-09-05,8,184!2022-09-10,13,204-->

容器：::容纳一组元素的对象。 <!--SR:!2022-09-05,14,206-->

迭代器：::提供一种访问容器中每个元素的方法。 <!--SR:!2022-09-08,17,226-->

函数对象：::一个行为类似函数的对象，调用它就像调用函数一样。 <!--SR:!2022-09-02,7,138-->

算法：::包括查找算法、排序算法等等。 <!--SR:!2022-09-02,11,184-->

适配器：::用来修饰容器等，比如[[算法与数据结构 Card|queue]]和[[算法与数据结构 Card|stack]]，底层借助了[[算法与数据结构 Card|deque]]。 <!--SR:!2022-09-17,19,184-->

空间配置器：::负责空间配置和管理。 <!--SR:!2022-09-02,11,190-->

![[C++ Lambda 表达式 详解#4 2 C 仿函数|仿函数]]

### 二、空间配置器详解
对象构造前的空间配置和对象析构后的空间释放，由<stl_alloc.h>负责
-   向==system heap==要求空间。
-   考虑==[[OS Card#多线程|多线程状态]]==。
-   考虑==[[OS Card#OS的内存管理|内存不足]]==时的应变措施。
-   考虑过多“小型区块”可能造成的==[[OS Card#OS的内存管理|内存碎片]]问题==。 <!--SR:!2022-09-03,12,184!2022-08-31,3,135!2022-09-09,12,195!2022-09-06,9,175-->

#### SGI双层级配置器：
#线索 
?
考虑小型区块造成的内存破碎问题，
第一级直接使用allocate()调用malloc()、deallocate()调用free()
第二级视情况使用不同的策略 <!--SR:!2022-09-10,19,238-->

##### 第一级 直接使用allocate()
?
直接使用allocate()调用malloc()、deallocate()调用free()，使用类似new_handler机制解决内存不足（抛出异常），配置无法满足的问题（如果在申请动态内存时找不到足够大的内存块，malloc 和new 将返回NULL 指针，宣告内存申请失败）。 <!--SR:!2022-09-12,21,244-->

##### 第二级视情况使用不同的策略，
?
当配置区块大于128bytes时，调用第一级配置器，当配置区块小于128bytes时，采用内存池的整理方式：配置器维护16个（128/8）自由链表，负责16种小型区块的此配置能力。内存池以malloc配置而得，如果内存不足转第一级配置器处理。 <!--SR:!2022-10-03,36,244-->

#### 1、第一级配置器详解

<center>
	<img style="border-radius: 0.3125em;
	 box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWFnZXMyMDE1LmNuYmxvZ3MuY29tL2Jsb2cvMTA5MjE2NS8yMDE3MDIvMTA5MjE2NS0yMDE3MDIyODIzMDQyODExMC0xMDY1NDA4MTk5LnBuZw?x-oss-process=image/format,png"> 
	 <br> 
	 <div style="color:orange; border-bottom: 1px solid #d9d9d9; 
	 display: inline-block;
	 color: #999; 
	 padding: 2px;">  第一级配置器 UML </div>

</center>
-   一级配置相对简单，直接封装了`malloc()`、`realloc()` 和 `free()`,并实现了类似C++ new-handler的机制（内存配置需求无法满足时，调用自己指定的函数），没有直接运用C++的，是因为没有使用new来配置内存。

```c++
template <int inst>
class __malloc_alloc_template {
private:
    //处理内存不足的情况
    static void (*__malloc_alloc_oom_handler)();

public:
    static void* allocate(size_t n) {
        void *result = malloc(n);
        if (result == 0) {
            /*不断的尝试释放，配置，释放, ···。直到配置成功*/
            result = oom_malloc(n);
        }
        return result;
    }
    static void deallocate(void *p, size_t) { free(p); }
    static void deallocate(void *p, size_t) {}
    static void reallocate(void *p, szie_t old_sz, size_t new_sz) {}
};
void (*__malloc_alloc_template<inst>::__malloc_alloc_oom_handler)() = 0;
```

-   oom_malloc逻辑如下

```C++
/*
 * 仿真C++ 的set_new_handler
 */
static void (*set_malloc_handler(void (*f)() ) ) () {
    void (*old)() = __malloc_alloc_oom_handler;
    __malloc_alloc_oom_handler = f;
    return old;
}
template<int inst>
void* __malloc_alloc_template<inst>::oom_malloc(size_t n) {
    void (*my_malloc_handler)();
    void *result;

    //不断尝试配置，释放，...
    for (;;) {
        my_malloc_handler = __malloc_alloc_oom_handler;
        //如果没有配置处理例程，则直接丢出bad_alloc异常
        if (0 == my_malloc_handler) {
            __THROW_BAD_ALLOC;
        }
        (*my_malloc_handler)();
        result = malloc(n);
        if (result)
            return result;
    }
}

注：
设计 内存不足处理例程，是客户端责任。
```

-   reallocate和allocate类似，这里不再累赘。


#### 2、第二级空间配置器详解

-   二级配置器，可以避免太多小额区块造==成[[OS Card#OS的内存管理|内存碎片]]==。
-   使用==内存池==管理
- 自由[[算法与数据结构 Card#|链表]]是一个指针[[算法与数据结构 Card|数组]]，有点类似与[[算法与数据结构 Card|TODO hash桶]]，它的数组大小为16，每个数组元素代表所挂的区块大小
<center>
	<img style="border-radius: 0.3125em;
	 box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWFnZXMyMDE1LmNuYmxvZ3MuY29tL2Jsb2cvODM1MjM0LzIwMTYwNi84MzUyMzQtMjAxNjA2MDMxOTAyNDI3MTEtNTIyMzI4MjU1LnBuZw?x-oss-process=image/format,png"> 
	 <br> 
	 <div style="color:orange; border-bottom: 1px solid #d9d9d9; 
	 display: inline-block;
	 color: #999; 
	 padding: 2px;">  第二级配置器 内存池</div> <!--SR:!2022-09-01,10,184!2022-09-02,5,164-->

</center>

#####  第二级空间配置器思路  
1.  先配置一块大内存，==[[|TODO内存池]]==。
2.  维护==自由[[算法与数据结构 Card#|链表]]==。空间的申请和归还会通过该链表。==配置器==会将小额空间调整为8的倍数。总共有16个自由链表依次为：8、16、24、32、···、128byte
3.  free_list中无可用的区块时，新的空间取自==内存池==。缺省取20个新节点，也可能小于20；
4.  池中无可用空间时，新的空间可能来自==free_list==,也可能来自==内存申请==。 <!--SR:!2022-09-03,12,184!2022-09-03,6,155!2022-09-02,11,195!2022-09-02,11,195!2022-09-14,17,195!2022-09-02,11,195-->

#### 补充:第二级空间配置器的内存池
参考链接 [STL配置器 初级篇 - 知乎](https://zhuanlan.zhihu.com/p/94402114#:~:text=%E4%B8%80%E7%BA%A7%E9%85%8D%E7%BD%AE%E5%99%A8%20%E4%B8%80%E7%BA%A7%E9%85%8D%E7%BD%AE%E7%9B%B8%E5%AF%B9%E7%AE%80%E5%8D%95%EF%BC%8C%E7%9B%B4%E6%8E%A5%E5%B0%81%E8%A3%85%E4%BA%86malloc%20%28%29%E3%80%81realloc%20%28%29,%E5%92%8C%20free%20%28%29%2C%E5%B9%B6%E5%AE%9E%E7%8E%B0%E4%BA%86%E7%B1%BB%E4%BC%BCC%2B%2B%20new-handler%E7%9A%84%E6%9C%BA%E5%88%B6%EF%BC%88%E5%86%85%E5%AD%98%E9%85%8D%E7%BD%AE%E9%9C%80%E6%B1%82%E6%97%A0%E6%B3%95%E6%BB%A1%E8%B6%B3%E6%97%B6%EF%BC%8C%E8%B0%83%E7%94%A8%E8%87%AA%E5%B7%B1%E6%8C%87%E5%AE%9A%E7%9A%84%E5%87%BD%E6%95%B0%EF%BC%89%EF%BC%8C%E6%B2%A1%E6%9C%89%E7%9B%B4%E6%8E%A5%E8%BF%90%E7%94%A8C%2B%2B%E7%9A%84%EF%BC%8C%E6%98%AF%E5%9B%A0%E4%B8%BA%E6%B2%A1%E6%9C%89%E4%BD%BF%E7%94%A8new%E6%9D%A5%E9%85%8D%E7%BD%AE%E5%86%85%E5%AD%98%E3%80%82)
`内存池`是内存配置的地方，STL内存池由下面几个变量维护
?
1.  `start_free`: 内存池的起始位置
2.  `end_free`: 内存池的结束位置
3.  `heap_size`: 保存申请总内存的大小 <!--SR:!2022-09-20,23,218-->

-   内存池是内存配置的根源，是一块地址连续的空间，free_list中的区块都来自于这里。
-   内存池中有两个比较重要的操作，一个是填充`free_list,`也就是（refill）； 另一个是配置容纳n个大小为size的一大块空间（chunk_alloc）

##### refill

-    当free_list中没有可用的区块时，refill就会登场。它主要功能是从内存池中缺省拿20个新的区块，一个返给客户端，剩余的放入free_list。
-   `refill` 为二级适配器的私有函数。下面直接看下面代码，
```c++
template <bool threas, int inst>
void* __default_alloc_template<threas, inst>::refill(size_t n) {
    //缺省取20个等大的区块
    int nobjs = 20, i;
    obj* volatile *my_free_list;
    obj *result, *current_obj, *next_obj;

    //从内存池申请nobjs个大小为n的块，第二个参数为引用传递，保存最终分配成功区块的个数
    char *chunk = chunk_alloc(n, nobjs);

    //只有一个则直接返回
    if (1 == nobjs)
        return chunk;

    //从配置的空间开始位置取n个字节，result保存了返回的块的地址
    result = (obj*)chunk;
    //设置头结点，此时free_list为空，头结点从去除返回块后的位置开始算起
    *my_free_list = next_obj = (obj*)(chunk + n);

    //各节点串起来
    for (i=1; ; i++) {
        //配置当前节点，下一节点
        current_obj = next_obj;
        next_obj = (obj*)((char*)next_obj + n);

        if (nobjs -1 == i) {
            //处理最后一个节点，下一跳指向空
            curent_obj->free_list_link = 0;
            break;
        }
        else {
            //链接到free_list 中
            current_obj->free_list_link = next_obj;
        }
    }
    return result;
}
```
##### chunk_alloc

-   从上面知道，free_list中的空间来自内存池。那内存池又是什么？内存池又怎么和free_list关联起来呢？ 答案就是 **chunk_alloc**。下面用代码说话。 <!--SR:!2022-08-11,1,218-->

```c++
/*
尝试配置nobjs个大小为size的区块，最终配置的个数保存在nobjs中

注：
    size 为8的倍数
*/
template <bool threads, int inst>
char* __default_alloc_template<threads, inst>::chunk_alloc(size_t size, in& nobjs) {
    char *resut;
    size_t total_bytes = size * objs;
    //内存池的剩余空间
    size_t bytes_left = end_free - start_free;

    if (bytes_left >= total_bytes) {
        //内存池剩余空间满足需求量，则返回空间，调整内存池大小
        result = start_free;
        start_free += total_byptes;
        return result;
    }

    if (bytes_left >= size) {
        //内存池不能满足预期需求量，但至少包含一个块，则返回最大能返回的块的空间。
        result = start_free;
        //此处先要取整，取完之后池中可能还有空间，大小肯定小于size。
        start_free += ((bytes_left / size) * size);
        return result;
    }

    if (bytes_left > 0) {
        //池中已经连一个块都无法提供，将剩余空间尝试编入更小块的free_list的头部
        //FREELIST_INDEX求指定大小空间应该在哪个free_list中。
        //由于每次申请的空间都是8的倍数，所以只有剩余空间大于0，就肯定能找到一个free_list存放最小的块，并且刚好可以存放
        obj* volatile *my_free_list = free_list + FREELIST_INDEX(bytes_left);
        ((obj*)start_free)->free_list_link = *my_free_list;
        *my_free_list = (obj*)start_free;
    }

    //此时内存池已经空了。申请2倍所需空间，RROUND_UP将指定大小调整为8的倍数
    size_t_bytes_to_get = 2 * total_bytes + ROUND_UP(heap_size >> 4);
    start_free = (char*) malloc(byptes_to_get);
    if (0 == start_free) {
        //申请失败，尝试从free_list中取块来填充内存池
        int i;
        obj* volatile *my_free_list, *p;

        //尝试从现有的free_list找可以满足的区块
        //从size开始，__ALIGN为相邻两个free_list块之间的差值，为定值
        for (i=size; i<=__MAX_BYTES; i+=__ALIGN) {
            my_free_list = free_list + FREELIST_INDEX(i);
            p = *my_free_list;
            if (0 ！= p) {
                //找到了可以使用的free_list，从里面取一个放到内存池中
                *my_free_list = p->free_list_link;
                start_free = (char*) p;
                end_free = start_free + i;
                //放到池中的块可能比需要的大，也可能可以拆分成好几个需要的块
                //池中又有水了，
                return (chunk_alloc(size, nobjs));
            }
        }
        //free_list中也不能满足需求，调用第一级配置器，尝试out-of-memory机制
        end_free = 0;
        start_free = (char*)malloc_alloc::allocate(byptes_to_get);
    }
    //内存申请成功了，放入池中，重新配置
    heap_size += bytes_to_get;
    end_free = start_free + bytes_to_get;
    return chunk_alloc(size, nobjs));
}
```

#### 3、空间配置器存在的问题
==浪费==、==静态区占用内存== <!--SR:!2022-09-03,12,178!2022-09-10,13,204-->

-   ==自由链表所挂区块都是8的整数倍==，因此当我们需要非8倍数的区块，往往会导致浪费。
    
-   ==由于配置器的所有方法，成员都是静态的，那么他们就是存放在静态区。释放时机就是程序结束，==这样子会导致自由链表一直占用内存，自己进程可以用，其他进程却用不了。 <!--SR:!2022-09-05,14,224!2022-09-03,12,204-->

### 三、各种容器的特点和适用情况

==vector、string 、array==、==deque==、==list、forwardlist==
<!--SR:!2022-09-09,12,184!2022-09-10,13,204!2022-09-10,13,204-->

vector：
?
实质：
可变大小的[[算法与数据结构 Card|数组]]
特点：
支持快速随机访问，在尾部之外的位置插入或者删除元素可能很慢 <!--SR:!2022-09-11,20,244-->

deque：
?
实质：
[[算法与数据结构 Card|双端队列]]|
特点：
支持快速随机访问
在头尾位置插入/删除速度很快 <!--SR:!2022-09-11,20,250-->

list：
?
实质：
[[算法与数据结构 Card|双向链表]]|
特点：
只支持双向顺序访问
在ist任何位置插入/删除速度很快 <!--SR:!2022-09-03,12,198-->

forward list：
?
实质：
[[算法与数据结构 Card|单向链表]]|
特点：
只支持单向顺序访问
在forward list任何位置插入/删除速度很快 <!--SR:!2022-09-01,10,206-->

array：
?
实质：
固定大小的[[算法与数据结构 Card|数组]]
特点：
支持快速随机访问
不能添加或者删除元素 <!--SR:!2022-09-07,12,246-->

string：
?
实质：
[[STL详解及常见面试题|与vectort相似的容器，专门存储字符]]|
特点：
随机访问快
在尾位置插入/删除速度很快
<!--SR:!2022-09-30,33,244-->

STL支持随机访问的容器：
==vector、array、string==、==deque==<!--SR:!2022-09-01,4,164!2022-09-05,8,183-->

STL支持在任意位置插入/删除元素：
==list、forward list== <!--SR:!2022-09-04,13,198-->

STL支持在尾部插入元素：
==vector、string==、==deque(头部他可以==) <!--SR:!2022-09-03,6,184!2022-09-06,9,184-->

### 四、各种容器的底层机制和常见面试题

#### 1、vector
##### （1）vector的底层原理
vector底层是一个动态[[算法与数据结构 Card|数组]]，包含三个[[STL详解及常见面试题|迭代器]]，start和finish之间是已经被使用的空间范围，end_of_storage是整块连续空间包括备用空间的尾部。
<center>
	<img style="border-radius: 0.3125em;
	 box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" src="https://img-blog.csdn.net/20180908193955105?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1dpemFyZHRvSA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70](https://img-blog.csdn.net/20180908193955105?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1dpemFyZHRvSA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70"> 
	 <br> 
	 <div style="color:orange; border-bottom: 1px solid #d9d9d9; 
	 display: inline-block;
	 color: #999; 
	 padding: 2px;">  vector 示意图</div> <!--SR:!2022-09-02,11,190!2022-08-30,8,174-->

</center>

当空间不够装下数据（==vec.push_back(va==l)时，会自动申请另一片更大的空间（1.5倍或者2倍），然后把原来的数据拷贝到新的内存空间，接着释放原来的那片空间。 <!--SR:!2022-09-15,20,218-->

当释放或者删除（==vec.clear()==）里面的数据时，其存储空间不释放，仅仅是清空了里面的数据。 <!--SR:!2022-09-01,7,184-->

对vector的任何操作==一旦引起了空间的重新配置==，[[STL详解及常见面试题#五、迭代器的底层机制和失效的问题|指向原vector的所有迭代器会都失效了]]。 <!--SR:!2022-09-09,18,226-->

##### （2）vector中的reserve和resize的区别
==功能、效果、参数== <!--SR:!2022-09-01,10,204-->

reserve()是==直接扩充到已经确定的大小，可以减少多次开辟、释放空间的问题（优化push_back）==，就可以==提高效率，其次还可以减少多次要拷贝数据的问题==。reserve()只有一个参数。
<!--SR:!2022-08-30,2,142!2022-09-11,13,162-->

resize()==只是保证vector中的空间大小（capacity）最少达到参数所指定的大小n==。resize()==可以改变有效空间的大小，也有改变默认值的功能==。capacity的大小也会随着改变。resize()可以有==多个==参数。 <!--SR:!2022-09-09,12,184!2022-08-31,9,197!2022-09-16,21,217-->

##### （3）vector中的size和capacity的区别
?
size表示当前==vector中有多少个元素（finish - start）==
capacity函数表示==已经分配的内存中**可以容纳多少元素**（end_of_storage - start）（finish - start）== <!--SR:!2022-09-10,19,238-->


##### （4）vector的元素类型可以是引用吗？
vector的底层实现要求连续的==对象排列==，==引用并非对象，没有实际地址，因此vector的元素类型不能是引用==
<!--SR:!2022-08-31,6,177!2022-08-31,9,192-->


##### （5）vector迭代器失效的情况
当插入一个元素到vector中，由于引起了==内存重新分配，==所以指向原内存的迭代器全部失效。
当删除容器中一个元素后,该迭代器==所指向的元素已经被删除，那么也造成迭代器失效==。erase()方法会==返回下一个有效的迭代器==，所以当我们要删除某个元素时，需要it=vec.erase(it);。
<!--SR:!2022-09-03,12,177!2022-09-08,11,182!2022-09-10,13,202-->

##### （6）正确释放vector的内存(clear(), swap(), shrink_to_fit())
?
|                             |                          |
| --------------------------- | ------------------------ |
| vec.clear()：               | 清空内容，但是不释放内存 |
| `vector<int>().swap(vec)`： |  清空内容，且释放内存，想得到一个全新的vector。                        |
| `vec.shrink_to_fit()`：     |  请求容器降低其capacity和size匹配。                        |
| vec.clear();vec.shrink_to_fit(); | 清空内容，且释放内存。                         |
<!--SR:!2022-09-03,6,174-->


##### （7）vector 扩容为什么要以1.5倍或者2倍扩容?
?
  根据查阅的资料显示，考虑可能产生的堆空间浪费，成倍增长倍数不能太大，使用较为广泛的扩容方式有两种，以2倍的方式扩容，或者以1.5倍的方式扩容。
<!--SR:!2022-09-03,12,184-->

  以2倍的方式扩容，导致下一次申请的内存必然大于之前分配内存的总和，导致之前分配的内存不能再被使用，所以最好倍增长因子设置为(1,2)之间：
##### （8）vector的常用函数
```cpp
vector<int> vec(10,100);        创建10个元素,每个元素值为100
vec.resize(r,vector<int>(c,0)); 二维数组初始化
reverse(vec.begin(),vec.end())  将元素翻转
sort(vec.begin(),vec.end());    排序，默认升序排列
vec.push_back(val);             尾部插入数字
vec.size();                     向量大小
find(vec.begin(),vec.end(),1);  查找元素
iterator = vec.erase(iterator)  删除元素

```

#### 2、list
##### （1）list的底层原理
list的底层是一个[[算法与数据结构 Card|双向链表]]|，以结点为单位存放数据，结点的地址在内存中不一定连续，每次插入或删除一个元素，就配置或释放一个元素空间。
list不支持随机存取，**如果需要大量的插入和删除**，而不关心随即存取
<center>
	<img style="border-radius: 0.3125em;
	 box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWFnZXMyMDE1LmNuYmxvZ3MuY29tL2Jsb2cvNzc5MzY4LzIwMTYxMC83NzkzNjgtMjAxNjEwMjQxNDI2NTU4NTktMTUyMjI3NzQ5Ny5wbmc?x-oss-process=image/format,png"> 
	 <br> 
	 <div style="color:orange; border-bottom: 1px solid #d9d9d9; 
	 display: inline-block;
	 color: #999; 
	 padding: 2px;">  list 示意图</div> 
</center>
##### （2）list的常用函数
```c++
list.push_back(elem)	在尾部加入一个数据
list.pop_back()	        删除尾部数据
list.push_front(elem)	在头部插入一个数据
list.pop_front()	    删除头部数据
list.size()	            返回容器中实际数据的个数
list.sort()             排序，默认由小到大 
list.unique()           移除数值相同的连续元素
list.back()             取尾部迭代器
list.erase(iterator)    删除一个元素，参数是迭代器，返回的是删除迭代器的下一个位置
```

#### 3、deque
##### （1）deque的底层原理
deque是一个双向开口的连续线性空间（**双端队列**），在头尾两端进行元素的插入跟删除操作都有理想的时间复杂度。
<center>
	<img style="border-radius: 0.3125em;
	 box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" src="https://img-blog.csdn.net/20150826111941696?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center"> 
	 <br> 
	 <div style="color:orange; border-bottom: 1px solid #d9d9d9; 
	 display: inline-block;
	 color: #999; 
	 padding: 2px;">  deque示意图</div> <!--SR:!2022-08-11,1,230-->

</center>

##### （2）什么情况下用vector，什么情况下用list，什么情况下用deque
vector可以==随机存储元素（即可以通过公式直接计算出元素地址，而不需要挨个查找），但在非尾部插入删除数据时，效率很低，适合对象简单，对象数量变化不大，随机访问频繁==。除非必要，我们尽可能选择使用vector而非deque，因为deque的迭代器比vector迭代器复杂很多。
list==不支持随机存储，适用于对象大，对象数量变化频繁，插入和删除频繁，比如写多读少的场景==。
deque==需要从首尾两端进行插入或删除操作的时候需要选择deque。==
<!--SR:!2022-08-31,6,174!2022-09-01,4,165!2022-09-01,4,165-->

##### （3）deque的常用函数
```c++
deque.push_back(elem)	在尾部加入一个数据。
deque.pop_back()	    删除尾部数据。
deque.push_front(elem)	在头部插入一个数据。
deque.pop_front()	    删除头部数据。
deque.size()	        返回容器中实际数据的个数。
deque.at(idx)	        传回索引idx所指的数据，如果idx越界，抛出out_of_range。

```
#### 4、priority_queue
##### （1）priority_queue的底层原理
 ?
 priority_queue：优先队列，其底层是用[[算法与数据结构 Card|堆]]来实现的。在优先队列中，队首元素一定是当前队列中优先级最高的那一个。
 
##### （2）priority_queue的常用函数
```c++
priority_queue<int, vector<int>, greater<int>> pq;   最小堆
priority_queue<int, vector<int>, less<int>> pq;      最大堆
pq.empty()   如果队列为空返回真
pq.pop()     删除对顶元素
pq.push(val) 加入一个元素
pq.size()    返回优先队列中拥有的元素个数
pq.top()     返回优先级最高的元素
```
#### 5、map 、set、multiset、multimap

##### （1）map 、set、multiset、multimap的底层原理
==map 、set、multiset、multimap==的底层实现都是==[[树和森林#2 1 2 5 红黑树|红黑树]]==
<!--SR:!2022-09-02,11,197!2022-09-10,17,215-->

对于STL里的map容器，==count方法与find方法，都可以用来判断一个key是否出现==，`mp.count(key) > 0`统计的是key出现的次数，因此只能为0/1，而`mp.find(key) != mp.end()`则表示key存在。
<!--SR:!2022-09-13,16,177-->

##### （2）map 、set、multiset、multimap的特点
?
①set和multiset会根据特定的排序准则自动将元素排序，set中元素不允许重复，multiset可以重复。
②map和multimap将key和value组成的pair作为元素，根据key的排序准则自动将元素排序（因为[[树和森林#2 1 2 5 红黑树|红黑树]]也是二叉搜索树，所以map默认是按key排序的，map中元素的key不允许重复，multimap可以重复。
③map和set的增删改查速度为都是$logN$，是比较高效的。
<!--SR:!2022-09-03,12,184-->

##### （3）为何map和set的插入删除效率比其他序列容器高，而且每次insert之后，以前保存的iterator不会失效？
因为==存储的是结点，不需要内存拷贝和内存移动。==
因为插入操作只是==结点指针换来换去，结点内存没有改变。==而iterator就像指向结点的指针，内存没变，指向内存的指针也不会变。
<!--SR:!2022-09-14,17,194!2022-09-21,24,217-->

##### （4）为何map和set不能像vector一样有个reserve函数来预分配数据?
?
因为在map和set内部存储的已经不是元素本身了，而是包含元素的结点。也就是说map内部使用的Alloc并不是`map<Key, Data, Compare, Alloc>`声明的时候从参数中传入的Alloc。
<!--SR:!2022-09-04,13,205-->

##### （5）map 、set、multiset、multimap的常用函数
```c++
it map.begin() 　        返回指向容器起始位置的迭代器（iterator） 
it map.end()             返回指向容器末尾位置的迭代器 
bool map.empty()         若容器为空，则返回true，否则false
it map.find(k)           寻找键值为k的元素，并用返回其地址
int map.size()           返回map中已存在元素的数量
map.insert({int,string}) 插入元素
for (itor = map.begin(); itor != map.end();)
{
    if (itor->second == "target")
        map.erase(itor++) ; // erase之后，令当前迭代器指向其后继。
    else
        ++itor;
}

```
<!--SR:!2022-08-12,1,204-->

#### 6、unordered_map、unordered_set
##### （1）unordered_map、unordered_set的底层原理
unordered_map的底层是一个防冗余的[[算法与数据结构 Card|哈希表]]（采用除留余数法）。哈希表最大的优点，就是把数据的存储和查找消耗的时间大大降低，时间复杂度为O(1)；而代价仅仅是消耗比较多的内存。

用一个下标范围比较大的数组来存储元素。可以设计一个函数（哈希函数（一般使用除留取余法），也叫做散列函数），使得每个元素的key都与一个函数值（即数组下标，hash值）相对应，于是用这个数组单元来存储这个元素；也可以简单的理解为，按照key为每一个元素“分类”，然后将这个元素存储在相应“类”所对应的地方，称为桶。

但是，不能够保证每个元素的key与函数值是一一对应的，因此极有可能出现对于不同的元素，却计算出了相同的函数值，这样就产生了“冲突”，换句话说，就是把不同的元素分在了相同的“类”之中。 一般可采用拉链法[[C++ Card#（17）解决哈希冲突的方式？ 线性探查、二次探查、双散列函数、开链法、建立公共溢出区|解决哈希冲突]]：
<center>
	<img style="border-radius: 0.3125em;
	 box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWFnZXMyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTI3Mjk3OC8yMDE4MDYvMTI3Mjk3OC0yMDE4MDYxMDE5Mjk1MzEwOS01NzQwNTg2MS5wbmc?x-oss-process=image/format,png"> 
	 <br> 
	 <div style="color:orange; border-bottom: 1px solid #d9d9d9; 
	 display: inline-block;
	 color: #999; 
	 padding: 2px;">  开链法 hash 示意图</div> <!--SR:!2022-08-11,1,230-->

</center>
##### （2）哈希表的实现
```c++
#include <iostream>
#include <vector>
#include <list>
#include <random>
#include <ctime>
using namespace std;

const int hashsize = 12;

//定一个节点的结构体
template <typename T, typename U>
struct HashNode 
{
    T _key;
    U _value;
};

//使用拉链法实现哈希表类
template <typename T, typename U>
class HashTable
{
public:
    HashTable() : vec(hashsize) {}//类中的容器需要通过构造函数来指定大小
    ~HashTable() {}
    bool insert_data(const T &key, const U &value);
    int hash(const T &key);
    bool hash_find(const T &key);
private:
    vector<list<HashNode<T, U>>> vec;//将节点存储到容器中
};

//哈希函数（除留取余）
template <typename T, typename U>
int HashTable<T, U>::hash(const T &key)
{
    return key % 13;
}

//哈希查找
template <typename T, typename U>
bool HashTable<T, U>::hash_find(const T &key)
{
    int index = hash(key);//计算哈希值
    for (auto it = vec[index].begin(); it != vec[index].end(); ++it)
    {
        if (key == it -> _key)//如果找到则打印其关联值
        {
            cout << it->_value << endl;//输出数据前应该确认是否包含相应类型
            return true;
        }
    }
    return false;
}

//插入数据
template <typename T, typename U>
bool HashTable<T, U>::insert_data(const T &key, const U &value)
{
    //初始化数据
    HashNode<T, U> node;
    node._key = key;
    node._value = value;
    for (int i = 0; i < hashsize; ++i)
    {
        if (i == hash(key))//如果溢出则把相应的键值添加进链表
        {
            vec[i].push_back(node);
            return true;
        }
    }
}

int main(int argc, char const *argv[])
{
    HashTable<int, int> ht;
    static default_random_engine e;
    static uniform_int_distribution<unsigned> u(0, 100);
    long long int a = 10000000;
    for (long long int i = 0; i < a; ++i)
        ht.insert_data(i, u(e));
    clock_t start_time = clock();
    ht.hash_find(114);
    clock_t end_time = clock();
    cout << "Running time is: " << static_cast<double>(end_time - start_time) / CLOCKS_PER_SEC * 1000 <<
        "ms" << endl;//输出运行时间。
    system("pause");
    system("pause");
    return 0;
}

```

##### （3）unordered_map 与map的区别？使用场景？
区别：
构造函数：==unordered_map 需要hash函数，等于函数;map只需要比较函数(小于函数).==
存储结构：==unordered_map 采用hash表存储，map一般采用==[[树和森林#2 1 2 5 红黑树|红黑树]]实现。因此其memory数据结构是不一样的。==
<!--SR:!2022-09-02,11,174!2022-09-09,12,204-->

总体来说，==unordered_map 查找速度会比map快，而且查找速度基本和数据数据量大小无关，属于常数级别$O(1$);而map的查找速度是$log(n$)级别。==并不一定常数就比$log(n)$小，hash还有hash函数的消耗
如果考虑效率，==特别是在元素达到一定数量级时，使用unordered_map== 。
对内存使用特别严格，希望程序尽可能少消耗内存，==unordered_map 可能不适用，==特别是unordered_map 对象特别多时，更难控制，而且unordered_map 的构造速度较慢。
<!--SR:!2022-09-11,14,150!2022-08-31,9,175!2022-09-09,12,195-->

##### （4）unordered_map、unordered_set的常用函数
?
```c++
unordered_map.begin() 　　  返回指向容器起始位置的迭代器（iterator） 
unordered_map.end() 　　    返回指向容器末尾位置的迭代器 
unordered_map.cbegin()　    返回指向容器起始位置的常迭代器（const_iterator） 
unordered_map.cend() 　　   返回指向容器末尾位置的常迭代器 
unordered_map.size()  　　  返回有效元素个数 
unordered_map.insert(key)  插入元素 
unordered_map.find(key) 　 查找元素，返回迭代器
unordered_map.count(key) 　返回匹配给定主键的元素的个数 

```
<!--SR:!2022-09-02,5,155-->

### 五、迭代器的底层机制和失效的问题

#### 1、迭代器的底层原理
底层实现包含两个重要的部分：==萃取技术==和==模板偏特化==。
<!--SR:!2022-09-01,4,170!2022-09-11,13,203-->

#TODO
萃取技术（traits）可以进行==类型推导，==根据不同类型可以执行不同的处理流程，比如容器是vector，那么traits必须推导出其迭代器类型为随机访问迭代器，而list则为双向迭代器。
<!--SR:!2022-08-31,2,190-->

  例如STL算法库中的distance函数，distance函数接受两个迭代器参数，然后计算他们两者之间的距离。显然对于不同的迭代器计算效率差别很大。比如对于vector容器来说，由于内存是连续分配的，因此指针直接相减即可获得两者的距离；而list容器是链式表，内存一般都不是连续分配，因此只能通过一级一级调用next()或其他函数，每调用一次再判断迭代器是否相等来计算距离。vector迭代器计算distance的效率为O(1),而list则为O(n),n为距离的大小。

使用萃取技术（traits）进行类型推导的过程中会使用到模板偏特化。模板偏特化可以用来推导参数，如果我们自定义了多个类型，除非我们把这些自定义类型的特化版本写出来，否则我们只能判断他们是内置类型，并不能判断他们具体属于是个类型。

#### 2、一个理解traits的例子

#### 3、迭代器的种类
==输入迭代器、输出迭代器==、==前向迭代器==、==双向迭代器==、==随机访问迭代器==
<!--SR:!2022-09-01,4,150!2022-09-10,13,204!2022-09-09,12,204!2022-09-09,12,204-->

输入迭代器：::是只读迭代器，在每个被遍历的位置上只能读取一次。例如上面find函数参数就是输入迭代器。
<!--SR:!2022-09-17,22,224-->

输出迭代器：::是只写迭代器，在每个被遍历的位置上只能被写一次。
<!--SR:!2022-09-24,27,224-->

#TODO operator– 是啥玩意
前向迭代器：::兼具输入和输出迭代器的能力，但是它可以对同一个位置重复进行读和写。但它不支持[[|operator–]]，所以只能向前移动。
<!--SR:!2022-09-09,12,184-->

双向迭代器：::很像前向迭代器，只是它向后移动和向前移动同样容易。
<!--SR:!2022-09-03,12,184-->

随机访问迭代器：::有双向迭代器的所有功能。而且，它还提供了“迭代器算术”，即在一步内可以向前或向后跳跃任意位置， 包含指针的所有操作，可进行随机访问，随意移动指定的步数。支持前面四种Iterator的所有操作，并另外支持it + n、it - n、it += n、 it -= n、it1 - it2和it[n]等操作。
<!--SR:!2022-10-04,37,244-->
 
#### 4、迭代器失效的问题
##### （1）插入操作
两个一组
基于动态数组对于==vector和string==、==基于队列deque==、==基于链表list、forward_list==
<!--SR:!2022-09-01,7,184!2022-09-10,13,204!2022-09-06,9,184-->

对于vector和string，
如果容器内存被重新分配，==iterators,pointers,references失效==；
如果没有重新分配，==那么插入点之前的iterator有效，插入点之后的iterator失效==；
<!--SR:!2022-09-07,9,177!2022-09-13,15,182-->

对于deque
如果插入点位==于除front和back的其它位置==，==iterators,pointers,references失效==；
当插入元素到==front和back==时，deque==的迭代器失效，但[[|reference引用]]和pointers有效==；
<!--SR:!2022-09-02,11,177!2022-09-02,5,157!2022-09-04,13,202!2022-09-01,4,162-->

对于list和forward_list，==所有的iterator,pointer和refercnce有效==。
<!--SR:!2022-09-04,7,197-->

##### （2）删除操作
基于==动态数组对于vector和string==、==基于队列deque==、==基于链表list、forward_list==,，==关于关联容器map==
<!--SR:!2022-09-03,12,177!2022-09-10,13,204!2022-09-05,8,184!2022-09-10,13,204-->

对于vector和string
==删除点==之前的==iterators,pointers,references==有效；
off-the-end迭代器==总是失效的==；
<!--SR:!2022-08-30,5,177!2022-09-18,20,202!2022-09-10,13,202-->

对于deque，
如果删除点位于==除front和back==的其它位置，==iterators,pointers,references==失效
当插入元素到==front和back==时，==off-the-end失效，其他的iterators,pointers,references有效==
<!--SR:!2022-08-30,8,157!2022-09-08,11,182!2022-09-10,13,202!2022-08-30,2,142-->

对于list和forward_list，
所有的==iterator,pointer和refercnce==有效。
<!--SR:!2022-09-04,7,197-->

对于关联容器map来说，
如果某一个元素已经被删除，那么其对应的==迭代器==就失效了，不应该再被使用，否则会导致程序无定义的行为。
<!--SR:!2022-09-02,11,174-->

### 六、STL容器的线程安全性

#### 1、线程安全的情况
-   ==**多个读取者是安全的**==。多线程可能同时读取一个容器的内容，这将正确地执行。当然，在读取时不能 有任何写入者操作这个容器；
-   ==**对不同容器的多个写入者是安全的**==。多线程可以同时写不同的容器。
<!--SR:!2022-09-11,13,164!2022-08-30,4,183-->

#### 2、线程不安全的情况
-   **在对同一个容器进行多线程的读写、写操作时**；

**如何避免迭代器不安全**
在==每次调用容器的成员函数期间==都要锁定该容器；   
在==每个容器返回的迭代器（例如通过调用begin或end）的生存期之内==都要锁定该容器；    
在每个在容器上==调用的算法执行期间==锁定该容器。
<!--SR:!2022-08-30,2,130!2022-08-30,2,135!2022-08-30,2,135-->



