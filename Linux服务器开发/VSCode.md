# 1 环境变量

> Ctrl + shift + p调出Command Palette，即用于执行命令的输入框
>
> cmake -G "MinGW Makefiles" ..
>
> mingw32-make.exe
>
> ```json
> /*
> ${workspaceFolder}：项目文件夹在VSCode中打开的路径
> ${file}：当前打开（激活）的文件
> ${fileBasename}：当前打开文件的名称
> ${fileBasenameNoExtension}：当前打开的文件名称，不带扩展名
> ${fileExtname}：当前打开文件的扩展名
> ${fileDirname}：当前打开文件的文件夹名称
> ${relativefile}：相对于{workspaceFolder}的文件路径
> */
> ```

# 2 单文件编译时

## 2.1 tasks.json

```json
{
    // 用于编译文件的配置信息，用于生成可执行文件
    "tasks": [
        {
            "type": "shell", //执行编译任务的命令方式为在 “windows powershell"中执行
            "label": "C/C++: g++.exe 生成活动文件", //任务名称
            "command": "g++", //命令名称
            "args": [//命令紧跟的参数
                "-g",
                "${workspaceFolder}//*.cpp", //当前文件名
                "-I",
                "${fileDirname}",
                "-o",  //指定编译后的文件名
                "${fileDirname}\\${fileBasenameNoExtension}.exe"
            ],
            "options": {
                "cwd": "${workspaceFolder}" //设置工作目录
            },
            "problemMatcher": [
                "$gcc" //使用gcc捕获错误
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "detail": "调试器生成的任务。"
        }
    ],
    "version": "2.0.0"
}
```

## 2.2 launch.json

```json
{
    // 使用 IntelliSense 了解相关属性。
    // 用于调试的配置信息 
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
    
        {
            "name": "(gdb) 启动", // 调试方式的名称，在调试窗口下拉列表可见
            "type": "cppdbg",   //调试类型，node，java cppdbg等
            "request": "launch", //直接启动方式，另一种方式是attach
            "program": "${fileDirname}\\${fileBasenameNoExtension}.exe", //启动的可执行程序路径，要与task编译生成的可执行程序路径一致
            "args": [], //程序启动时传递的参数
            "stopAtEntry": false, //选择是否启动前暂停
            "cwd": "${workspaceFolder}", //程序启动时的根目录配置
            "environment": [], //程序启动时传递的环境变量
            "externalConsole": false, // 通过VS Code外部的集成终端显示运行结果
            "MIMode": "gdb",
            "miDebuggerPath": "D:\\Program Files\\mingw64\\bin\\gdb.exe", // MinGW编译器下gdb的目录
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
            "preLaunchTask": "C/C++: g++.exe 生成活动文件" //在启动前需要执行的任务（通常是先编译生成可执行文件后启动），参数为task.json中的任务名称
        }
    ]
}
```

## 2.3 c_cpp_properties.json

```json
{
    "configurations": [
        {
            "name": "Win32",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "defines": [
                "_DEBUG",
                "UNICODE",
                "_UNICODE"
            ],
            "windowsSdkVersion": "10.0.17763.0",
            "compilerPath": "D:\\Program Files\\mingw64\\bin\\g++.exe",
            "cStandard": "c17",
            "cppStandard": "c++17",
            "intelliSenseMode": "${default}"
        }
    ],
    "version": 4
}
```

# 3 cmake

```json
{   
    "version": "2.0.0",
    "options": {
        "cwd": "${workspaceFolder}/build"
    },
    "tasks": [
        {
            "type": "shell",
            "label": "cmake",
            "command": "cmake",
            "args": [
                ".."
            ],
        },
        {
            "label": "make",
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "command": "mingw32-make",
            "args": [

            ],
        },
        {
            "label": "Build",
            "dependsOn":[
                "cmake",
                "make"
            ]
        }
    ],

}
```

