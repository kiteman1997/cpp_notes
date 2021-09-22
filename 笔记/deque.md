# deque

## 要点

``` shel
1.有一个中控区和若干缓存区，中控区中每个元素都是指针(T*)，指向一片连续空间；缓冲区每个元素都是T*
2.deque主体中只包含一个map_pointer(T**)，指向一片空间，空间里都是指向缓冲区头部元素的指针，即都是T*
3.缓存区是分段连续的，但是迭代器对外实现了看起来连续，因此支持随机访问
4.迭代器的设计中有四个指针，三个是T*，一个是T**(也叫map_pointer，和主体中一样)
```

<img src="https://pic1.zhimg.com/80/v2-45154a314f51cf7519ff1c31bc13d044_1440w.jpg" alt="img" style="zoom: 67%;" />

## 迭代器

```shell
1.由于deque主体只有一个map_pointer，因此大部分功能都得看迭代器
2.迭代器中有三个T*指针，分别是cur当前位置，first缓冲区'数组头'，last'数组尾'
3.一个T**指针，指向当前缓冲区在map中的位置
```

```c++
struct __deque_iterator {
 ...
  typedef T value_type;
  T* cur;
  T* first;
  T* last;
  typedef T** map_pointer;
  map_pointer node;
  ...
};
```

<img src="https://pic4.zhimg.com/80/v2-fab8a14b71f13e3510e3df728555dbc7_1440w.jpg" alt="img" style="zoom:67%;" />

### 缓冲区大小

```shell
buffer_size参数是deque模板类的第三个参数，填0或不填都是预设值，即512除以元素类型的大小，不足则为1
```

```c++
inline size_t __deque_buf_size(size_t n, size_t sz) {
  return n != 0 ? n : (sz < 512 ? size_t(512 / sz): size_t(1));
}
//如果 n 不为0，则返回 n，表示缓冲区大小由用户自定义
//如果 n == 0，表示 缓冲区大小默认值
//如果 sz = (元素大小 sizeof(value_type)) 小于 512 则返回 521/sz
//如果 sz 不小于 512 则返回 1
```

### 迭代器的操作

```shell
++：需要判断是否到达缓冲区的尾部，即'加完'等于last的话就转到下一个node的first，注意last是屁股后面一个位置
--：需要判断是否到达缓冲区的头部，即'当前'等于first的话就到上一个node的last'再--'
-：返回两个位置的距离即相差多少个元素，等于buffer_size*相差的node个数 + 位置A与first之差 + 位置B与last之差；分开计算是因为没法直接算它们之间的位置差(可能不在同一个buffer)，也正是重载-的意义所在
```

```c++
self& operator++() { 
  ++cur;      //切换至下一个元素
  if (cur == last) {   //如果已经到达所在缓冲区的末尾
 set_node(node+1);  //切换下一个节点
 cur = first;  
  }
  return *this;
}
```

```c++
self& operator--() {     
  if (cur == first) {    //如果已经到达所在缓冲区的头部
 set_node(node - 1); //切换前一个节点的最后一个元素
 cur = last;  
  }
  --cur;       //切换前一个元素
  return *this;
}
```

```c++
operator- (const self& x) const {
    return difference_type(buffer_size()) * (node - x.node - 1) + 
        (cur - first) + (x.last - x.cur);
}
```

## 构造和析构

```c++
1.默认构造——deque()
2.拷贝构造——deque(const deque& x)
3.初始化大小和值——deque(size_type n, const value_type& value)
4.初始化大小——explicit deque(size_type n)
5.接收两个迭代器构造范围——deque(InputIterator first, InputIterator last)
```

### create_map_and_nodes函数

构造函数中会调用该函数或者fill_initialize()或者range_initialize()；后两个用于填充同一个值以及传入其他类型迭代器的情况

```shell
1.计算node的个数num_nodes，nums/buffer_size() + 1
2.map_size为初始size和num_nodes + 2取大的
3.初始化两个map_pointer:nstart, nfinish
4.为每个nstart和nfinish之间的map_pointer分配buffer_size的数组
5.修改start, finish以及分别指向的数组的cur指针的位置, 说通俗就是start和finish分别指向第一个和最后一个元素的位置
```

```c++
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::create_map_and_nodes(size_type num_elements) 
{
    // 计算初始化类型参数的个数
  size_type num_nodes = num_elements / buffer_size() + 1;
	// 因为deque是头尾插入都是O(1), 就是deque在头和尾都留有空间方便头尾插入
    // 两者取最大的. num_nodes是保证前后都留有位置
  map_size = max(initial_map_size(), num_nodes + 2);
  map = map_allocator::allocate(map_size);	// 分配空间

    // 计算出数组的头前面留出来的位置保存并在nstart.
  map_pointer nstart = map + (map_size - num_nodes) / 2;
  map_pointer nfinish = nstart + num_nodes - 1;
    
  map_pointer cur;
  __STL_TRY 
  {
      // 为每一个a[cur]分配一个buffer_size的数组, 即这样就实现了二维数组即map
    for (cur = nstart; cur <= nfinish; ++cur)
      *cur = allocate_node();
  }
	// 修改start, finish, cur指针的位置
  start.set_node(nstart);
  finish.set_node(nfinish);
  start.cur = start.first;
  finish.cur = finish.first + num_elements % buffer_size();
}
```

