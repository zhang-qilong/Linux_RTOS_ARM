​
@[toc]
# 1、什么是交叉编译工具链
交叉编译工具链是一个用于在一种操作系统上编译可在另一种操作系统上运行的程序的工具集合。ARM-Linux-gcc 交叉编译工具链就是为针对 ARM 架构的 Linux 系统开发软件而设计的工具链
# 2、交叉编译工具链安装
## 环境确认

在终端输入`arm-linux-gcc -v`
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/207d583b8d6a401abbb295edbefbd484.png)
说明此时环境下没有安装交叉编译工具链
## 下载交叉编译工具链
进入以下网址下载交叉编译工具链，文件后缀名为 .tar.bc2 格式。
[ARM-Linux-gcc 交叉编译工具链下载网址](https://developer.arm.com/downloads/-/gnu-rm)
> Ubuntu 源中已经有了交叉编译工具链，因此使用 apt-get install 命令即可直接安装，缺点是工具链的版本不能确定，和源中提供的版本有关。
![交叉编译工具链](https://i-blog.csdnimg.cn/direct/e498bd095b33498dbf0410a79a8acbd0.png)
## 安装交叉编译工具链
- 将交叉编译工具链从 Windows 移动至 Ubuntu 系统（或者保证 Ubuntu 可以联网，直接通过 Ubuntu 系统中的浏览器下载）
- 解压文件：在终端输入`sudo tar -jxvf 文件路径 -C解压到的文件夹`等待解压完成（一般放在用户文件夹下）
arm-linux-gcc 位于解压后的文件夹下的 bin 目录下
![gcc 文件](https://i-blog.csdnimg.cn/direct/efd5847e717c4c4683a6d3b9a66610b1.png)
## 修改环境变量

- 在终端输入 `arm-none-eabi-gcc -v`仍然显示找不到命令，想要像 gcc 命令一样在任何位置都能使用，就要将其文件位置添加至环境变量
![无法调用](https://i-blog.csdnimg.cn/direct/9dd1643eb75242778ff5e3dea401ecb0.png)
- 终端输入`sudo vim /etc/environment`打开环境变量配置文件，模仿文件中已有格式将 arm-none-eabi-gcc 所在路径添加到环境变量，保存文件
![修改环境变量](https://i-blog.csdnimg.cn/direct/de2061d641b8408490eba390c0da4209.png)
> 此处提前对 bin 的上一级文件夹进行了重命名，重命名为 arm-linux-gcc ，命令为`mv 当前名称 重命名后名称`
- 终端输入`source /etc/environment`使环境配置生效，再次输入`arm-none-eabi-gcc -v`查看编译器信息
![交叉编译链信息](https://i-blog.csdnimg.cn/direct/abd94e57802b4ef5827c269d90de4d47.png)
# 补充
当前交叉编译命令较长，为了方便使用，可以为 bin 文件夹下文件创建链接，使用更加方便，`ln -s 指向文件 链接文件`
![创建链接文件](https://i-blog.csdnimg.cn/direct/d0a1f016a1654f2485bfaa324abe94f1.png)
![链接文件展示](https://i-blog.csdnimg.cn/direct/f88fdfb4c03b46fd8953e92d583270cc.png)

 >参考文献：[手把手教你如何安装交叉编译工具链-韦东山](https://www.bilibili.com/video/BV1rT4y1w7ZS/?spm_id_from=333.999.0.0&vd_source=6a6c56e434d9639407c80ef57b3873ae)

