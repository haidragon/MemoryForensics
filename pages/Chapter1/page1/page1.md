# 第一节linux内存取证工作流程
	第一步先安装一些工具有：
```
pip install distorm3
pip install yara
pip install pycrypto
pip install pil
pip install openpyxl
pip install ujson
```
	第二步下载Volatility源码，输入如下命令：
```
git clone https://github.com/volatilityfoundation/volatility.git
```
	第三步制作linux的profile，制作profile就是把system.map与moudle.dwarf打包成zip放到到volatility/volatility/plugins/overlays/linux/路径下，命令如下：
```
sudo zip volatility/volatility/plugins/overlays/linux/Ubuntu1404.zip volatility/tools/linux/module.dwarf /boot/System.map-'uname -r'(Ubuntu1404是对应的版本号)。
```
	在这运行之前要先创建module.dwarf，去volatility/tools/linux/目录make即可。
	检查是否存在制作的profile，输入命令：
```
python vol.py --info|grep Linux
```
	能够看到如图1所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page1/images/LinuxUbuntu1404x64.png)
图1
	如果没有可以手动解压出来(Volatility源码中有自动解压操作),Volatility环境已经搭建完成，现在我们要做的是内存镜像提取，这里我用的一个开源的驱动，输入命令：
```
git clone https://github.com/504ensicsLabs/LiME.git
```
	然后编译，用法作者已经有写,LiME提供了3种获取内存格式:raw、padded和lime。raw格式是指获取全部物理内存 段的物理内存;padded 是指除获取物理内存段物理内存外，其他段以0填充;lime除获取物理内存外，每段内存段前附加了格式为lime_header结构体的数据。例如这里：
```
sudo insmod ./lime-4.4.0-142-generic.ko "path=./mem.lime format=lime"
```
	最后我们就开始分析了，可以查看Volatility运行哪些插件，运行命令：
```
python vol.py --info | grep -i linux_
```
	输入要查看的信息，比如linux_pslist,运行命令：
```
python vol.py -f ../src/mem.lime --profile=LinuxUbuntu1404x64 linux_pslist
```
	如图2所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page1/images/pslist.png)
	对于不同的插件，Volatility取证得出的结果可以以不同的方式输出。输出方式包含：
```
    text 以表的形式文本输出
    dot 以点阵图形式输出
    html
    json 可用于不同的api之间数据交换
    sqlite 存到数据库中
    quick 以|符号分隔地输出文本
    xlsx 以excel文件输出
```

	一般我们可以-h命令来获取不同插件的输出格式。如：
```
python vol.py linux_pstree -h
Module Output Options: dot, greptext, html, json, sqlite, text, xlsx
---------------------------------
Module PSTree
---------------------------------
Print process list as a tree
```
	以json为例输入如下命令：
```
python vol.py -f ../src/mem.lime --profile=LinuxUbuntu1404x64 linux_pslist --output=json --output-file=pstree1.json
```
	结果如图3所示：
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page1/images/json.png)
图3
