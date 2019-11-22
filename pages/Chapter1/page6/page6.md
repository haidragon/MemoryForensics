# volatility 简单插件编写
在/volatility/plugins/linux/目录下创建一个任意名字的文件，代码如下：
```
import volatility.plugins.linux.common as linux_common
class linux_test_plugin(linux_common.AbstractLinuxCommand):
    def __init__(self, config, *args, **kwargs):
        linux_common.AbstractLinuxCommand.__init__(self, config, *args, **kwargs)
    def calculate(self):
        linux_common.set_plugin_members(self)
        print 'linux_test_plugin'
```
把参数修改成linux_test_plugin下断运行，如图1所示。



