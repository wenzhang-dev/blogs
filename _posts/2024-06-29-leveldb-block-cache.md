---
layout: post
title: 代码精读| LevelDB BlockCache
categories: LevelDB
description: LevelDB 在 SST Table 中使用 block cache，缓存热点 block 数据，减少磁盘 IO
keywords: LevelDB
---

## 简介

LevelDB 在 SST Table 中使用 block cache，缓存热点 block 数据，减少磁盘 IO。

## 实现

在打开 LevelDB 数据库时，可以使用配置 Options 参数，设置 block cache 大小，其默认大小为 8 MB。

```cpp
Options SanitizeOptions(const std::string& dbname,
						const InternalKeyComparator* icmp,
						const InternalFilterPolicy* ipolicy,
						const Options& src) {
   // 略
   Options result = src;
   if (result.block_cache == nullptr) {
     result.block_cache = NewLRUCache(8 << 20); // 默认使用 8 MB block_cache
   }
   return result;
}
```

SST Table 读取 block 大致流程
* 如果开启了 block cache，则拼接 cache_key，尝试在 cache 中查找
* 如果找到了直接返回，否则就需要从磁盘读取 block，并插入到 block_cache
* 如果没有开启 block cache，则直接从磁盘读取 block，并直接返回

如果 block_cache 仅使用 block offset 作为 key，那么不同的 sst，它们的 block offset 可能是相同的。所以，每个 sst 应该有一个唯一的标识，这就是 cache_id。通过 cache_id 和 block offset 组装出来的 key，就不会冲突。

```cpp
Iterator* Table::BlockReader(void* arg, const ReadOptions& options, const Slice& index_value) {
  Table* table = reinterpret_cast<Table*>(arg);
  Cache* block_cache = table->rep_->options.block_cache;
  Block* block = nullptr;
  Cache::Handle* cache_handle = nullptr;

  BlockContents contents;
  if (block_cache != nullptr) {
    char cache_key_buffer[16];
    EncodeFixed64(cache_key_buffer, table->rep_->cache_id);
    EncodeFixed64(cache_key_buffer + 8, handle.offset());
    Slice key(cache_key_buffer, sizeof(cache_key_buffer)); // 使用 cache_id 和 block offset 拼接 key
    cache_handle = block_cache->Lookup(key);
    if (cache_handle != nullptr) { // 从 cache 找到，直接返回 Block
      block = reinterpret_cast<Block*>(block_cache->Value(cache_handle));
    } else { // 不在 cache，就需要读 block，然后构造出 Block
      s = ReadBlock(table->rep_->file, options, handle, &contents);
      if (s.ok()) {
        block = new Block(contents);
        if (contents.cachable && options.fill_cache) { // cachable 且开启了 fill_cache
	      // 插入 block_cache 时，charge 填入了 block_size
          cache_handle = block_cache->Insert(key, block, block->size(), &DeleteCachedBlock);
        }
      }
    }
  } else { // 未开启 block_cache，就直接读
     s = ReadBlock(table->rep_->file, options, handle, &contents);
     if (s.ok()) {
       block = new Block(contents);
     }
  }
  // 略
}

static void DeleteCachedBlock(const Slice& key, void* value) {
  Block* block = reinterpret_cast<Block*>(value); // LRUHandle 节点被析构时，回调
  delete block;
}
```

获取 DB 内部状态时，调用 GetProperty 可以获取 block_cache 内存使用量。其主要利用 LRUCache charge 机制，统计用量。

```cpp
bool DBImpl::GetProperty(const Slice& property, std::string* value) {
  // 略
  if (in == "approximate-memory-usage") {
    size_t total_usage = options_.block_cache->TotalCharge();
    if (mem_) {
      total_usage += mem_->ApproximateMemoryUsage();
    }
    if (imm_) {
      total_usage += imm_->ApproximateMemoryUsage();
    }
  }
}
```


