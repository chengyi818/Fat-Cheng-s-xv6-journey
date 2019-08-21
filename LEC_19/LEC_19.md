# 内核伸缩性

# 动机
1. 现代CPU大多数都是多核的.
2. 应用程序严重依赖内核的网络,文件系统.
3. 如果内核在多核场景性能不佳,那么应用程序的性能同样受到限制.
4. 因此,内核必须支持并发执行系统调用.

# 数据共享
1. 内核本身会创建很多公共的数据结构.比如进程表,文件描述符表,buffer缓存,调度列表等.
2. 依赖锁保证其一致性.
3. 应用程序可能会抢锁导致竞争,从而降低性能.

# OS的演进
1. 早期内核使用**大内核锁**来保护内核数据.
2. 不久,内核就推出了更细粒度的锁
3. 现在,许多`lock-free`的方法也在大量使用.
4. 极端情况下,一些实验性质的内核正在尝试不共享任何数据.比如FOS和Barrelfish.
5. 优点在于,不存在缓存一致性问题.
6. 缺点在于,负载均衡的性能很差.

# 大纲
1. Read-copy-update
2. 每个CPU 引用计数
3. 核间通信规则

---

# 大量读操作的数据结构
内核经常面临这样的情况,读的操作远远多于写.比如网络相关的路由表,ARP.比如文件描述符数组,大多数的系统调用状态.RCU(Read-copy-update)有助于提升这些场景的性能.目前内核中大约有10,000处使用了RCU的API.

目标:
1. 在写的同时,支持并发读取.
2. 较小的空间消耗
3. 较小的执行时长

---

## 方案一: 自旋锁
这种方案的主要问题在于,所有临界区中的执行都必须按顺序.这样会导致读进程必须等待其他读进程完成.

改进的方案在于,是否可以允许多个进程同时读取.当写入时,再抢锁写入.因此引入了第二种方案.

---

## 方案二: 读写锁
读写锁允许并发读取,能够提升性能.

### 读写锁实现

```
typedef struct { volatile int cnt; } rwlock_t;

void read_lock(rwlock_t *l) {
  int x;
  while (true) {
    x = l->cnt;
    if (x < 0) // is write lock held?
      continue;

    if (CMPXCHG(&l->cnt, x, x + 1))
      break;
  }
}

void read_unlock(rwlock_t *l) {
  ATOMIC_DEC(&l->cnt);
}

typedef struct { volatile int cnt; } rwlock_t;

void write_lock(rwlock_t *l) {
  int x;
  while (true) {
    x = l->cnt;
    if (x != 0) // is the lock held?
      continue;
    if (CMPXCHG(&l->cnt, 0, -1))
      break;
  }
}

void write_unlock(rwlock_t *l) {
  ATOMIC_INC(&l->cnt);
}
```

### 读写锁性能分析
每个读进程都会调用`CMPXCHG`指令.该指令会将`Shared`状态的缓存改为`Modified`.因此`read_lock()`将会查找并使得`l->cnt`变量失效.`read_unlock()`同样如此.另外,如果写进程占有锁,那么所有读进程序必须自旋等待.

进一步的优化目标在于,在写数据的同时,支持并发读取.

---

## 方案三: Read-copy-update(RCU)
RCU的实现原理为:
1. 读进程读取对象不需要锁.
2. 写进程首先拷贝对象,然后修改新对象.
3. 最终将对象指针指向新对象.

示意图如下:
![rcu](./images/rcu.bmp)

### 释放旧对象
1. 在一个特定的时间点,读进程要么可以访问新对象,要么可以访问旧对象.
2. idea: 是否可以在对象不再可达时,释放对象?
3. 通常只有一个指针指向RCU对象,RCU对象不能被拷贝保存到栈或寄存器上
4. 需要定义一个*切换时间*,在此时间点之后,对象可以安全释放.
  4.1 等待所有core都进行至少一次上下文切换.
  4.2 指针引用对象只能在临界区内修改
  4.3 读进程在临界区内,需要禁止抢占.

Q: 为什么RCU读进程在临界区内需要禁止抢占?
1. 如果不禁止抢占,等待所有core都进行至少一次上下文切换就不是一个有效的*切换时间*.
2. 一个进程在被抢占后,仍然可能持有指向RCU对象的指针.
  2.1 很难确定安全的指针释放时间
  2.2 除非等到所有当前运行的进程全部结束
3. 需要定义一个读临界区,所有对RCU对象的引用不能在临界区之外.

### 简化的RCU API
```
void rcu_read_lock() {
  preempt_disable[cpu_id()]++;
}
void rcu_read_unlock() {
  preempt_disable[cpu_id()]--;
}
void synchronize_rcu(void) {
  for_each_cpu(int cpu)
    run_on(cpu);
}
```

### 真实的RCU API
1. rcu_read_lock(): 开始RCU临界区
2. rcu_read_unlock(): 结束RCU临界区
3. synchronize_rcu(): 等待所有的RCU临界区执行完成
4. call_rcu(callback, argument): 当所有RCU临界区执行完成,执行回调函数
5. rcu_dereference(pointer): 在读临界区中,获取pointer的指针引用
6. rcu_dereference_protected(pointer,check): 在临界区外获取pointer的指针引用
7. rcu_assign_pointer(pointer_addr,	pointer): 在临界区内修改pointer指向的对象

