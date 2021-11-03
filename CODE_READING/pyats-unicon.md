# PYATS -- UNICON

[TOC]

## Pyats简介
Pyats是思科开源的自动化测试框架。主要使用CLI与设备交互，支持多种device 类型，比如思科自己的IOS-XE设备，IOS-XR设备，以及JunOS设备和linux Server。
[Pyats Official Doc](https://developer.cisco.com/docs/pyats/)

## Unicon简介
Unicon是Pyats的一个library, 主要用于和设备建立连接，发送认证信息，以及发送接收message。可以看成是一个程序实现的telnet/ssh shell。
和大多数类似的实现一样，Unicon主要包含两个部分:
1. telnet/ssh client，复杂连接device
2. input/output 处理，主要是向设备发送字符，并接收设备输出。

## Unicon代码分析
Unicon的代码目前在公网上还找不到下载的地方，github中有[unicon的plugin代码](https://github.com/CiscoTestAutomation/unicon.plugins)。从pypi上可以找到unicon的.whl文件，但是展开以后是编译好的cpython库，也看不到源代码。

在Java的生态里面有实现上述功能的库，比如：
1. ssh/telnet client -- sshj, apache/commons-net
2. input/output处理 -- expectj
但是只有这些零散的库，而使用这些库对相应的device进行处理，则需要用户自己编写代码，不如Unicon方便。

### example code
下面这段对unicon使用的代码来自于unicon下面的demo目录

```python
from unicon import Connection

dev_single = Connection(hostname='step-n5k-1',
                        start=['telnet 10.64.70.24 2060'],
                        tacacs_username='admin',
                        tacacs_password=r'Cscats123!',
                        enable_password='lab',
                        os='nxos')

dev = dev_single
dev.connect()
from time import sleep
from unicon.eal.dialogs import Statement, Dialog

def send_response(spawn, response=""):
    sleep(0.5)
    spawn.sendline(response)
write_erase = Statement(pattern=r'.*Do you wish to proceed anyway\? \(y/n\)\s*\[n\]',
                                action=send_response, args={'response': 'n'},
                                loop_continue=True,
                                continue_timer=False)
dev.execute("write erase", reply=Dialog([write_erase]))
dev.config()
```
可以看到有了unicon以后，用户可以直接使用connect()和execute()进行登录和运行CLI命令，而不用在用户代码中添加一堆的处理函数来应对Username：， Password：这种输入情况的处理。

Unicon这是一个大的框架，而对于不同设备的具体处理，在plugin的代码里面。从plugin的__init__.py中可以看到支持的设备类型非常多。

```python
supported_os = [
    'generic',
    'ios',
    'nxos',
    'iosxe',
    'iosxr',
    'aireos',
    'linux',
    'cheetah',
    'ise',
    'asa',
    'nso',
    'confd',
    'vos',
    'cimc',
    'fxos',
    'junos',
    'staros',
    'aci',
    'sdwan',
    'sros',
    'apic',
    'windows'
]
```
### plugin的发现与安装
先看一下plugin是怎么和unicon结合在一起的。
在根目录的__init__.py中会初始化PluginMananger,并且把plugins/ 下的所有python module都import 进系统。

### Connection 类
从一个cat9k的Connection实现来看继承的层级
![Cat9k Connection](https://github.com/lpfwd/MyReadingNotes/blob/main/pics/Cat9kConnection.png?raw=true)

最基础的类是Connection, 在Connection基础上增加了 Single RP, IosXE, Cat9k的特殊属性

那么关于Connection类的定义

```python
class Connection(Lockable, metaclass=ConnectionMeta):
```

这里用到了meta class, 可以参考[廖雪峰老师的教程](https://www.liaoxuefeng.com/wiki/1016959663602400/1017592449371072)
plugin manager 在注册Plugin的时候会根据os, series, model来保存不通的plugin的connection类，详细的数据结构见下图：
![plugin manager内部数据](https://github.com/lpfwd/MyReadingNotes/blob/main/pics/pluginmanager_internal_data.png?raw=true)


用户代码在创建connection的时候，只需要调用base connection类，base connection类会根据传入的os,series,model参数选择正确的子类进行实例化。

我们主要研究IOS-XE平台，所以先从plugins/ios-xe入手
目录结构
unicon
  +--plugins
       +-- ios-xe
            +-- cat3k
            +-- cat9k
            ....
            +-- __init__.py
            +-- patterns.py
            +-- service_implementation.py
            +-- service_statements.py
            +-- settings.py
            +-- statemachine.py
            +-- statements.py
  
  其中 '__init__.py' 会包含一些重要的class
  
###   connect()函数
  初始化完一个*Connection class*以后，就会调用其 *connect()* 函数对device进行连接。
对于 *IosXESingleRpConnection* 会直接调用其base claas -- Connection的connect()
在继续进行connect()函数的介绍之前，需要先看一下Connection类的几大组件。

###组件
#### Spawn
![Spawn继承结构图](https://github.com/lpfwd/MyReadingNotes/blob/main/pics/unicon_spawn.png?raw=true) 
  
从上图可以看出Spawn的继承结构还是比较复杂的，Spawn主要复杂最底层的功能，连接设备，收发字符，判断buffer中的字符是否匹配用户的pattern.
从Spawn base class的抽象函数可以看出来其基本功能

```python
    @abstractmethod
    def expect(self, *args, **kwargs):
        raise NotImplementedError('this method must be overwritten ..')

    @abstractmethod
    def send(self, *args, **kwargs):
        raise NotImplementedError('this method must be overwritten ..')

    @abstractmethod
    def sendline(self, *args, **kwargs):
        raise NotImplementedError('this method must be overwritten ..')

    @abstractmethod
    def read_update_buffer(self, size=None):
        raise NotImplementedError('this method must be implemented ..')

    @abstractmethod
    def match_buffer(self, pat_list):
        raise NotImplementedError('this method must be implemented ..')

    @abstractmethod
    def trim_buffer(self):
        raise NotImplementedError('this method must be implemented ..')

    @abstractmethod
    def close(self, *args, **kwargs):
        raise NotImplementedError('this method must be implemented ..')
```

对于其他的实现，如果收到 *ssh -l user ip* 需要用ssh client, 如果是收到
*telnet ip* 需要调用 telnet client. 在这里unicon 用了其他的方式实现，他使用了一个库pty. 并调用了 pty.fork(), fork出来的子进程需要调用 *os.execvp()* 去直接调用shell的telnet 和 ssh命令， 返回的fd被父进程保存下来，用来 read/write 来实现收发字符。

```
pty.fork()
   将子进程的控制终端连接到一个伪终端。 返回值为 (pid, fd)。 请注意子进程
获得 pid 0 而 fd 为 invalid。 父进程返回值为子进程的 pid 而 fd 为一个连接到
子进程的控制终端（并同时连接到子进程的标准输入和输出）的文件描述符。
```
在Spawn中的expect()函数包含几个操作:
1. 读取stdin
2. 保存在buffer里面
3. 和传入expect()函数的正则表达式列表做匹配
expect() 在读取stdin到buffer时的默认设置：
> timeout : 10s
> buffer size: 4096bytes

#### Dialog
CLI一般是用于与设备交互，就是用户输入一条命令，根据输入再进行下一步的动作。而unicon实现的就是类似这样的结构

```
execute(command);
expect(pattern);
...
execute(command);
expect(pattern);
```
但是对程序来说，还有很多更复杂的情况。比如reload命令，用户输入完以后，如果没有保存running config,那么设备会输出 *System configuration has been modified. Save? [yes/no]:* 如果选择yes,那么设备接着输出 P*roceed with reload? [confirm]* 这个时候之前的一个命令，一个expect就不能处理了，所以unicon设计出了Dialog

Dialog就是Statement的list:

```python
            dialog = Dialog([
                Statement(pattern=r"^username:$",
                          action=lambda spawn: spawn.sendline("admin"),
                          args=None,
                          loop_continue=True,
                          continue_timer=False ),
                Statement(pattern=r"^password:$",
                          action=lambda spawn: spawn.sendline("admin"),
                          args=None,
                          loop_continue=True,
                          continue_timer=False ),
                Statement(pattern=r"^host-prompt#$",
                          action=None,
                          args=None,
                          loop_continue=False,
                          continue_timer=False ),
            ])
```

那么Statement是什么呢？根据代码注释：

```python
    Description:
        Dialogs are nothing but a set of statements. Statement contains
        five things:

            * pattern: a match is used to invoke the callback.
            * action: a function which is called in case the pattern is matched.
            * args: a dict containing all the arguments required by callback function.
            * loop_continue: When it is true, the dialog doesn't exit and continues to look for the match.
            * continue_timer: timer is restarted after every match.
            * trim_buffer: When it is False, matched content will not be removed from buffer.
            * matched_retries: retry times if statement pattern is matched, default is 0
            * matched_retry_sleep: sleep between retries, default is 0.02 sec
```

Dialog 实例化以后，会调用方法process()处理这个Dialog. process()会实例化一个DialogProcessor()，然后调用其方法process()
在实例化DialogProcessor的时候会传入相应的spawn, process()方法会使用spawn读取设备的输出，匹配Statement里面的pattern, 然后执行Statement里面的action.


#### ExpectMatch
先看一下ExpectMatch的内部字段

```python
    def __init__(self):
        self.last_match = None
        self.last_match_index = None
        self.last_match_mode = None
        self.match_output = ""
```
如果expect()或者Dialog的process(), 匹配到一个pattern, 那么会把re.search()的结果存入last_match, 把pattern list里面match到的pattern的index存入last_match_index, 把re.search().group()存入match_output。最后会调用spawn.trim_buffer(),把match的output从spawn的buffer里面拿掉。

#### StateMachine
Unicon内部实现了一个简单的state machine框架，目录在
unicon
  +--statemachine
        +-- __init__.py
        +-- statemachine.py      *main function of state machine*
        +-- statetransition.py    *helper function to do transition operation.*

其中抽象出来的有：
* State
* Path
* StateMachine
* StateTransition

State比较好理解就是状态，其中包含了name和pattern, 比如当前状态是config, 那么对于的pattern 就是匹配 *Hostname(config)#* 的正则表达式
Path就是状态迁移，State类似图上的点，那么path就是连接两个点的有向边。其中包含了
from_state, to_state, command 和Dialog
code example:

```python
            path1 = Path(from_state='enable',
                         to_state='config',
                         command='config terminal'
                         dialog=None)

            path2 = Path(from_state='disable',
                         to_state='enable',
                         command='en'
                         dialog=Dialog([r'password:', action=send_password, None, True, True]))
```

StateMachine类就是包含了所有的State，Path的集合体,以及当前的state.
如果要驱动一个状态转义，就是比如从disable到config, 那么需要先建立一个StateTransition，然后调用 .do_transitions()

State Machine需要解决的一个问题就是最短路径。
比如有下图的state machine:
![State Machine Path](https://github.com/lpfwd/MyReadingNotes/blob/main/pics/state_machine_path.png?raw=true)
从**state1**到**state5**,总共有3条路径
* 1，2，5
* 1，4，2，5
* 1，4，6，5
但是最短的路径是1,2,5, 那么状态转换就从这条路走。
unicon里面使用的算法是把from_state和to_state中间所有的Path都找出来，每一条路由存成一个list,然后比较list的长度，找出最短的路径。

```
可以对比Spring中StateMachine的实现
```





