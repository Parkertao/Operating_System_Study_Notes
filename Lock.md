# 锁

## 锁的基本思想

### 锁就是一个变量，保存锁在某一时刻的状态，要么是可用的，要么是被占用的

### POSIX将锁称为互斥量mutex

```c++
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_lock(&lock);
...
pthread_mutex_unlockj(&lock);
```

- 允许使用不同锁来保护不同数据结构

### 锁的实现

- 需要硬件和操作系统的配合来实现一个可用的锁

### 锁的评价

- 提供互斥性
- 公平性
- 性能

## 锁的实现

### 控制中断

- 优点：简单
- 缺点：一个贪婪的程序可能恶意独占cpu；不支持多处理器；关闭中断导致中断丢失，可能引发严重的系统问题

### 测试并设置（原子交换）

```c++
int TestAndSet(int *old_ptr, int new)
{
    int old = *old_ptr;
    *old_ptr = new;
    return old;
}
```

### 实现自旋锁

```c++
void init(lock_t *lock)
{
	lock->flag = 0;
}

void lock(lock_t *lock)
{
    // 自旋
	while (TestAndSet(&lock->flag, 1) == 1);
}

void unlock(lock_t* lock)
{
	lock->flag = 0;
}
```

- 自旋锁在单cpu上无法使用，因为一个自旋的线程永远不会放弃cpu
- 评价自旋锁

  - 提供互斥性
  - 不提高公平性保证
  - 性能开销很大

### 比较并交换

```c++
int CompareAndSwap(int *ptr, int expected, ine new)
{
    int actual = *ptr;
    if (actual == expected) *ptr = new;
    return actual;
}
```

- 实现自旋锁

```c++
void lock(lock_t* lock)
{
	while (CompareAndSwap(&lock->flag, 0, 1) == 1);
}
```

### 连接加载+条件式储存

- 条件是存储指令，只有上一次加载的地址在期间没有更新时才会成功（同时更新刚才链接加载的地址的值）

### 获取并增加

- ```c++
  int FetchAndAdd(int *ptr)
  {
  int old = *ptr;
  *ptr = old + 1;
  return old;
  }
  ```

- ```c++
  typedef struct lock_t {
  	int ticket;
  	int turn;
  } lock_t;
  
  void lock_init(lock_t *lock) {
      lock->ticket = 0;
      lock->turn = 0;
  }
  
  void lock(lock_t *lock) {
      int myturn = FetchAndAdd(&lock->ticket);
      while (lock->turn != myturn); // spin
  }
  
  void unlock(lock_t *lock) {
  	FetchAndAdd(&lock->turn);
  }
  ```

- 提供公平性保障

### 自旋过多的解决方案

- 在要自旋时放弃cpu

	- 上下文切换的成本仍然很高
	- 不能保证公平性，可能会有线程饿死

- 使用队列：休眠代替自旋

	- 必须显示地施加某种控制，决定锁释放时，谁能抢到锁
	- Solaris提供的支持

		- typedef struct

### 两阶段锁

- 两阶段锁意识到自旋可能很有用，尤其是在很快就要释放锁的场景
- 如果第一个自旋阶段没有获得锁，第二阶段调用者会睡眠，直到锁可用

## 基于锁的并发数据结构

### 并发计数器

- 给普通计数器加锁，得到简单的有锁计数器

```c++
typedef struct counter_t {
    int value;
    pthread_mutex_t lock;
} counter_t;

void init(counter_t *c) {
    c->value = 0;
    Pthread_mutex_init(&c->lock, NULL);
}

void increment(counter_t *c) {
    Pthread_mutex_lock(&c->lock);
    c->value++;
    Pthread_mutex_unlock(&c->lock);
}

void decrement(counter_t *c) {
    Pthread_mutex_lock(&c->lock);
    c->value--;
    Pthread_mutex_unlock(&c->lock);
}

int get(counter_t *c) {
    Pthread_mutex_lock(&c->lock);
    int rc = c->value;
    Pthread_mutex_unlock(&c->lock);
    return rc;
}
```

