# 第四节 volatility 源码整体架构分析下
上一节我们看到了插件的初始化操作，这里同样以插件linux_pslist(源码文件为pslist)开讲，构造函数的第一个参数就是为配置文件，这个文件保存在哪呢，我们简单看linux_pslist类中没有这个字段啊，如图1所示。
我们继续看他调用的父类构造函数，如图2所示。
发现原来在再上面，如图2所示。
下断动态调试下，看下调用堆栈，分析没错如图2所示。
那插件是怎么调用起来的呢，我们回到最上面的入口，看如下代码：
```
 try:
        if module in cmds.keys():
            command = cmds[module](config)

            ## Register the help cb from the command itself
            config.set_help_hook(obj.Curry(command_help, command))
            config.parse_options()

            if not config.LOCATION:
                debug.error("Please specify a location (-l) or filename (-f)")

            command.execute()
```
我们发现它是找到对应的插件类执行execute函数，那这个execute函数不要想了肯定是一个父类的函数，我们翻command源码看看，如图3所示。
发现execute函数最终调用的是上面没有实现的calculate函数(虚拟函数),到这里肯定能猜到所有插件的入口就是calculate函数的实现，下断看下调用堆栈如图4所示。
到这里并没有结束，还有一个很重要的事情就是怎么获取profile的，猜想这个一般应该放在配置文件中，看看有没有如图5所示。
发现什么也没有，那放到哪呢，没什么好的分析思路，继续从插件入口下手，一开始就调用一个set_plugin_members函数，进去看看，调用了 obj_ref.addr_space = utils.load_as(obj_ref._config)，继续进入，又发现了熟悉的地方是遍历BaseAddressSpace子类操作，如图6所示。
继续看BaseAddressSpace类发现了一行代码self._set_profile(config.PROFILE),如图7所示。
但是还是有点迷，没关系下断动态调试下，发现就是set我们传进去的如图8所示。
返回往上看看所有的子类，发现是一个镜像操作相关的子类，如图9所示。
现在又深入了一点，那么那个传的LinuxUbuntu1404x64是什么个流程呢，我们继续看_set_profile函数如图10所示。
又是一个获取子类的操作，下断看看有哪些子类，如图11所示。
原来每个profile又是一个类，这里我又有疑问了这个类怎么生成的，因为LinuxUbuntu1404x64名字是我自己取的啊，好了又是到了怎么找这个类的问题，好办下断到找到的地方看看哪个类如图12所示。
发现FileAddressSpace类，进入standard源码看看，没什么线索，不要找了思路错了(我找了好久),后面返回到最前面的main函数执行了一句registry.PluginImporter()，然后一路跟到了LinuxProfileFactory，原来他是一个工厂函数遍历所有的profile获取文件名创建一个类出来，下断到这个函数看堆栈如图13所示。

