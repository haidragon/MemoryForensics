# 第三节 volatility 源码整体架构分析上
    这节开始讲volatility整体运行流程，我们这里使用到一个python工具pycharm,先导入volatility，打开vol.py文件，如图1所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page1/images/LinuxUbuntu1404x64.png)
![avatar](https://github.com/haidragon/MemoryForensics/tree/master/pages/Chapter1/page3/images/vol.png)
    图1
    然后设置对应的环境与参数，如图2所示。
![avatar](https://github.com/haidragon/MemoryForensics/tree/master/pages/Chapter1/page3/images/arg.png)
    图2
    现在先不急着调试，先大概看下代码，我们看到入口主要函数是main(),进入main()函数，代码如下：
```
def main():

    # 打印版本信息
    sys.stderr.write("Volatility Foundation Volatility Framework {0}\n".format(constants.VERSION))
    sys.stderr.flush()

    # 初始化debug模块
    debug.setup()
    # Load up modules in case they set config options
    registry.PluginImporter()

    ## Register all register_options for the various classes
    registry.register_global_options(config, addrspace.BaseAddressSpace)
    registry.register_global_options(config, commands.Command)

    if config.INFO:
        print_info()
        sys.exit(0)

    ## Parse all the options now
    config.parse_options(False)
    # Reset the logging level now we know whether debug is set or not
    debug.setup(config.DEBUG)

    module = None
    # 获取插件的字典 插件名：插件体
    cmds = registry.get_plugin_classes(commands.Command, lower = True)
    for m in config.args:
        # 找到参数里的第一个插件名
        if m in cmds.keys():
            # module即为插件名 比如Linux_arp
            module = m
            break

    if not module: # 插件不存在
        config.parse_options()
        debug.error("You must specify something to do (try -h)")

    if module in cmds.keys():
        # 插件存在，则获取插件对象，并将config赋值module这个插件的_config变量
        command = cmds[module](config)
        
        # hook 上help函数
        config.set_help_hook(obj.Curry(command_help, command))
        config.parse_options()

        if not config.LOCATION:
            debug.error("Please specify a location (-l) or filename (-f)")

        # 开始执行插件
        command.execute()
```
大概流程是这样，但是会有很多疑问，比如参数在哪里获取的，插件怎么加载的，这里我先告诉大家，获取参数先不要管，因为有点绕后面会讲，先看插件也就是下面这行代码：
```
cmds = registry.get_plugin_classes(commands.Command, lower = True)
```
好了开始可以调试了，我们先在这行代码上下段然后查看cmds是什么，如图3所示。
![avatar](https://github.com/haidragon/MemoryForensics/tree/master/pages/Chapter1/page3/images/3cmds.png)
发现全是一些类，我们随便找到类比如linux_plthook,进入源码如图4所示。
![avatar](https://github.com/haidragon/MemoryForensics/tree/master/pages/Chapter1/page3/images/linuxplthook.png)
然后翻父类，得到的继承链是linux_plthook -> linux_pslist -> AbstractLinuxCommand -> Command ,而我们要看的那行代码是获取所有的子类，因些也就是每一个插件全是Command的子类，同时每种系统都有一个抽象类，linux为AbstractLinuxCommand，设计思路是如图5所示。
![avatar](https://github.com/haidragon/MemoryForensics/tree/master/pages/Chapter1/page3/images/5linux.png)
如上图可见，vol中的插件以commands的Command类为父类，分别派生出AbstractWindowsCommand类、AbstractMacCommand类以及AbstractLinuxCommand类，这分别代表三个平台插件的基类，这三类为不同平台的插件定义了一些共同的函数，以供具体的插件继承调用，当然也方便了我们自定义插件时复用这些函数。
现在我们知道什么是插件，那他是怎么运行插件的呢，我们看下面代码：
```
...
  module = None
    ## Try to find the first thing that looks like a module name
    cmds = registry.get_plugin_classes(commands.Command, lower = True)
    for m in config.args:
        if m in cmds.keys():
            module = m
            break

    if not module:
        config.parse_options()
        debug.error("You must specify something to do (try -h)")

    try:
        if module in cmds.keys():
            command = cmds[module](config)

            ## Register the help cb from the command itself
            config.set_help_hook(obj.Curry(command_help, command))
            config.parse_options()

            if not config.LOCATION:
                debug.error("Please specify a location (-l) or filename (-f)")

            command.execute()
...
```
通过上面我们知道cmds保存的是所有插件，因为module又是通过config.args获取的，先直接告诉大家module就是输入的参数只匹配一个，我们动态调试下如图5与6所示。
![avatar](https://github.com/haidragon/MemoryForensics/tree/master/pages/Chapter1/page3/images/mod1.png)
![avatar](https://github.com/haidragon/MemoryForensics/tree/master/pages/Chapter1/page3/images/mod2.png)
然后去与cmds里面所有类比较，然后传入配置文件初始化它，怎么初始化的在下一节说。
