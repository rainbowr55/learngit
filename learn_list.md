## GestureOverlayView 使用 ##

# 反编译apk #
## jadx-gui查看源码 ##

[ jadx github 地址](https://github.com/skylot/jadx/ " jadx github 地址") github上已经给出详细的用法，这里就写下windows下的用法：

1. 将源码down下来
2. cd到jadx目录下 如果计算机已经配好gradle的环境变量则直接执行 gradle dist命令
3. 命令执行问之后就进入生产的build目录\build\jadx\bin的文件中 运行jadx-gui.bat文件选择想要反编译的应用的classes.dex文件，即可查看源代码，也可以执行通过执行**`jadx -d out classes.dex`**命令查看
4. 通过jadx-gui 的File菜单中的Save as gradle project可以将所有的java保存下来

# apktool #
apktool 是google提供的apk编译工具

该命令用于进行反编译apk文件，一般用法为

**`apktool d <file.apk> <dir>`**

**`<file.apk>`**代表了要反编译的apk文件的路径，最好写绝对路径，比如C:\MusicPlayer.apk

**`<dir>`**代表了反编译后的文件的存储位置，比如C:\MusicPlayer

如果你给定的<dir>已经存在，那么输入完该命令后会提示你，并且无法执行，需要你重新修改命令加入-f指令
    
    apktool d –f <file.apk> <dir>

这样就会强行覆盖已经存在的文件
    
    apktool d –f <file.apk>
默认在当前目录生成file目录
