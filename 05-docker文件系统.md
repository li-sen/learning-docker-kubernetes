# 前言
说的docker文件系统，就不得不提AUFS，AUFS作者也是挺有意思的一个日本人，至今AUFS也没进入linux主干中，
难道跟linus有仇么？不过还好比较激进的ubantu接纳了他，现在这么多人能使用它这种思想的文件系统也算是一种成功了。
那我们先简单来说下AUFS。

# AUFS
AUFS Advance UnionFS 的简称，是 UnionFS 后期版本，所谓UnionFS就是把不同物理位置的目录合并mount到同一个目录中；
例如 把一张CD/DVD和一个硬盘目录给联合 mount在一起，然后，你就可以对这个只读的CD/DVD上的文件进行修改，修改的文件存于硬盘上的目录里。

我们一起来做个AUFS示例：
```bash
# 先创建两个目录，然后进行联合挂载，我这里使用的ubantu系统 centos要使用AUFS需要升级带aufs模块的内核。
root@shadowsocket01:~/test# tree
.
├── a
│   ├── 1
│   └── 2
└── b
    ├── 2
    └── 3

2 directories, 4 files
root@shadowsocket01:~/test# mkdir mnt
root@shadowsocket01:~/test# mount -t aufs -o dirs=./a:./b none ./mnt

# 往 mnt 2中写入内容
root@shadowsocket01:~/test# echo 'test' > ./mnt/2
root@shadowsocket01:~/test# cat a/2
test
root@shadowsocket01:~/test# cat b/2
root@shadowsocket01:~/test#
# 可以看到 a/2 内容变了，而 b/2 却没变
# 这是因为 aufs 默认上来说，命令行上第一个（最左边）的目录是可读可写的，后面的全都是只读的。

# 当然我们也可以指定权限 进行挂载
root@shadowsocket01:~/test# > ./mnt/2
root@shadowsocket01:~/test# cat a/2
root@shadowsocket01:~/test# cat b/2
root@shadowsocket01:~/test# umount ./mnt
root@shadowsocket01:~/test# mount -t aufs -o dirs=./a=rw:./b=rw none ./mnt
root@shadowsocket01:~/test# echo 'test' > ./mnt/2
root@shadowsocket01:~/test# cat a/2
test
root@shadowsocket01:~/test# cat b/2
root@shadowsocket01:~/test#
# 可见，如果有重复的文件名，在mount命令行上，越往前的就优先级越高，只对优先级高的进行操作。

```