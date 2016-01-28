## Linux提高

#### vim介绍
* _vim_是类_vi_的文本编辑器，可看成是_vi_的加强升级版，最明显的区别就是代码高亮
* CentOS安装_vim_指令为`yum install -y vim-enhanced`
* `vim +n filename`，n是一个数字，会在打开文件的时候将光标定位到第n行，在_vim_中使用`:set nu`可显示行号

#### vim一般模式下光标移动
| 按键 | 作用 |
|:----:|:----|
| h或左方向键 | 光标左移一个字符 |
| l或右方向键 | 光标右移一个字符 |
| k或上方向键 | 光标上移一个字符 |
| j或下方向键 | 光标下移一个字符 |
| Ctrl+f或PageUp | 屏幕前翻一页 |
| Ctrl+b或PageDown | 屏幕后翻一页 |
| 数字0或shift+6 | 移动到本行行首（后者移动到行首第一个非空字符）|
| shift+4 | 移动到本行行尾 |
| gg | 移动到首行 |
| G | 移动到尾行|
| nG（n是数字） | 移动到第n行 |

#### vim一般模式下复制剪切粘贴
| 按键 | 作用 |
|:----: |:---- |
| x,X | x向后删除一个字符，X向前删除一个字符 |
| nx | 向后删除n个字符 |
| dd | 删除/剪切光标所在行 |
| ndd（n为数字） | 删除/剪切光标所在行起向下共n行 |
| yy | 复制光标所在行 |
| nyy | 从光标所在行起向下复制n行 |
| p | 从光标所在行向下粘贴已复制或剪切的内容 |
| P | 从光标所在行向下粘贴已复制或剪切的内容 |
| u | 还原上一步操作（最多还原50步） |
| Ctrl+r | 撤销上一次还原 |
| v | 按v后进入可选模式，选中指定字符后可进行剪切 |
| V | 按行选择 |

#### vim编辑模式
进入编辑模式可修改字符，一般模式下按键（`i`，`I`，`a`，`A`，`o`，`O`，`r`，`R`）即可进入编辑模式  

| 按键 | 作用 |
|:----:|:----|
| i | 在当前字符前插入 |
| I | 在光标所在行行首插入 |
| a | 在当前字符后插入 |
| A | 在当前行行尾插入 |
| o | 在当前行下一行插入新的一行 |
| O | 在当前行上一行插入新的一行 |
| r | 替换光标下的字符（一次性，不会进入编辑模式） |
| R | 用输入的字符替换原有字符 |

#### vim命令行模式
一般模式下输入`:`或`/`即可进入命令模式，该模式下，可以进行搜索，替换，保存，显示行号，退出等

| 按键 | 作用 |
|:----:|:----|
| /word | 向光标后寻找字符串word，按n指向下一个 |
| ?word | 向光标前寻找字符串word，按n指向下一个 |
| :n1,n2s/word1/word2/g | 在n1，n2行之间查找word1并替换为word2，不加g则只替换每行第一个word1 |
| :1,$s/word1/word2/g | 替换文档中所有word1为word2，`1，$`可用`%`代替 |
| :w | 保存 |
| :q | 退出vim |
| :w! | 强制保存，root用户下即使文本只读也可保存 |
| :q! | 强制退出，所有改动不生效 |
| :wq | 保存并退出 |
| :set nonu | 不显示行号 |

#### gzip讲解
* `gzip [-#] filename`用于压缩文件，不保留源文件，__#__是压缩等级，1最差，9最好，默认6，不支持压缩目录，生成压缩文件后缀名`.gz`
* `gzip -d`可以解压压缩文件
* `gunzip`也可以解压，不加参数
* `zcat`可查看`.gz`的文本文件内容

#### bzip2讲解
* `bzip2 [-dz] filename`  
`-d`解压缩  
`-z`压缩，可不加  
* __bzip2__不能压缩目录，压缩不保留源文件  
* `bzcat`可查看`.bz2`的文本文件内容

#### zip和unzip
* `zip filename.zip filename`  
用于压缩文件，先跟压缩后文件名，再跟要压缩的文件名或目录，压缩保留源文件
* `zip -r`  
用于级联压缩，_zip_默认不压缩二级目录下文件，压缩目录时需要加参数`r`
* `unzip filename.zip`用于解压,参数`-d`可指定解压路径

