# 第五节 linux部分profile源码分析
这里我们分析linux的profile加载整个过程，先从入口的registry.PluginImporter()开始，进入看到如下代码：
```
    def __init__(self):
        """Gathers all the plugins from config.PLUGINS
           Determines their namespaces and maintains a dictionary of modules to filepaths
           Then imports all modules found
        """
        self.modnames = {}

        # Handle additional plugins
        for path in plugins.__path__:
            path = os.path.abspath(path)

            for relfile in self.walkzip(path):
                module_path, ext = os.path.splitext(relfile)
                namespace = ".".join(['volatility.plugins'] + [ x for x in module_path.split(os.path.sep) if x ])
                #Lose the extension for the module name
                if ext in [".py", ".pyc", ".pyo"]:
                    filepath = os.path.join(path, relfile)
                    # Handle Init files
                    initstr = '.__init__'
                    if namespace.endswith(initstr):
                        self.modnames[namespace[:-len(initstr)]] = filepath
                    else:
                        self.modnames[namespace] = filepath

        self.run_imports()
```
这个意思很明确就是把指定目录下的所有py文件拉起来导入起来，我们下断看看如图式1所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page5/images/1.png)
果然全是，那我们进入linux的源码，打开volatility/plugins/overlays/linux/linux.py文件，有几行代码是加载文件就会执行的，代码如下：
```

################################
# Track down the zip files
# Push them through the factory
# Check whether ProfileModifications will work

new_classes = []

for path in set(volatility.plugins.__path__):
    for path, _, files in os.walk(path):
        for fn in files:
            if zipfile.is_zipfile(os.path.join(path, fn)):
                new_classes.append(LinuxProfileFactory(zipfile.ZipFile(os.path.join(path, fn))))

################################

# really 'file' but don't want to mess with python's version
class linux_file(obj.CType):

```
这个代码我们看到过，他主要就是把对应的profile名称做一个类初始化，下断看下堆栈如图2所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page5/images/2.png)
profile加载就是这么个过程。
