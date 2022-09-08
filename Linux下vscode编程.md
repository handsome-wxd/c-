# 前言

学习linux下通过vscode进行c++编程的学习总结，知识内容包括使用g++,cmake编译，已经最后在vscode上对代码进行调试

# 环境搭建

用的是虚拟机 ubantu18

**需要工具**

gdb(调试工具)，g++(编译工具，c与c++都可以编译)，gcc(编译工具，用来编译c)

```bash
#直接在终端输入，将三个都安上
sudo apt-get install build-essential gdb
#测试是否安装成功
g++ --version
gdb --version
gcc --version
```

**安装vscode**

软件链接,ubantu下载.deb的

[vsocde]: https://code.visualstudio.com/

安装可以直接在目录中点击软件进行安装也可以使用命令  sudo dpkg  包名

**vscode使用**

```
#直接在终端输入,vscode就打开当前文件夹
code ./
```

vscode必备插件：c/c++，cmake,cmake tool

**cmake的使用**

[使用cmake的理由]: https://blog.csdn.net/qq_27825451/article/details/103392719

**cmake编译工程步骤**

编写CMakeLists.txt，用来指明如何进行编译和链接

在这里CMakeLists.txt文件和main文件放在同一级目录下

```txt
cmake_minimum_required(VERSION 3.0) //cmake最低要求
project(SOLIDER) //设置工程名
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall") //设置一些编译信息 -wall是显示警告
set(CMAKE_BUILD_TYPE Debug) //开启调试模式，没有设置将无法进行调试
include_directories(include) //将include文件加入包含目录中，include文件中放着自己定义的头文件，这样在.cpp文件引用的时候就不用加include的相对路径
add_executable(myexeCmake main.cpp src/Gun.cpp src/Soilder.cpp)//选择要执行的.cpp文件
```

进行编译时，一般采用外部构建方式，在当前文件夹下建一个build文件夹，然后进行此文件夹后执行 cmake ..命令

生成一系列文件 然后执行 make 就可以生成 名为myexeCmake的可执行文件

**cmake对工程进行调试**

需要加入launch.json 和 tasks.json

launch.json创建 点击 run and debug 点击第一个进行配置 然后点击config 

```json
{
            "name": "(gdb) 启动",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/build/myexeCmake",//需要修改 myexeCmake 是通过Cmake编译出的可执行文件
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",//需要修改
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                },
                {
                    "description":  "将反汇编风格设置为 Intel",
                    "text": "-gdb-set disassembly-flavor intel",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "Build",//提前执行Task.json里面的Build任务，这样可以开启自动调试模式，然后在代码中打断点，按F5进行调试，没有这一句，需要先手动make之后在进行调试
            "miDebuggerPath": "/usr/bin/gdb"
        }

    ]
}
```

 tasks.json点击terminal 然后点击configure default build task 然后点击第一个

```json
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "options": {
        "cwd": "${workspaceFolder}/build"
    },
    "tasks": [
        {
            "type": "shell",
            "label": "cmake",
            "command": "cmake",
            "args": [".."]
        },
        {
            "label": "make",
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "command":"make",
            "args": []
        },
        {
            "label": "Build",//Build任务包括 cmake任务 和make任务
            "dependsOrder": "sequence",//指定执行顺序
            "dependsOn":["cmake","make"]
        }
    ]
}
```

**虚拟机和windows设置共享文件夹在，实现文件互传**

![image-20220824234639255](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220824234639255.png)

![image-20220824234919540](C:\Users\18440\AppData\Roaming\Typora\typora-user-images\image-20220824234919540.png)

同步后的文夹 虚拟机中位置在 /mnt/hgfs/fileshare/中