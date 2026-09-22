# ESOPT试用说明

ESOPT是由C++开发的高效建模求解工具，能有效对工程问题进行数学建模求解，其效率超越市面上其他同类产品。

![esopt-pyomo-gams](.\esopt-pyomo-gams.png)

## 安装方法

1. 在英文路径下解压对应平台的压缩包。
2. 安装TinyCC。Windows平台上，解压`tcc.zip`到任意全英文路径下，然后设置环境变量`LIBTINYCC_DIR=<绝对路径/tcc>`；Linux平台上，从`git://repo.or.cz/tinycc.git`拉取源码编译安装。
3. Windows平台使用15.2.0 posix-seh-ucrt-rt_v13 ([official prebuilt 7z](https://github.com/niXman/mingw-builds-binaries/releases/download/15.2.0-rt_v13-rev0/x86_64-15.2.0-release-posix-seh-ucrt-rt_v13-rev0.7z))进行编译。

## 学术申请

ESOPT支持Linux平台和Windows平台使用，在没有许可证的情况下，求解限制了300个变量和约束。

可进行学术申请免费试用完整版一年，请使用下面的命令获取主机的MAC地址，发送到邮箱：juyuntao@ncut.edu.cn进行学术申请。

```bash
Windows:
​```powershell
getmac /v
​```

Linux:
​```bash
ip link show
​```
```

许可证使用方法，通过设置环境变量保存许可证在主机中的路径。

```bash
Linux:
​```bash
export LICENSE_LOCATION=/absolute/path/customer.ini
​```

Windows PowerShell:
​```powershell
$env:LICENSE_LOCATION = 'C:\absolute\path\customer.ini'
​```
```

联系人邮箱：juyuntao@ncut.edu.cn