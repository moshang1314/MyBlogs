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

```python
import torch
import torch.nn as nn
import math
from basic import *
import torch.nn.functional as F

class DepthPre(nn.Module):
    def __init__(self):
        super(DNet, self).__init__()
        self.conv_lidar = nn.Sequential(Convbnrelu(1, 32), Convbnrelu(32, 64))
        self.conv_img = nn.Sequential(Convbnrelu(3, 32), Convbnrelu(32, 64))
        self.pre_lidar = hourglass_pre()
        self.pre_img = hourglass_img()
        self.hourglass_40 = hourglass_40()
        self.hourglass_41 = hourglass_41()
        self.re_4 = nn.Sequential(self._make_layers(64, 32, blocks=1, stride=1), nn.Conv2d(32, 1, kernel_size=3, stride=1, padding=1, bias=False))
    def forward(self, img, lidar):
        c0 = self.conv_lidar(lidar)
        # 1/16, 1/8, 1/4, 1/2, 1/1
        l_10, l_20, l_30, l_40, l_50 = self.pre_lidar(c0)
        i_0 = self.conv_img(img)
        # 1/4, 1/2, 1/1
        img_1, img_2, img_3 = self.pre_img(i_0)
        c_41, c_42, c_43 = self.hourglass_40(l_30, l_20, l_10, img_1)
        c_44, c_45, c_46 = self.hourglass_41(c_43, c_41, c_42)
        result_4 = self.re_4(c_46)
        return (l_40, l_50), (img_2, img_3), (c_44, c_45, c_46), result_4
        
class DNet(nn.Module):
    def __init__(self):
        super(DNet, self).__init__()
        self.backbone = DepthPre()
        
        """ 1/1分支 """
        # self.changeChannel_3 = Convbnrelu(128, 64, kernel_size=1, stride=1, padding=0)
        self.hourglass_10 = hourglass_10()
        self.hourglass_11 = hourglass_11()
        self.re_1 = nn.Sequential(self._make_layers(64, 32, blocks=1, stride=1),
                                  nn.Conv2d(32, 1, kernel_size=3, stride=1, padding=1, bias=False))
        self.init_weights()

    def forward(self, img, lidar):
        # valid_mask = torch.where(lidar > 0, torch.full_like(lidar, 1.0), torch.full_like(lidar, 0.0))
        lidar_f, img_f, li_f, result_4 = self.backbone()
        """ 1/2 branch """
        # c_20 = torch.cat((l_40, img_2), dim=1)
        # c_20 = self.changeChannel_2(c_20)
        # 1/16, 1/8, 1/4, 1/2
        c_21, c_22, c_23, c_24 = self.hourglass_20(lidar_f[0], li_f[0], li_f[1], li_f[2], img_f[0])
        # 1/16, 1/8, 1/4, 1/2
        c_25, c_26, c_27, c_28 = self.hourglass_21(c_24, c_21, c_22, c_23)
        c_29 = self.re_2(c_28)
        result_4_up = F.interpolate(input=result_4, scale_factor=2, mode='bilinear')
        result_2 = result_4_up + c_29

        """1/1 branch"""
        # c_10 = torch.cat((l_50, img_3), dim=1)
        # c_10 = self.changeChannel_3(c_10)
        # 1/16, 1/8, 1/4, 1/2, 1/1
        c_11, c_12, c_13, c_14, c_15 = self.hourglass_10(lidar_f[1], c_25, c_26, c_27, c_28, img_f[1])
        c_16 = self.hourglass_11(c_15, c_11, c_12, c_13, c_14)
        c_17 = self.re_1(c_16)
        result_2_up = F.interpolate(input=result_2, scale_factor=2, mode='bilinear')
        result = result_2_up + c_17

        return result_4, result_2, result

    def init_weights(self):
        # Initialize filters with Gaussian random weights
        for m in self.modules():
            if isinstance(m, nn.Conv2d):
                n = m.kernel_size[0] * m.kernel_size[1] * m.out_channels
                m.weight.data.normal_(0, math.sqrt(2. / n))
                if m.bias is not None:
                    m.bias.data.zero_()
            elif isinstance(m, nn.ConvTranspose2d):
                n = m.kernel_size[0] * m.kernel_size[1] * m.in_channels
                m.weight.data.normal_(0, math.sqrt(2. / n))
                if m.bias is not None:
                    m.bias.data.zero_()
            elif isinstance(m, nn.BatchNorm2d):
                m.weight.data.fill_(1)
                m.bias.data.zero_()

    def _make_layers(self, inplanes, planes, blocks, stride=1):
        layers = [BasicBlock(inplanes, planes, stride=stride)]
        for i in range(1, blocks):
            layers.append(BasicBlock(planes, planes))
        return nn.Sequential(*layers)
```

