论文阅读

架构

![image-20210613105640119](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210613105640119.png)

master：持有所有文件系统的元数据，包括命名空间、访问控制信息、从文件到chunk的映射，当前chunks的位置。

chunkserver：存储chunks在linux本地磁盘上，每个chunk被复制多份放在不同的chunkserver

chunks：文件被划分为固定大小的chunkss，每个chunk被64位chunk handle标记，它是不可修改和全球唯一的，在chunk被创建是指定。

client





