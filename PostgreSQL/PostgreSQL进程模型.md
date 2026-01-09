“每用户一进程模式” 客户端/服务器 模式。实际上是“**每连接一进程**”。

使用 `ps -ef | grep postgres`命令能看到至少有如下几个进程：
postgres: logger
postgres: checkpointer
postgres: background writer
postgres: walwriter
postgres: autovacuum launcher
postgres: logical replication launcher
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260107144614093.png)

使用`pstree -ap | grep postgres`命令看进程树，看到主进程postgres是所有其他进程的父进程
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260107150843465.png)

奇怪？
为什么主进程在14版本及以后的文档中显示叫[postmaster](https://www.postgresql.org/docs/18/glossary.html#GLOSSARY-POSTMASTER "Postmaster (process)") ，在13的文档里叫`postgres`。但是17版本的PostgreSQL的主进程实际名称就是`postgres` 。

![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260107180725249.png)

![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260107180901760.png)
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260107180807024.png)
