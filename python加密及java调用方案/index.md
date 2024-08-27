# Python加密及Java调用方案




<!--more-->
# Python加密及Java调用方案

```
createBy lln
updateTime 2021-09-28
```

## 加密方案

```
官方网站
http://www.pyinstaller.org
官网文档
https://pyinstaller.readthedocs.io/en/stable/
参考博客
https://www.cnblogs.com/weix-l/p/10881373.html
https://zhuanlan.zhihu.com/p/85760495
https://blog.csdn.net/x_w_haohan/article/details/99233561
https://blog.csdn.net/Iv_zzy/article/details/107462210
https://blog.csdn.net/qq_16104903/article/details/90444269
https://blog.csdn.net/qq_41699621/article/details/103596742

```

### Windows

#### pyinstaller打包方法

```
#安装pyinstaller
pip install pyinstaller
```

```
PyInstaller提供了两种把.py文件包成.exe文件的方式：
```

```text
第一种：把由.py文件打包而成的.exe文件及相关文件放在一个目录中。
语法：pyinstaller 应用程序
eg：pyinstaller Hello.py
```

```text
第二种：加上 -F 参数后把制作出的.exe打包成一个独立的.exe格式的可执行文件。
语法：pyinstaller -F 应用程序
eg：pyinstaller -F Hello.py 
```

```
虽然扩平台，但是pyinstaller也只能在当前操作系统中运行，比如你用mac只能打包出mac上的可执行脚本，要是你想打包出windwos电脑上的可执行程序，你就要用windows执行打包命令。

如果你的脚本文件中包含其他脚本，比如hello.py包含自定义脚本(world.py)或是系统脚本(sys.py)：则需要在打包的时候加上其他脚本的路径。通过-p指定第三方包的路径，一条路径对应一个-p
```

```text
eg：pyinstaller -F -p C:\SystemLib\site-packages -p C:\MyLib Hello.py
```

```
执行一次打包命令通常会生成两个目录一个附件，分别是build、dist、和xx.spec。build是编译过程中的中间产物，dist是最终可执行程序目录，spec文件是类似缓存，如果你第二次打包，则需要先把spec删掉，否则第二次打包会受影响。
```

```
参数介绍
-a：不包含编码.在支持Unicode的python版本上默认包含所有的编码
-c：使用控制台子系统执行(默认)(只对Windows有效)
-d：产生debug版本的可执行文件
-i ：指定打包程序使用的图标（icon）文件
-F：打包成可执行程序
-h：查看帮助
-p：添加使用的第三方库路径
-v：查看 PyInstaller 版本
-w：取消控制台显示（默认是显示控制台的）
```

```text
eg:
pyinstaller -F -p C:\SystemLib\site-packages -p C:\MyLib main.py -i C:\image\excel.ico
解释：
打包 main.py 脚本
main.py包含第三方脚本，一个是系统脚本，一个是自定义脚本
设置可执行程序的图标为excel.ico
```

#### 测试python脚本

```
from tkinter import *


def btnClick():
    textLabel['text'] = '我点击了按钮'


root = Tk(className="测试打包");

textLabel = Label(root, text='提示显示', justify=LEFT, padx=10)
textLabel.pack(side=TOP)

btn = Button(root)
btn['text'] = '点击测试'
btn['command'] = btnClick
btn.pack(side=BOTTOM)

mainloop()

```

### Linux

#### 将python编译成so文件进行调用

```
1.安装cython，gcc
sudo apt install python3-dev gcc
2.安装cpython
pip3 install cython
3.将需要转化的python源文件，代码调用python文件，打包加密文件放在同一个目录中
4.在当前目录中执行
python3 setup.py build_ext
5.生成文件 /build/lib.linux-x86_64-3.6/t1.cpython-36m-x86_64-linux-gnu.so*
6.复制 /build/lib.linux-x86_64-3.6/t1.cpython-36m-x86_64-linux-gnu.so* 与
t3 处于同一文件夹
7.删除 t1.py rm t1.py
8.调用t3.py  python t3.py
9.正常输出t1.py 的内容
```

#### 测试python脚本

脚本1 	t1.py

说明：数据脚本，即需要被加密的源文件

```
import datetime

class Today():
    def get_time(self):
        print(datetime.datetime.now())
        
    def say(self):
        print("hello from JC!")

```

脚本2 t2.py  

说明：加密脚本

```
from distutils.core import setup
from Cython.Build import cythonize

setup(ext_modules = cythonize(["t1.py"]))

```

脚本3   t3.py

说明： 进行了对脚本  t1 的调用

```
from mytest import Today

t = Today()
t.get_time()
t.say()

```

## 调用方案

#### 数据对接思路

```
1.python方，
	将核心代码加密为 .so 文件，通过外层函数处理和转换数据，转换后的数据，对加密脚本进行黑箱调用。
2.java方，
	定义python脚本路径，定义JSON数据对象格式，转换JSON数据为数据流，调用python暴露的外层壳函数，使用如下代码进行调用
```



#### 测试调用代码 Java

```
@Component
public class AlgorithmExecuteHelper {

    private static final Logger logger = LoggerFactory.getLogger(AlgorithmExecuteHelper.class);

    private final String pythonPath;

    private final ThreadPoolTaskExecutor taskExecutor;

    public AlgorithmExecuteHelper(@Value("${python-path}") String pythonPath, ThreadPoolTaskExecutor taskExecutor) {
        this.pythonPath = pythonPath;
        this.taskExecutor = taskExecutor;
    }


    public ProcessResultDto execute( File directory,  File enterFile,  DataConsumer consumer){
        Process process = null;
        try {
            process = new ProcessBuilder()
                    .command(pythonPath, enterFile.getName())
                    .directory(directory)
                    .start();

            final BlockingQueue<String> inputQueue = new LinkedBlockingQueue<>();
            final BlockingQueue<String> errorQueue = new LinkedBlockingQueue<>();
            taskExecutor.execute(this.readStreamToQueue(new BufferedInputStream(process.getInputStream()), inputQueue));
            taskExecutor.execute(this.readStreamToQueue(new BufferedInputStream(process.getErrorStream()), errorQueue));

            try (final OutputStream outputStream = new BufferedOutputStream(new GZIPOutputStream(process.getOutputStream()))) {
                consumer.accept(outputStream);
            }

            return new ProcessResultDto(process.waitFor(), inputQueue, errorQueue);
        } catch (Exception e) {
            throw new RuntimeException();
        } finally {
            if (Objects.nonNull(process)) {
                process.destroy();
            }
        }
    }

    private Runnable readStreamToQueue( InputStream inputStream,  BlockingQueue<String> queue) {
        return () -> {
            String line;
            try (final BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream))) {
                while ((line = reader.readLine()) != null) {
                    queue.add(line);
                }
            } catch (Exception e) {
                logger.warn(e.getMessage(), e);
            }
        };
    }

    @FunctionalInterface
    public interface DataConsumer {
        void accept(OutputStream out) throws Exception;
    }

}

```

#### 测试调用代码 Python

```
import sys
import gzip
import json

def getInputData():
    gzipByteArray = sys.stdin.buffer.read()
    jsonStr = gzip.decompress(gzipByteArray).decode("utf-8")
    return json.loads(jsonStr)

```
