# PYATS -- UNICON
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




