---
title: BufferPoolManager
date: '2024-02-05'
summary: In the CMU15445 project, the BufferPoolManager utilizes the LRU-K algorithm for memory management, handling virtual and physical page allocations within a buffer pool. Key functions include page creation, eviction based on access history, and writing 'dirty' pages back to disk, thereby optimizing database performance and memory use.
---

# Project1 BufferPoolManager

## LRU-K

LRU-K算法是干什么的？它是一个页面置换算法，系统将虚拟内存地址划分为各个虚拟页(page_id),将物理内存地址划分为物理页(frame_id)，当物理内存满了以后，就得决定该淘汰哪个虚拟页，从而能够加载新的虚拟页。

### 成员变量

LRUKReplacer成员变量

`std::unordered_map<frame_id_t, LRUKNode> node_store_;` frame_id具体存的LRUKNode

`size_t current_timestamp_{0};` 当前的时间戳，用每加入一个新页面自增来模拟

`size_t curr_size_{0};` 当前可淘汰的(evictable)页面数量

`size_t replacer_size_;` max size of frame

`size_t k_;` 

`std::mutex latch_;`

LRUKNode成员变量

`frame_id_t fid_;`

`std::list<size_t> history_;` 

`size_t k_;`

`bool is_evictable_{false};` 标记页面是否可以被淘汰

### 函数

LRUKNode函数

- `auto GetKDistance(size_t cur) -> uint64_t`

计算时间差，如果该页面的历史次数小于k次，则为正无穷，如果大于等于k次，则计算当前时间戳与第k次的时间差，并返回。

LRUKReplacer函数

- `auto Evict(frame_id_t *frame_id) -> bool;`

淘汰可淘汰的页面中(is_evictabe = true)，时间距离最远的page

- `void RecordAccess(frame_id_t frame_id, AccessType access_type = AccessType::*Unknown*);`

记录访问了哪个frame页面。考虑节点是否已经存在，如果不存在且replacer的size已经满的情况下访问失败，否则如果存在history_list加入当前时间戳，然后current_timestamp自增；如果不存在先建立新的LRUKNode节点，再做后续的步骤

- `void LRUKReplacer::SetEvictable(frame_id_t frame_id, bool set_evictable)`

该函数会影响curr_size的大小，因为curr_size返回的是当前可淘汰页面的数量，如果通过该函数把以前false的设置为true了，那curr_size要自增，反之则自减

- `void LRUKReplacer::Remove(frame_id_t frame_id)`

驱逐某个页面。首先要保证该页面是否存在，如果存在还要保证该页面是否可以被驱逐，在前两个条件都满足的情况下，可以驱逐这个页面，curr_size要减1。

## BufferPoolManager

### 成员变量

BufferPoolManager成员变量

`const size_t pool_size_;` bufferpool的size，代表了可以有多少页面在bufferpool里

`std::atomic<page_id_t> next_page_id_ = 0;` 下一个page_id，从0开始保持自增

`Page *pages_;` 一个数组，存放的是bufferpool中的页面，根据frame_id得到当前frame保存的page

`std::unique_ptr<DiskScheduler> disk_scheduler_ __attribute__((__unused__));` 负责向磁盘读写页面内容

`std::unordered_map<page_id_t, frame_id_t> page_table_;` 记录虚拟页面对应的物理内存页面

`std::unique_ptr<LRUKReplacer> replacer_;` 

`std::list<frame_id_t> free_list_;` 空闲的物理页面，如果有就不需要考虑替换哪个页面了

`std::mutex latch_;` latch 是一种轻量级的同步机制，用于保护短期操作，而 lock 是一种用于控制长期访问共享资源的同步机制，通常用于实现事务的隔离和一致性。

DiskScheduler

`DiskManager *disk_manager_ __attribute__((__unused__));`

`Channel<std::optional<DiskRequest>> request_queue_;` 处理请求队列

`std::optional<std::thread> background_thread_;` 在后台线程处理请求

```cpp
struct DiskRequest {
  /** Flag indicating whether the request is a write or a read. */
  bool is_write_;
  /**
   *  Pointer to the start of the memory location where a page is either:
   *   1. being read into from disk (on a read).
   *   2. being written out to disk (on a write).
   */
  char *data_;
  /** ID of the page being read from / written to disk. */
  page_id_t page_id_;
  /** Callback used to signal to the request issuer when the request has been completed. */
  std::promise<bool> callback_;
};
```

### 函数

BufferPoolManager

- 构造函数

构造函数需要做三件事，首先初始化replacer，第二根据pool_size初始化pages数组，最后把所有页面都放入free_list中。

- `auto NewPage(page_id_t *page_id) -> Page *;`

新生成一个虚拟页面，虚拟页面得有对应的物理页面。所以需要判断free_list有无可用的物理页面，如果有就直接取空闲页面，如果没有得调用replacer根据LRUK驱逐一个页面。如果驱逐出去的页面是dirty的，那我们得把这个页面写回给磁盘。然后要将新页面的pin_count设置为1，调用

replacer_->RecordAccess(frame_id);和replacer_->SetEvictable(frame_id, false);两个函数，因为生成了新页面以后肯定是要用的，所以要调用这两个函数。

- `auto NewPageGuarded(page_id_t *page_id) -> BasicPageGuard;`

该函数在project2中完成，用于管理pin_count，不需要手动unpin

- `auto FetchPage(page_id_t page_id, AccessType access_type = AccessType::*Unknown*) -> Page *;`

如果缓冲池中有该page，可以直接取到page，pin_count加1，调用

replacer_->RecordAccess(frame_id);replacer_->SetEvictable(frame_id, false);然后返回该page。

如果不存在，先确定page加载到哪个frame页面，然后从磁盘读取该page数据，加载到frame中。这一步其实与NewPage逻辑有点相似，只不过NewPage只要返回空的page就可以，但是fetchPage得返回从磁盘中读取的page

- `auto UnpinPage(page_id_t page_id, bool is_dirty, AccessType access_type = AccessType::*Unknown*) -> bool;`

通过is_dirty参数告诉bpm该页面是否被修改过，然后需要对引用次数减1。如果引用次数为0，那么调用replacer_->SetEvictable(frame_id, true);说明该页面已经被用完了，可以被驱逐了。

- `auto FlushPage(page_id_t page_id) -> bool;`

通过DiskScheduler将脏页面写回磁盘

```cpp
auto promise = disk_scheduler_->CreatePromise();
  auto future = promise.get_future();
  disk_scheduler_->Schedule({true, page->GetData(), page->GetPageId(), std::move(promise)});
  future.get();
```
