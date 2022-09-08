shell就是语法解析器，通过shell脚本去执行一系列命令

1.shell的基本语法

hello.sh 文件

```bash
#！/bin/bash   指定默认解析器是bash
echo "hello word"   //打印echo
bash /root/shellLearn/hello.sh  递归调用hello.sh
```

2.执行shell 文件的命令

```bash
bash hello.sh 创建子shell,在子shell里面执行
sh hello.sh 创建子shell,在子shell里面执行，在子shell设置的环境变量在父shell里面可能不变
./hello.sh  需要设置hello.sh权限 chomd +x hello.sh
source ./hello..sh 不创建子shell
```

3.变量

3.1系统预定义变量

1）常用系统变量

$HOME $PWE $SHELL $USER

2）案例实操

查看系统变量值