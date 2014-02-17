Title:  ptmalloc中的fastbin chunk的合并过程
Date: 2014-02-17 10:04:00
Category: ptmalloc
Tags: ptmalloc
Slug: ptmalloc
Author: djjsindy
Summary:  最近在设计一个memcache协议队列的内存管理模块，其实`malloc`和`free`并不是想象中的那种，`malloc`完空间`free`就会马上把内存返回给操作系统，所以了解一下glibc 中malloc和free函数的实现是十分必要的。了解了glibc中`malloc`实现原理对于设计一个自己的内存管理模块是十分有意义的，可以从中学到很多的优良设计。
#ptmalloc中的fastbin chunk的合并过程

&nbsp;&nbsp;&nbsp;&nbsp;最近在设计一个memcache协议队列的内存管理模块，其实`malloc`和`free`并不是想象中的那种，`malloc`完空间`free`就会马上把内存返回给操作系统，所以了解一下glibc 中malloc和free函数的实现是十分必要的。了解了glibc中`malloc`实现原理对于设计一个自己的内存管理模块是十分有意义的，可以从中学到很多的优良设计。

&nbsp;&nbsp;&nbsp;&nbsp;glibc中`malloc`类函数的实现是ptmalloc，分配内存大致思想就是依赖于底层`brk`或者`mmap`系统调用，来开辟新的地址空间，`free`函数会把释放的地址空间放到不同的容器中，fastbin存放的都是一些`free`后小的地址空间，具体这个阈值是可以`mallopt`函数进行配置的，默认的size_t*16字节。

&nbsp;&nbsp;&nbsp;&nbsp;分配出的地址空间的结构体是`malloc_chunk`，而`malloc_chunk`按照所存储的地址空间的大小，存放在不同的容器中，有fastbin，small bin，unsorted bin ,large bin,但是本文只分析small bin和fast bin

fastbin结构图：

<img src="https://lh4.googleusercontent.com/-EEFeehsWSp0/Uv19DpYYahI/AAAAAAAAAM4/rfPLFp-gk9I/s800/Screen%2520Shot%25202014-02-14%2520at%252010.16.03%2520AM.png" height="300"/>

smallbin结构图：

<img src="https://lh4.googleusercontent.com/--MnFI0y8sDI/Uv2InfaFz6I/AAAAAAAAANY/NnjZzvrTP_8/s912/Screen%2520Shot%25202014-02-14%2520at%252011.05.00%2520AM.png" height="370"/>


&nbsp;&nbsp;&nbsp;&nbsp;从上面两张图可以看出都是由数组＋双向链表构成，fast bin缓存16B－64B大小的`malloc _chunk`，而small bin缓存的是16B－511B大小的`malloc_chunk`，这两个bin存储的`malloc_chunk`大小是有重叠的，fast bin存在的意义就在于非fast bin中`malloc_chunk`在free的时候，如果这个chunk前后的chunk是空闲的，那么这个chunk会合并，然后放到unsort bin中，这里需要注意的是这个前后，并不是指的是`malloc_chunk`双向链表的prev和next，而是虚拟地址空间前后，试想malloc函数肯定是通过brk或者mmap分配一段虚拟地址空间，然后从这个大地址空间根据malloc大小切割地址空间，所以这样在地址空间相邻的`malloc_chunk`才能彼此合并。双向链表的前后元素在地址空间上根本不能保证相邻。在合并的过程中肯定会判断地址空间相邻的`malloc_chunk`是否是空闲，如果不是空闲，那么肯定是被malloc函数分配出去了，还有一种可能就是fast bin中的`malloc_chunk`。fast bin中malloc_chunk的特点就是不会清除空闲标志，free chunk之后直接放到了fast bin中，并且保证不会被合并。保证了`malloc_chunk`空闲容器中会存在这种小的地址空间，方便malloc快速分配小地址空间。

&nbsp;&nbsp;&nbsp;&nbsp;通过上面的分析，如果malloc函数请求的大小小于等于64B，那么首先会去fast bin中找合适的`malloc_chunk`，这样malloc的速度会大大加快，主要是fast bin中`malloc_chunk`保证了不会其他容器的`malloc_chunk`被合并。

&nbsp;&nbsp;&nbsp;&nbsp;通过代码分析一下，在free `malloc_chunk`的过程中会尝试与地址空间相邻的空闲的`malloc_chunk`进行合并(fast bin 中的chunk不清除inuse状态)。