- 可扩展计数器

  - 理想状态下，多处理器上运行多线程就想单线程一样快，这种状态称为完美扩展
  - 懒惰计数器

  	- 每个cpu核心有一个局部计数器和局部锁，同时还有一个全局计数器和全局锁
  	- 如果一个核心上的线程想要增加计数器，增加它的局部计数器，通过局部锁保证同步，不同cpu上的线程不会竞争
  	- 为了保持全局计数器更新，局部值会定期转移给全局计数器，方法是获取全局锁，让全局计数器加上局部计数器的值，然后局部计数器置零
  	- 局部转全局的频度有阈值S决定，S越小懒惰计数器越精确，同时越接近普通非扩展计数器，S越大扩展性越强，但是全局计数器和实际计数的偏差越大。需要在性能和准确性之间折衷

### 并发链表

- 调整加锁位置，使之只环绕代码的真正临界区

```c++
void List_Init(list_t *L) {
    L->head = NULL;
    pthread_mutex_init(&L->lock, NULL);
}

void List_Insert(list_t *L, int key) {
    // synchronization not needed
    node_t *new = malloc(sizeof(node_t));
    if (new == NULL) {
        perror("malloc");
        return;
    }
    new->key = key;

    // just lock critical section
    pthread_mutex_lock(&L->lock);
    new->next = L->head;
    L->head = new;
    pthread_mutex_unlock(&L->lock);
}

int List_Lookup(list_t *L, int key) {
    int rv = -1;
    pthread_mutex_lock(&L->lock);
    node_t *curr = L->head;
    while (curr) {
        if (curr->key == key) {
            rv = 0;
            break;
        }
        curr = curr->next;
    }
    pthread_mutex_unlock(&L->lock);
    return rv; // now both success and failure
}
```

### 并发队列

- 使用两个锁，一个负责队列头，一个负责队列尾，使得入队操作和出队操作可以并发执行

```c++
typedef struct node_t {
	int value;
	struct node_t *next;
} node_t;

typedef struct queue_t {
    node_t *head;
    node_t *tail;
    pthread_mutex_t headLock;
    pthread_mutex_t tailLock;
} queue_t;

void Queue_Init(queue_t *q) {
    node_t *tmp = malloc(sizeof(node_t));
    tmp->next = NULL;
    q->head = q->tail = tmp;
    pthread_mutex_init(&q->headLock, NULL);
    pthread_mutex_init(&q->tailLock, NULL);
}

void Queue_Enqueue(queue_t *q, int value) {
    node_t *tmp = malloc(sizeof(node_t));
    assert(tmp != NULL);
    tmp->value = value;
    tmp->next = NULL;

    pthread_mutex_lock(&q->tailLock);
    q->tail->next = tmp;
    q->tail = tmp;
    pthread_mutex_unlock(&q->tailLock);
}

int Queue_Dequeue(queue_t *q, int *value) {
    pthread_mutex_lock(&q->headLock);
    node_t *tmp = q->head;
    node_t *newHead = tmp->next;
    if (newHead == NULL) {
        pthread_mutex_unlock(&q->headLock);
        return -1; // queue was empty
    }
    *value = newHead->value;
    q->head = newHead;
    pthread_mutex_unlock(&q->headLock);
    free(tmp);
    return 0;
}
```

### 并发散列表

- 使用并发链表，每个散列桶都有一个锁，支持许多并发操作

```c++
#define BUCKETS (101)

typedef struct hash_t {
	list_t lists[BUCKETS];
} hash_t;

void Hash_Init(hash_t *H) {
    int i;
    for (i = 0; i < BUCKETS; i++) {
    	List_Init(&H->lists[i]);
    }
}

int Hash_Insert(hash_t *H, int key) {
	int bucket = key % BUCKETS;
	return List_Insert(&H->lists[bucket], key);
}

int Hash_Lookup(hash_t *H, int key) {
	int bucket = key % BUCKETS;
	return List_Lookup(&H->lists[bucket], key);
}
```



