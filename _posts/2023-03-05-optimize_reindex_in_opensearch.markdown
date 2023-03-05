---
layout: post
title: "优化OpenSearch集群的Reindex任务性能"
date: 2023-03-05 09:13:36 +0800
categories: 笔记
---

OpenSearch将数据以文档（Document）的形式存储在索引（Index）中，而索引可以根据需要划分为多个分片（Shard）。

每个索引和分片都需要一些内存资源来记录内部数据结构（segment）在磁盘上的分布和数据类型。通常来讲，大量的小索引/分片会比少量的大索引/分片需要更多内存。因此，OpenSearch集群需要尽可能降低索引/分片数量，以提高资源利用效率。通常来讲，10GB-50GB的分片大小是合理的。

Reindex就是一种将已有索引合并的方法。

```
POST  _reindex
{
    "source": {
        "index": [
            "idx-1",
            "idx-2"
        ]
    },
    "dest": {
        "index": "new-idx"
    }
}
```

Reindex的默认参数效率较低，任务执行时间很长。稍加优化，就可以节约不少时间。

1. 调整任务批量读取数据的文档数量 `size`，默认为1000。最大可以是10000。
2. 并行化执行 ，设置`slices`为一个合理数字。
3. 目标索引的`refresh_interval`默认为1s。设置为更长时间可以减少刷盘的频率。
4. 目标索引的`number_of_replica`设置为0。避免在副本上浪费多余的时间。
5. 如果是多个索引较大的索引需要合并。可以分别合入同一个索引中，而不是一起合入新的索引。例如需要，将idx-1,idx-2,idx-3合并。

    - 这个方案较慢
    ```
    POST  _reindex
    {
        "source": {
            "index": [
                "idx-1",
                "idx-2",
                "idx-3"
            ]
        },
        "dest": {
            "index": "new-idx"
        }
    }
    ```
    - 这个方案更快
    ```
    POST  _reindex?slices=5
    {
        "source": {
            "index": "idx-2",
            "size": 10000
        },
        "dest": {
            "index": "idx-1"
        }
    }

    # 前一个任务执行结束后，再执行下一个
    POST  _reindex?slices=5
    {
        "source": {
            "index": "idx-3",
            "size": 10000
        },
        "dest": {
            "index": "idx-1"
        }
    }
    ```
6. Reindex比较消耗CPU资源，通常都需要异步的执行，可以给URI加上`wait_for_completion=false`。返回响应中包含一个task对象id。随后可以通过Task API来确认Reindex的任务进度。例如，`GET tasks/<task_id>`。

### 参考资料

- [Operational best practices for Amazon OpenSearch Service](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/bp.html)
- [Reindex API](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/docs-reindex.html)
- [Size your shards](https://www.elastic.co/guide/en/elasticsearch/reference/master/size-your-shards.html)
