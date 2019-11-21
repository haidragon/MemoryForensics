# 第一节linux内存取证工作流程
	第一步先安装一个必要的工具有：
	pip install distorm3
	pip install yara
	pip install pycrypto
	pip install pil
	pip install openpyxl
	pip install ujson
	第二步下载Volatility源码，git clone https://github.com/volatilityfoundation/volatility.git。
	第三步制作linux的profile，制作profile就是把system.map与moudle.dwarf打包成zip放到到volatility/volatility/plugins/overlays/linux/路径下，命令为sudo zip volatility/volatility/plugins/overlays/linux/Ubuntu1404.zip volatility/tools/linux/module.dwarf /boot/System.map-'uname -r'(Ubuntu1404是对应的版本号)。在这运行之前要先创建module.dwarf，去volatility/tools/linux/目录make即可。
检查是否存在制作的profile，输入命令python vol.py --info|grep Linux，能够看到如图1所示。
<center>
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page1/images/LinuxUbuntu1404x64.png)
</center>
<center><font color=bule size=10>图1</font></center>

