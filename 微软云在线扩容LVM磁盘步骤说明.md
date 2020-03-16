#### 微软云在线扩展LVM磁盘步骤说明

*作者：floyd*

##### 环境说明：

​	当前微软云主机独立挂在的数据盘因业务需要，需要对磁盘进行扩容，一方面要达到预期容量增长需求，一方面也是升级磁盘达到相应的IOPS需求。独立磁盘在系统挂在的磁盘名字为/dev/sdc，当前是只有一个分区 /dev/sdc1分区。数据盘使用LVM管理，唯一分区/dev/sdc1上创建了逻辑卷/dev/cp/data。逻辑卷被挂在在/disktest目录下

![image-20200314191353488](https://raw.githubusercontent.com/acheng-floyd/markdown_pic/master/img/image-20200314191353488.png)

![image-20200314191330624](https://raw.githubusercontent.com/acheng-floyd/markdown_pic/master/img/image-20200314191330624.png)

##### 扩容思路：

​		扩容的思路是先将磁盘上的应用停掉，将磁盘umount掉（不umount会导致将磁盘与虚拟机分离后挂在回来报错的问题），然后在微软云管理后台将磁盘与虚拟机分离；分离后再在管理平台中修改磁盘大小，然后重新挂回到虚拟机中；挂回虚拟机后对挂在后的数据盘进行分区，并且将分区格式改为LVM格式；最后将新增加的分区添加到原来的Volume Group cp中，最后将新增的空间扩容到原来的data逻辑卷。

##### 扩容步骤：

1. 首先把有数据保存在要扩容磁盘上并且正在运行中的应用停掉，比如mysql，确保能顺利umount磁盘。

   ![image-20200314191642112](https://raw.githubusercontent.com/acheng-floyd/markdown_pic/master/img/image-20200314191642112.png)

2.  在微软虚拟机管理后台，将数据磁盘与虚拟机分离，分离后保存虚拟机设置

   ![image-20200314192750244](https://raw.githubusercontent.com/acheng-floyd/markdown_pic/master/img/image-20200314192750244.png)

3. 在所有资源中找到刚刚被分离的数据磁盘，编辑磁盘大小为需要扩容到的磁盘大小，并且保存

   ![image-20200314193116030](https://raw.githubusercontent.com/acheng-floyd/markdown_pic/master/img/image-20200314193116030.png)

4. 重新将修改大小后的数据盘添加到原来的虚拟机中，fdsik可以看到系统中的磁盘已经扩容到500G了

   ![image-20200314193356705](https://raw.githubusercontent.com/acheng-floyd/markdown_pic/master/img/image-20200314193356705.png)

   ![image-20200314193621593](https://raw.githubusercontent.com/acheng-floyd/markdown_pic/master/img/image-20200314193621593.png)

5. 使用fdsik对挂载的扩容后的磁盘重新进行分区，将扩容后的空间分配到新增加的分区中，目前只能看到老的分区。（此处要注意的是重新挂在扩容后的数据盘后，系统看到的磁盘名字已经变成了/dev/sdd）。使用fdisk分区是用命令n创建分区，默认创建主分区，First sector默认会自动算出来，接着原来那个分区，默认回车就行；Last sector也会自动算出来，默认也回车；回车完后，fdisk显示Partition 2 of type Linux and of size 300GiB is set,说明新创建的分区为300G，正好是我们扩容后的磁盘空间

   ![image-20200314200134431](https://raw.githubusercontent.com/acheng-floyd/markdown_pic/master/img/image-20200314200134431.png)

6. 将新创建的磁盘分区类型修改为LVM类型，然后按w保存分区退出fdisk

   ![image-20200314200905734](https://raw.githubusercontent.com/acheng-floyd/markdown_pic/master/img/image-20200314200905734.png)

7. 在新创建的分区上创建物理卷，pvcreate /dev/sdd2

   ![image-20200314201046759](https://raw.githubusercontent.com/acheng-floyd/markdown_pic/master/img/image-20200314201046759.png)

8. 用新创建的物理卷扩容原来的Volume Group cp,可以看到扩容后的volume group cp已经有剩余空间300G

   ![image-20200314201258241](https://raw.githubusercontent.com/acheng-floyd/markdown_pic/master/img/image-20200314201258241.png)

9. 将volume group中的剩余空间扩容到原来的逻辑卷中

   ![image-20200314202312540](https://raw.githubusercontent.com/acheng-floyd/markdown_pic/master/img/image-20200314202312540.png)

10.  将逻辑卷重新挂会到原来的目录，此时发现磁盘容量还没显示成扩容后的容量，还需要将扩容后的空间进行格式化扩容

    ![image-20200314202706123](https://raw.githubusercontent.com/acheng-floyd/markdown_pic/master/img/image-20200314202706123.png)

    因为我们使用的xfs分区格式，使用命令xfs_growfs命令

    ![image-20200314202936399](https://raw.githubusercontent.com/acheng-floyd/markdown_pic/master/img/image-20200314202936399.png)

至此，我们已经完成了在线磁盘扩容操作

