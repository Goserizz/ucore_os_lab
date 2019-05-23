# **Lab8 Report**

### **练习1: 完成读文件操作的实现**

- ### sfs_io_nolock的实现

  sfs_bmap_load_nolock

  > 获取index对应的ino，具体的脏活累活交由sfs_bmap_get_nolock去做，而它主要善后工作。如果申请的index超出了din->blocks，即申请的index还并没有对应的文件系统，需要吩咐sfs_bmap_get_nolock去做这件事。

  sfs_bmap_get_nolock

  > 具体的ino查询函数。如果index值小于12，说明申请的是direct中的inode，若direct\[index\] == 0，则说明还未创建对应的inode（ino为0代表超级块），调用sfs_block_alloc进行创建并从其返回值得到ino，若direct\[index\] != 0，则直接返回它。若index不小于12，则说明它请求的是indirect间接索引块里的数据块，且index-12即为它在间接索引块里的index，调用sfs_bmap_get_sub_nolock进行递归查询。

  实现过程

  > 首先需要判断起始的offset是否处于某一数据块的开始位置，若不是，则第一块不能直接整块读取，若结束的endpos大于第一块的末位置，则说明要读取不止一块，所以第一块应读取offset到第一块末位置，若endpos小于第一块的末位置，则只需读取该块的offset到endpos的数据。调用sfs_bmap_load_nolock函数获取当前第一块的ino值，然后调用sfs_buf_op进行读写操作。

  > 中间的块是完整的整块读写，仍然调用sfs_bmap_load_nolock函数获取当前读写块的ino值，然后调用sfs_block_op进行整块读写

  > 最后一块需要判断读取从该块其实位置到endpos，用endpos % SFS_BLKSIZE可以计算出endpos在该块中的位置，仍然调用sfs_bmap_load_nolock函数获取最后一块的ino值，然后调用sfs_buf_op进行读写操作。

  ```C
    blkoff = offset % SFS_BLKSIZE;
    if(blkoff != 0){
        size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset);
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0)
            goto out;
        if ((ret = sfs_buf_op(sfs, buf, size, ino, blkoff)) != 0)
            goto out;
        alen += size;
        if(nblks == 0)
            goto out;
        buf += size;
        blkno ++;
        nblks --;
    }

    while(nblks != 0){
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0){
            goto out;
        }
        if ((ret = sfs_block_op(sfs, buf, ino, 1)) != 0){
            goto out;
        }
        alen += SFS_BLKSIZE;
        buf += SFS_BLKSIZE;
        blkno ++;
        nblks --;
    }

    blkoff = endpos % SFS_BLKSIZE;
    if(blkoff != 0){
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0){
            goto out;
        }
        if ((ret = sfs_buf_op(sfs, buf, blkoff, ino, 0)) != 0){
            goto out;
        }
        alen += blkoff;
    }
    ```

- ### 实现UNIX的PIPE机制的概要设计方案

  > 管道的描述符即为文件的描述符，在进程结构体proc_struct新增读管道描述符数组和写描述符数组，两个要进行管道交流的进程初始化时创建管道的描述符及其文件系统，分别存入proc_struct的读写描述符数组中，这样两个进程就可以从管道描述符数组中获取管道描述符，利用描述符进行对管道的读写操作。



### **练习2 完成基于文件系统的执行程序机制的实现**

- ### 改写proc.c中的load_icode函数和其他相关函数

  > 该练习我无从下手，便复制答案的代码进行实现的。我仔细阅读并理解了答案的代码，在代码上添加了注释。

- ### 实现UNIX的应链接和软链接机制的概要方案

  > 软硬链接都是inode。软链接在创建时不增加它链接inode的链接数，释放链接时也不减少链接inode的链接数，硬链接在创建或释放时要将所有硬链接inode的链接数加1或减1，当某个硬链接释放后inode的链接数减到0，则将它对应硬盘空间释放。