### 如何同步写?
#### 对于写进程:
1. 只允许一个写进程
2. 或者使用常规的同步机制,比如锁

#### 对于读进程(内存顺序):
1. 在更新指针前,写进程必须全部完成对象修改.
2. 读进程不能重排读取指令,以防止指针内容在指针前被读取.
3. `rcu_dereference()`和`rcu_assign_pointer()`会自动插入编译和内存屏障

### RCU使用示例
假设存在一个简单的线上书店,需要一个对象来代表每件商品的价格.结构体定义如下:
```
typedef struct {
  const char *name;
  float price;
  float discount;
} item_t;
__rcu item_t *item;
lock_t item_lock;
```
注意: 最终价格为`price-discount`

#### 读示例
```
float get_cost(void) {
  item_t *p;
  float cost;
  rcu_read_lock();
  p = rcu_dereference(item); // read
  cost = p->price – p->discount;
  rcu_read_unlock();
  return cost;
}
```

#### 写示例
```
void set_cost(float price, float discount) {
  item_t *oldp, *newp;
  spin_lock(&item_lock);
  oldp = rcu_dereference_protected(item, spin_locked(&item_lock));
  newp = kmalloc(sizeof(*newp));
  *newp = *oldp; // copy
  newp->price = price;
  newp->discount = discount;
  rcu_assign_pointer(item, newp); // update
  spin_unlock(&item_lock);
  rcu_synchronize();
  kfree(oldp); // free
}
``**

### RCU是非常强大的工具
1. 可以兼容很多复杂的数据结构,比如链表,hash表.
2. 大部分使用场景和读写锁类似.
3. 可以用于等待并发线程同步.
4. 可以用于省略引用计数.

### RCU是否完成设计目标?
* 目标一: 在更新过程中,并发读取
完成,读进程不会因为更新而停止.

* 目标二: 空间消耗少
完成,一个RCU指针和常规指针大小相同哦,且没有其他同步数据.当然旧对象的释放必须等到*切换时间*之后,这会导致一定的时间消耗.

* 目标三: 执行时间少
对于读进程,RCU几乎没有执行消耗.对于写进程,因为RCU必须分配,释放,拷贝,因此会导致少量的时间消耗.实践中这样的消耗是适中的.细粒度的锁有助于提供更新的并发性.

## 引用计数
1. 记录指向对象的指针数量.
2. 当指针数量降为0,可以安全释放对象.
3. 挑战: 会有真正的内存共享,在内核中很多资源都有引用基数,常常会导致性能瓶颈.

### 标准方法

```
typedef struct { int cnt; } kref_t;

void kref_get(kref_t *r) {
  WARN_ON(r->cnt == 0);
  ATOMIC_INC(&r->cnt);
}

void kref_put(kref_t *r,

void (*release)(kref_t *r)) {
  if (ATOMIC_DEC_AND_TEST(&r->cnt))
    release(r);
}
```

### 执行消耗
1. `kref_get()`和`kref_put()`都要求排他性的占据缓存,会修改`r->cnt`.
2. 如果对象引用频繁,那么会造成大量的缓存抖动.

### 缓解措施: 每CPU引用计数
1. 为每个CPU核维护一个引用计数数组.
2. `percpu_ref_get()`和`percpu_ref_put()`仅操作本地CPU核的数组,不会引起缓存抖动.
3. `percpu_ref_kill()`会将引用计数转为正常的共享计数.
4. 当引用计数频繁在一个进程中修改时,会有显著的性能提升.而根据局部性原理,通常都是如此.

### 如何实现`percpu_ref_kill(**`?
1. 将共享计数设置为一个基础差值.
2. 在共享计数中设置一个`kill`flag.
3. 等待一个*切换时间*,以便所有的引用计数器都能看到flag.(使用rcu_synchronize())
4. 统计所有每CPU中的引用计数,并将总和更新到共享计数.
5. 最终,减去基础差值.
6. 当共享引用计数为0,释放对象.

## 代码伸缩性
我们看了两个高伸缩性的算法,但是都有适用场景.在更一般的情况下,如何使得代码更有伸缩性呢?

### 规则
1. 如果两个操作通信,那么存在一个无竞争的实现.
2. 根据直觉,如果操作间通信,顺序完全无所谓.那么操作间的通信就没有必要.
3. 如果操作违反了这条规则,那么伸缩性的实现仍然是可能的.但是如果遵守了这条规则,则一定可以实现扩展性.

### 扩展性的关键是良好的接口设计
1. `POSIX`接口要求`open()`返回最小的可用fd.
2. 那么两个`open()`是否会通信呢?
3. 该如何修改`open()`保证更好的扩展性?

---

# 结论
1. RCU提供了零消耗的只读访问,写访问消耗稍高一点.非常适合大量读取的场景.
2. 引用计数在对象长时间存在的场景下,几乎是零消耗的.当然我们额外需要设计一个线程作为`killer`.每CPU引用计数会增加额外的空间消耗.
3. 扩展性规则提供了如何设计扩展性良好的接口规范.
