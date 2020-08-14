# 动态库编译

## 这几个文件编译成动态库libdynamic.so
```
g++ dynamic_a.cpp dynamic_b.cpp dynamic_c.cpp -fPIC -shared -o libdynamic.so
```

参数解释
```$xslt
-shared：该选项指定生成动态连接库
-fPIC：表示编译为位置独立的代码，不用此选项的话编译后的代码是位置相关的所以动态载入时是通过代码拷贝的方式来满足不同进程的需要，
而不能达到真正代码段共享的目的。


```
## 生成可执行文件
```$xslt
将main.cpp与libdynamic.so链接成一个可执行文件main
g++ main.cpp -L. -ldynamic -o main

-L.：表示要连接的库在当前目录中
-ldynamic：编译器查找动态连接库时有隐含的命名规则，
即在给出的名字前面加上lib，后面加上.so来确定库的名称
libxxxx.so
```

```$xslt

测试可执行程序main是否已经链接的动态库libdynamic.so，如果列出了libdynamic.so，那么就说明正常链接了
ldd main
```
```shell script
ldd main:
	linux-vdso.so.1 =>  (0x00007ffda8d7f000)
#标记位置
    libdynamic.so => ./libdynamic.so (0x00007ff410851000)
	libstdc++.so.6 => /usr/lib64/libstdc++.so.6 (0x00007ff41054a000)
	libm.so.6 => /usr/lib64/libm.so.6 (0x00007ff410248000)
	libgcc_s.so.1 => /usr/lib64/libgcc_s.so.1 (0x00007ff410032000)
	libc.so.6 => /usr/lib64/libc.so.6 (0x00007ff40fc65000)
	/lib64/ld-linux-x86-64.so.2 (0x00007ff410a53000)
---------------------------------------------
如果出现，则表名 ld 没有搜到当前库，ld 和 g++搜索位置不同
libdynamic.so => not found
ld提示找不到库文件，而库文件就在当前目录中。
 
链接器ld默认的目录是/lib和/usr/lib，如果放在其他路径也可以，需要让ld知道库文件在哪里
```
## ld not found 解决办法
```
1、编辑/etc/ld.so.conf文件，在新的一行中加入库文件所在目录；比如笔者应添加：
  /home/neu/code/Dynamic_library
    
   sudo ldconfig     # 目的是用ldconfig加载，以更新/etc/ld.so.cache文件

2、在/etc/ld.so.conf.d/目录下新建任何以.conf为后缀的文件，在该文件中加入库文件所在的目录
  sudo ldconfig  #ld.so.cache的更新是递增式的，就像PATH系统环境变量一样，不是从头重新建立，而是向上累加。除非重新开机，才是从零开始建立ld.so.cache文件。

3、在bashrc或profile文件里用LD_LIBRARY_PATH定义，然后用source加载
    也可以 export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:lujing
    补充:
        LD_RUN_PATH在链接时使用
        LD_LIBRARY_PATH在执行时使用
4、直接采用在编译链接的时候告诉系统你的库在什么地方
    g++ -Wl,-rpath=<path:path>
        多个路径是使用冒号隔开，<=>或者换成<,>
    
    readelf -dl  <输出的可执行文件>
    搜索 RPATH 可以看到
    0x000000000000000f (RPATH)              Library rpath: [/root/dynamic]

    -Wl
    这个是gcc的参数，表示编译器将后面的参数传递给链接器ld。W 大写
    -rpath  使用 man ld命令查看手册，找到了-rpath的讲解
      1. 添加一个文件夹作为运行时库的搜索路径。在将ELF可执行文件与共享对象链接时使用此选项；
      2. 在链接时，一些动态库明确的链接了其他动态库， 则-rpath选项也可用于定位这些链接的动态库（没太理解这个）；
      3. 在运行链接时，会优先搜索-rpath的路径，再去搜索LD_RUN_PATH的路径。

5、直接将库放入/lib 或者 /usr/lib 路径中

这四种方法的优先顺序：
    4 > 3 > 2 == 1 == 5
```

# 静态库编译

### 生成中间 .o 文件
```shell script
g++ -c dynamic_a.cpp dynamic_b.cpp dynamic_c.cpp  
```
### 生成静态库
```shell script
ar cr libstatic.a dynamic_a.o dynamic_b.o dynamic_c.o  # cr标志告诉ar将object文件封装(archive)

nm -s libstatic.a  查看 .a 文件内容
```

### 生成可执行文件
```shell script
g++ main.cpp -lstatic -L. -static -o main2

#这里的-static选项是告诉编译器,static是静态库也可以用
#或者
g++ main.cpp -lstatic -L.  -o main2 
```

### 参考
```shell script
https://www.cnblogs.com/zjiaxing/p/5557629.html
```

# 静动混合编译

```shell script

g++ main.cpp -lpthread /usr/lib64/libboost_thread.a /usr/lib64/libboost_system.a

# 静态库直接写路径. 动态前面加-l  这样也可以实现.
  
或者
g++  main.cpp -lrt -Wl,-Bstatic -lboost_system -lboost_thread -Wl,-Bdynamic
# 后面加  -Wl,-Bdynamic   其它的库才能默认动态链接
# -Wl,-Bstatic指示跟在后面的-lxxx选项链接的都是静态库
# -Wl,-Bdynamic指示跟在后面的-lxxx选项链接的都是动态库
```