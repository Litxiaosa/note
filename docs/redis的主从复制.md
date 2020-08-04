#### **redis的主从复制**

1:  下载、解压，然后进入到redis，执行

​	`make`



2: 进入到配置文件: redis.conf,（主从服务器都改）

​	<img src="https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200804154723167.png" alt="image-20200804154723167" style="zoom:50%;" />

  注释掉！取消只允许在本地访问的限制

​	<img src="https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200804155114237.png" alt="image-20200804155114237" style="zoom:50%;" />

  protected-mode no  取消掉保护模式

​	<img src="https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200804155143549.png" alt="image-20200804155143549" style="zoom:50%;" />

 daemonize yes  守护进程改称yes



3: 从服务器增加

 <img src="https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200804155233408.png" alt="image-20200804155233408" style="zoom:50%;" />



4: 启动

​	进入到src 目录下，命令：

 `./redis-server ../redis.conf`



##### **哨兵模式(主从都改)**

1: 修改 :vim sentinel.conf

<img src="https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200804155401940.png" alt="image-20200804155401940" style="zoom:50%;" />

  sentinel monitor mymaster 127.0.0.1 6379 2  改称Master的ID和端口号，2:代表多数派，集群一半+1

2: 启动哨兵：

`./redis-server ../sentinel.conf --sentinel` 