free函数片段

	/* consolidate backward */  //如果前面的chunk空闲，合并前面的chunk
    if (!prev_inuse(p)) {
      prevsize = p->prev_size;
      size += prevsize;
      p = chunk_at_offset(p, -((long) prevsize)); //必须更新这个chunk的首地址
      unlink(p, bck, fwd);
    }

    if (nextchunk != av->top) {
      /* get and clear inuse bit */
      nextinuse = inuse_bit_at_offset(nextchunk, nextsize); 

      /* consolidate forward */  //判断后面的chunk是否空闲
      if (!nextinuse) {
		unlink(nextchunk, bck, fwd); //如果空闲，不需要在改变首地址了，只需要更新size
		size += nextsize;
      } else
		clear_inuse_bit_at_offset(nextchunk, 0); //如果后面的chunk在用，更新标志。


&nbsp;&nbsp;&nbsp;&nbsp;为什么上面的chunk只关心相邻的chunk，不用再去递归再相邻的chunk了，因为相邻的chunk如果它的周围还是空闲的，那么之前的free过程肯定会去合并他们，所以这里根本不用考虑那种情况。在fast bin中我们不必担心`malloc_chunk`中的空闲状态，这个状态只是给合并操作看的，告诉不要合并它们。任何容器中的chunk都是空闲的。一旦被malloc函数所征用，会移除容器的。

&nbsp;&nbsp;&nbsp;&nbsp;那么fast bin中的chunk也不是一成不变的，在一些情况下fast bin中的chunk也会被merge移动到unsort bin中，其中fast bin的chunk会参考相邻的chunk，如果是空闲的就会被fast bin中的chunk所合并，然后移动到unsort bin中。也就是说有一些除了fast bin中彼此虚拟地址空间相邻的被合并到unsort bin，small bin或者large bin中的一些和fast bin地址相邻的chunk合并到unsort bin，

1.	如果size是small bin中的，对应的链表没有chunk，那么会合并一下fast bin中的chunk,这样有可能产生满足chunk。

2.	如果size是large bin中的，直接合并一下fast bin中的chunk，注释上说避免fast bin中太多碎片，很有可能在合并过后，分配到了large bin中，正是我们所需要的。

3.	在调用brk或者mmap函数之前会最后一次尝试合并一下fast bin中chunk。

4.	在free函数的最后，如果free的`malloc_chunk`和虚拟地址空间相邻的空闲chunk合并后的size>    `FASTBIN_CONSOLIDATION_THRESHOLD`,那么会合并fast bin中的chunk，目的是更新top chunk的指针，因为fast bin中`malloc_chunk`都是使用状态，这样合并后，会更新top chunk的指针，因为合并操作会合并地址空间相邻的chunk，如果chunk的next是top chunk指针，说明可以更新top chunk指针了，这样更有利于释放空间到系统。


下面看下合并fast bin中`malloc_chunk`的代码：
<pre><code>
static void malloc_consolidate(mstate av)
{
  mfastbinptr*    fb;                 /* current fastbin being consolidated */
  mfastbinptr*    maxfb;              /* last fastbin (for loop control) */
  mchunkptr       p;                  /* current chunk being consolidated */
  mchunkptr       nextp;              /* next chunk to consolidate */
  mchunkptr       unsorted_bin;       /* bin header */
  mchunkptr       first_unsorted;     /* chunk to link to */

  /* These have same use as in free() */
  mchunkptr       nextchunk;
  INTERNAL_SIZE_T size;
  INTERNAL_SIZE_T nextsize;
  INTERNAL_SIZE_T prevsize;
  int             nextinuse;
  mchunkptr       bck;
  mchunkptr       fwd;

  /*
     If max_fast is 0, we know that av hasn't
     yet been initialized, in which case do so below
   */

  if (get_max_fast () != 0) {  //表示已经初始化过了
    clear_fastchunks(av);

    unsorted_bin = unsorted_chunks(av); 

    maxfb = &fastbin (av, NFASTBINS - 1);
    fb = &fastbin (av, 0);
    //开始递归fastbin每一个双向链表的每一个元素，第一层递归每一个链表，第二层递归链表中每一个元素
    do {
      p = atomic_exchange_acq (fb, 0);
      if (p != 0) {
        do {
          check_inuse_chunk(av, p);
          nextp = p->fd;
          size = p->size & ~(PREV_INUSE|NON_MAIN_ARENA);
          nextchunk = chunk_at_offset(p, size);
          nextsize = chunksize(nextchunk);
          //如果虚拟地址空间的前一个chunk是空闲的，就开始合并到当前chunk中，同时更新size，和p指针，把前一个chunk从它的链表中摘除
          if (!prev_inuse(p)) {
            prevsize = p->prev_size;
            size += prevsize;
            p = chunk_at_offset(p, -((long) prevsize));
            unlink(p, bck, fwd);
          }
		  //判断next chunk是否是top chunk，就是说判断当前chunk的虚拟地址空间的后面是否是top chunk的初始地址。
          if (nextchunk != av->top) {
            nextinuse = inuse_bit_at_offset(nextchunk, nextsize);
			//如果next不是top chunk，那么当前chunk就开始合并next chunk
            if (!nextinuse) {
              size += nextsize;
              unlink(nextchunk, bck, fwd);
            } else
              clear_inuse_bit_at_offset(nextchunk, 0);//如果next chunk不空闲，那么更新它的inuse状态，因为chunk的是否空闲决定于next chunk（chunk的结构决定）
			//放到unsorted bin中
            first_unsorted = unsorted_bin->fd;
            unsorted_bin->fd = p;
            first_unsorted->bk = p;

            if (!in_smallbin_range (size)) {
              p->fd_nextsize = NULL;
              p->bk_nextsize = NULL;
            }

            set_head(p, size | PREV_INUSE);
            p->bk = unsorted_bin;
            p->fd = first_unsorted;
            set_foot(p, size);
          }
		  //如果当前chunk地址空间紧紧挨着top chunk的指针，那么更新top chunk指针就行了，相当于合并了
          else {
            size += nextsize;
            set_head(p, size | PREV_INUSE);
            av->top = p;
          }

        } while ( (p = nextp) != 0);
      }
    } while (fb++ != maxfb);
  }
  else {
    malloc_init_state(av);
    check_malloc_state(av);
  }
}

</code></pre>

总结：

&nbsp;&nbsp;&nbsp;&nbsp;ptmalloc的分配中，fast bin中chunk，通过不清除inuse状态位，防止了free函数，被动的把fast bin中的chunk合并到其他bin中，大大加速连续小块内存分配的速度；fast bin中的chunk在一定条件是还是会被合并到unsorted bin中，然后分配到small bin或者large bin中的，减少内存碎片，有机会满足大块内存的请求。



	



