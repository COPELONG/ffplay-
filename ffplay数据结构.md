# ffplay全面解析



**ffplay源码深度解析：https://zhuanlan.zhihu.com/p/44694286**



![image-20231019164430544](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231019164430544.png)

 ![image-20231021100249317](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231021100249317.png)![image-20231021100342793](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231021100342793.png)![image-20231021100314246](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231021100314246.png)![image-20231021100257258](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231021100257258.png)

## 数据结构

`struct VideoState` 主要用于管理视频播放过程中的状态信息。它包括了一系列字段，用于存储有关视频播放的各种参数和状态信息，如视频帧、音频帧、时钟、缓冲等。

![image-20231021100738210](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231021100738210.png)

## PacketQueue

![image-20231021101934008](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231021101934008.png)

![image-20231021102239188](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231021102239188.png)

![image-20231021103003744](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231021103003744.png)

### 启动队列start

```c++
static void packet_queue_start(PacketQueue *q)
{
    SDL_LockMutex(q->mutex);
    q->abort_request = 0;
    packet_queue_put_private(q, &flush_pkt);//这里放入了一个flush_pkt
    SDL_UnlockMutex(q->mutex);
}
//flush_pkt定义是static AVPacket flush_pkt;，是一个特殊的packet，主要用来作为非连续的两端数据的“分界”标记。
```

### put

```c++
static int packet_queue_put_private(PacketQueue *q, AVPacket *pkt)
{
    MyAVPacketList *pkt1;

    if (q->abort_request)//如果已中止，则放入失败
       return -1;

    pkt1 = av_malloc(sizeof(MyAVPacketList));//分配节点内存
    if (!pkt1)//内存不足，则放入失败
        return -1;
    pkt1->pkt = *pkt;//拷贝AVPacket(浅拷贝，AVPacket.data等内存并没有拷贝)
    pkt1->next = NULL;
    if (pkt == &flush_pkt)//如果放入的是flush_pkt，需要增加队列的序列号，以区分不连续的两段数据
        q->serial++;
    pkt1->serial = q->serial;//用队列序列号标记节点

    //队列操作：如果last_pkt为空，说明队列是空的，新增节点为队头；否则，队列有数据，则让原队尾的next为新增节点。 最后将队尾指向新增节点
    if (!q->last_pkt)
        q->first_pkt = pkt1;
    else
        q->last_pkt->next = pkt1;
    q->last_pkt = pkt1;

    //队列属性操作：增加节点数、cache大小、cache总时长
    q->nb_packets++;
    q->size += pkt1->pkt.size + sizeof(*pkt1);
    q->duration += pkt1->pkt.duration;

    /* XXX: should duplicate packet data in DV case */
    //发出信号，表明当前队列中有数据了，通知等待中的读线程可以取数据了
    SDL_CondSignal(q->cond);
    return 0;
}

计算serial。serial标记了这个节点内的数据是何时的。一般情况下新增节点与上一个节点的serial是一样的，但当队列中加入一个flush_pkt后，后续节点的serial会比之前大1.
```

![image-20231124150422110](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231124150422110.png)





-----------

------------------

## FrameQueue

节点内存以数组形式预分配，无需动态分配

```c++
    for (i = 0; i < f->max_size; i++)
        if (!(f->queue[i].frame = av_frame_alloc()))
```

环形缓冲区

![image-20231021105954540](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231021105954540.png)

### write

FrameQueue的“写”分两步，先调用`frame_queue_peek_writable`获取一个可写节点，在对节点操作结束后，调用`frame_queue_push`告知FrameQueue“存入”该节点。

```c++
Frame* vp = frame_queue_peek_writable(q);//获取一个可写入的缓冲区，若无，则加锁循环等待
//将要存储的数据写入frame字段，比如：
av_frame_move_ref(vp->frame, src_frame);
//存入队列
frame_queue_push(q);//更新windex索引。这个索引时钟指向一个可写入的缓冲区
```

![image-20231124120254886](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231124120254886.png)

### read：

```c++
Frame* vp = frame_queue_peek_readable(f);//返回一个可读的缓冲区
//读取vp的数据，比如
printf("pict_type=%d\n", vp->frame->pict_type);
frame_queue_next(f);//更新rindex索引，始终指向一个可读的缓冲区
```

```c++
static void frame_queue_next(FrameQueue *f)
{
    //如果支持keep_last，且rindex_shown为0，则rindex_shown赋1，返回
    if (f->keep_last && !f->rindex_shown) {
        f->rindex_shown = 1;
        return;
    }

    //否则，移动rindex指针，并减小size
    frame_queue_unref_item(&f->queue[f->rindex]);
    if (++f->rindex == f->max_size)
        f->rindex = 0;
    SDL_LockMutex(f->mutex);
    f->size--;
    SDL_CondSignal(f->cond);
    SDL_UnlockMutex(f->mutex);
}
```

![image-20231124120633625](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231124120633625.png)

什么时候释放frame数据：只有更新读索引的时候才真正释放对应的Frame数据.

size属性解析：当keep_last =1 时，说明在队列中保留前一帧数据不释放，销毁队列的时候才将其真正释放。

![image-20231021144324669](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231021144324669.png)

![image-20231124120650386](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231124120650386.png)

![image-20231021144543919](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231021144543919.png)

![image-20231021144703655](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231021144703655.png)



参考：https://zhuanlan.zhihu.com/p/43564980

















