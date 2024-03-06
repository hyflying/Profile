---
title: Extendible Hash
date: '2024-03-01'
summary: CMU15445's Project2 implements an Extendible Hash for dynamic data indexing. It manages keys with global and local depths and handles variable scenarios during insertion. Identified bugs pertain to BufferPoolManager capacity and invalid page handling.
---

# Project2 Extendible Hash

# 算法简述

两个重要概念

- directory：目录页，根据取最低的global depth位bits，得到key对应的bucket
- bucket：存hashed key

三个重要变量：

- global depth：规定看最低多少位bits，global depth为n，有2^n个directories
- local depth：bucket的depth，规定bucket只看最低local depth位的bits来分配hashed key放到哪个bucket中，local depth不能大于global depth
- bucket size：规定了一个bucket里最多放多少个hashed key

## insert一个key

讨论最常见的三种情况。

1. bucket没有满

这种情况下直接插入就行

1. bucket满了，但是bucket的local depth还是小于global depth

此时需要split bucket，并将local depth+1，更新directory指向local depth的指针，同时根据新的local depth重新分配被分裂桶中的元素以及新加的元素

1. bucket满了，并且bucket的local depth等于global depth

因为local depth不能大于global depth，因此这种情况下，需要global depth先加1，global depth加1之后，directories要✖️2，就是复制一份原来的directories，bucket的local depth也要加1，然后split bucket，directory指向分裂的buckets的指针要重新分配，分裂的buckets里的hashed key也要重新分配。

# 项目代码简述

## ExtendibleHTableDirectoryPage

### 成员变量

```cpp
uint32_t max_depth_; 

uint32_t global_depth_; 

uint8_t local_depths_[HTABLE_DIRECTORY_ARRAY_SIZE]; //local depth数组

page_id_t bucket_page_ids_[HTABLE_DIRECTORY_ARRAY_SIZE];//指向对应的bucket
```

### 函数

这里的bucket_idx是根据key以及global depth hash得来的，指向具体的bucket

```cpp
auto ExtendibleHTableDirectoryPage::GetSplitImageIndex(uint32_t bucket_idx) const -> uint32_t {
  //GetSplitImageIndex要在local depth + 1之后。
  uint32_t local_depth = local_depths_[bucket_idx];
  return bucket_idx^(1U << (local_depth - 1));
}
```

## ExtendibleHTableBucketPage

这里的bucket_idx是指向当前桶里的某个元素的idx，桶里元素是key，value的pair

## ExtendibleHTableHeaderPage

创建一个headerPage，不能用构造函数。要从bpm中拿到新的一个page

```cpp
page_id_t header_page_id = INVALID_PAGE_ID;
BasicPageGuard header_guard = bpm_->NewPageGuarded(&header_page_id);
header_page_ = header_guard.AsMut<ExtendibleHTableHeaderPage>();
header_page_->Init(header_max_depth_);
```

## PageGuard

```
auto BasicPageGuard::UpgradeWrite() -> WritePageGuard
auto BasicPageGuard::UpgradeRead() -> ReadPageGuard
```

page先加对应的锁，然后再生成Read/WritePageGuard,原先的bpm_和page_置空

# Bug记录

1. BufferPoolManager的pool size很小的时候，如果fetch完页面，用完之后不及时释放锁，可能会导致pool里所有页面都是锁着的，而此时如果再fetch新页面，会有问题。因此用完页面以后要及时释放
2. GetValue的时候，可能访问的directory_page或者bucket_page都不存在，那这种情况下返回的page_id会是IN_VALID_PAGE_ID,需要直接返回false