#### xz压缩和解压缩
* `xz [-dz] filename`  
`-d`解压缩，解压时`-f`可强行覆盖已存在文件  
`-z`压缩，可不加
* xz不能压缩目录，压缩不保留源文件
* `xzcat`查看`.xz`的文本文件内容

#### tar打包工具详解
* 语法`tar [-zjxcvfpP] filename`  
`-z`同时用_gzip_压缩  
`-j`同时用_bzip2_压缩  
`-x`解包或解压缩  
`-t`查看_tar_包里的文件  
`-c`建立一个_tar_包或者压缩包  
`-v`可视化  
`-f`后面跟文件名，压缩时`-f filename`意为压缩后的文件名为_filename_，解压时跟`-f filename`，意为解压_filename_。如果多参数组合，`-f`需写到最后  
`-p`使用原文件的属性（不常用）  
`-P`可使用绝对路径（不常用）  
`--exclude filename`打包或压缩时，不将_filename_包括在内（不常用）  

#### tar打包和压缩并用
* `tar -zcvf filename.tar.gz filename/directory`，打包并使用_gzip_压缩  
`tar -zxvf filename.tar.gz`用于解压
* `tar -jcvf filename.tar.bz2 filename/directory`，打包并使用_bzip2_压缩  
`tar -jxvf filename.tar.bz2`用于解压
* `tar -Jcvf filename.tar.xz filename/directory`，打包并使用_xz_压缩  
`tar -Jxvf filename.tar.xz`用于解压
* `tar -tf 包名`可查看压缩包文件目录，低版本tar在查看`tar.xz`时需要加上参数`J`

#### rpm安装和卸载
* `rpm -ivh filename.rpm`用于安装_rpm_包  
`-i`安装  
`-v`可视化  
`-h`显示安装进度  
常用附带参数：  
`--force`强制安装，即使覆盖其他包或文件也要安装  
`--nodeps`忽略依赖安装
* `rpm -Uvh package`升级_rpm_包，`-U`用于升级
* `rpm -e package`卸载_rpm_包，输入包名而不是完整文件名

#### rpm查询
* `rpm -q package`查询一个包是否安装
* `rpm -qa`查询当前系统所有安装过的_rpm_包
* `rpm -qi package`得到一个已安装过_rpm_包的相关信息
* `rpm -ql package`列出一个_rpm_包安装的文件
* `rpm -qf 文件绝对路径`列出某个文件属于哪个_rpm_包

#### yum工具详解
* `yum list`列出所有可用的_rpm_包，`yum list |grep package`可用来过滤列出内容，起到搜索的效果，结果中有`@`的为已安装的包
* `yum search [关键词]`搜索_rpm_包
* `yum install [-y] [package]`安装_rpm_包，加`-y`可跳过询问用户是否安装
* `yum remove [-y] [package]`卸载_rpm_包，不建议加`-y`，可能会因为破坏依赖关系影响别的组件正常工作
* `yum update [-y] [package]`升级_rpm_包
* `yum grouplist`列出软件包组  
`yum groupinstall "套件名"`安装套件（软件包组）  
`yum groupremove`卸载套件

#### 搭建本地yum仓库
1. `mount /dev/cdrom /mnt`挂载_cd_
2. `cp -r /etc/yum.repos.d /etc/yum.repos.d.bak`备份_repo_  
`rm -rf /etc/yum.repos.d/`删除旧的源信息  
或者只删除`/etc/yum.repos.d/CentOS-Base.repo`
3. 创建新repo文件或者修改`CentOS-Media.repo`，内容如下所示  
```
[dvd]
name=install dvd
baseurl=file:///mnt
enabled=1
gpgcheck=0
```
第一行`[dvd]`为本地软件源的名字，会显示在`yum list`列表右侧  
第二行是该源的名字标识，可不加  
第三行路径，第四行启用，第五行关闭_gpg_验证

#### yum下载rpm包到本地
* 需要安装`yum-plugin-downloadonly`，6.7版本中已经不需要额外安装
* `yum install package -y --downloadonly --downloaddir=/tmp`可以将_rpm_下载到指定目录

#### 源码编译安装
1. `./configure`可加上相应选项定制功能，会检测编译所需的库是否完整，检测通过会生成__Makefile__文件
2. `make`会根据__Makefile__中预设参数进行编译
3. `make install`进行安装
4. 注意的是，并非所有软件都是上述步骤，需先阅读源码包中_install_或者_readme_之类说明文件
5. 查看某一步骤是否成功，可用`echo $？`，返回值若为0则成功