### range_initialize函数

```shell
1.如果输入的迭代器是InputIterator，则遍历一个一个push
2.如果输入的是ForwardIterator，则调用uninitialized_copy范围复制
```

```c++
template <class T, class Alloc, size_t BufSize>
template <class InputIterator>
void deque<T, Alloc, BufSize>::range_initialize(InputIterator first,
                                                InputIterator last,
                                                input_iterator_tag) {
  create_map_and_nodes(0);
    // 一个个进行插入操作
  for ( ; first != last; ++first)
    push_back(*first);
}

template <class T, class Alloc, size_t BufSize>
template <class ForwardIterator>
void deque<T, Alloc, BufSize>::range_initialize(ForwardIterator first,
                                                ForwardIterator last,
                                                forward_iterator_tag) {
  size_type n = 0;
    // 计算距离, 申请空间. 失败则释放所有空间
  distance(first, last, n);
  create_map_and_nodes(n);
  __STL_TRY {
    uninitialized_copy(first, last, start);
  }
  __STL_UNWIND(destroy_map_and_nodes());
}
```

### 析构函数

```shell
1.遍历每个map_pointer，调用deallocate_node析构node
2.map_allocator::deallocate释放内存
```

```c++
// This is only used as a cleanup function in catch clauses.
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::destroy_map_and_nodes() {
// 便利所有的数组, 一个个析构
  for (map_pointer cur = start.node; cur <= finish.node; ++cur)
    deallocate_node(*cur);
    // 内存释放
  map_allocator::deallocate(map, map_size);
}
```

## 基本属性获取

```c++
1.size_type buffer_size() // 获取缓冲区大小
2.iterator start;	// 指向第一个元素的地址
3.iterator finish;	// 指向最后一个元素的后一个地址, 即尾
4.map_pointer map;		// 定义map, 指向指针的指针
5.size_type map_size;	// map的实际大小
6.iterator begin() { return start; }	// 获取头地址
7.iterator end() { return finish; }		// 获取尾地址
8.reference front() { return *start; }  // 获取第一个和最后一个元素的值
9.size_type size() const { return finish - start;; } 	// 获取数组的大小
10.reference operator[](size_type n) { return start[difference_type(n)]; }	//下标访问
```

## 一些操作

### push，pop

```c++
//push_back(const value_type& t)
1.判断函数是否达到了数组尾部(注意这里是判断是否等于finish.last-1). 没有达到就直接进行插入
2.如果到达尾部则调用push_back_aux(t)——先申请一个buffer:*(finish.node + 1)，然后构造node，finish移动到下一个位置(没办法++,得移动self了，所以调用finish.set_node)，然后finish.cur指向下一个数组的first
//push_front(const value_type& t)    
1.判断是否到了头(注意这里是判断start.cur != start.first)。没有到达则直接插入
2.如果到达头部则和尾部类似，但是需要注意这里是'先移动node再构造'，原因是finish指向最后一个元素的后面，因此'其实上面的push_back还有一个位置'，可以在那个位置放好再移动到新的空间；但是start就是指向第一个元素，所以没有位置去构造了，只能'移动到新空间的last-1位置再构造'    
//pop_back(), pop_front()
1.判断的方式和push一样，不过是pop_back()要判断是不是finish.first；pop_front()判断start.last - 1
2.析构的地方需要注意：两个都是先析构再移动node，但是pop_back是在'移动完以后释放新的finish.cur'；pop_front是在'移动前释放start.cur'
```

### erase(pos)

```c++
1.删除的位置是中间偏前，将pos前面的元素后移一个copy_backward(start, pos, next)，然后pop_front()
2.中间偏后，则后面的元素前移一个copy(next, finish, pos),然后pop_back()
```

### erase(iterator first, iterator last)

```shell
1.计算两个迭代器的距离
2.同样的copy_backward，但是后面是destroy(start, new_start)
3.destroy完了以后还有一步是deallocate每一个中间的map_pointer，因为有可能跨越buffer了，单个元素显然不会，所以只有在范围erase时才会考虑
```

### insert

```c++
//几种重载
1.iterator insert(iterator position, const value_type& x);
2.iterator insert(iterator position) ;
3.void insert(iterator pos, size_type n, const value_type& x);
4.void insert(iterator pos, InputIterator first, InputIterator last);
```

```c++
//基本流程
1.判断是否是头部或尾部，都可以直接push；如果是批量插入则是uninitialized_fill或uninitialized_copy
2.都不是则调用insert_aux
```

### insert_aux

```shell
//插入单个元素
1.判断插入位置，离头近还是离尾近
2.头部往前移动(即push_front)
3.中间元素copy过去，这里和vector一样的
//插入多个元素
1.同样判断头尾
2.判断插入点到start的距离和插入个数n哪个大
3.分段移动的逻辑和vector一样，前挪k-n个位置，然后挪出来的位置都fill成n
4.如果小于n则
```

