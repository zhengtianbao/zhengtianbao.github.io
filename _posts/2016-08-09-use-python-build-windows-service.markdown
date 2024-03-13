---
layout: post
title:  "Use Python Build Windows Service"
date:   2016-08-09 14:25:53
categories: Python
---


起因是想写一个脚本在 Windows 虚拟机启动的时候对 SQLServer 服务进行一些初始化的操作，如修改用户密码，修改配置之类的。
于是写了一个 PowerShell 脚本去做这些事情，同时修改注册表设为自启动。
但是 Windows 实际情况却和 CentOS下 的 /etc/rc.d/rc.local 启动脚本不太一样，关键的区别就在于 Windows 需要等用户登录以后才会执行这个脚本而 linux 则是先执行完 rc.local 的内容后才加载 login 界面。
因此，用 python 写个 Service 跟随 Windows 系统启动来做这些操作，同时也借此学习了下 Windows 的服务。

代码很简单，用现成的 pywin32 包：

```python
import win32serviceutil   
import win32service   
import win32event   
  
class PythonService(win32serviceutil.ServiceFramework):   

    # 服务名  
    _svc_name_ = "PythonService"  
    # 服务显示名称  
    _svc_display_name_ = "Python Service Demo"  
    # 服务描述  
    _svc_description_ = "Python service demo."  
  
    def __init__(self，args):   
        win32serviceutil.ServiceFramework.__init__(self，args)   
        self.hWaitStop = win32event.CreateEvent(None，0，0，None)  
        self.logger = self._getLogger()  
        self.isAlive = True  
          
    def _getLogger(self):  
        import logging  
        import os  
        import inspect  
          
        logger = logging.getLogger('[PythonService]')  
          
        this_file = inspect.getfile(inspect.currentframe())  
        dirpath = os.path.abspath(os.path.dirname(this_file))  
        handler = logging.FileHandler(os.path.join(dirpath，"service.log"))  
          
        formatter = logging.Formatter('%(asctime)s %(name)-12s %(levelname)-8s %(message)s')  
        handler.setFormatter(formatter)  
          
        logger.addHandler(handler)  
        logger.setLevel(logging.INFO)  
          
        return logger  
  
    def SvcDoRun(self):  
        import time  
        self.logger.error("svc do run....")   
        while self.isAlive:  
            # 这里执行要做的的事
            self.setup_sqlserver()
            time.sleep(60)
        # 等待服务被停止   
        #win32event.WaitForSingleObject(self.hWaitStop，win32event.INFINITE)   

    def setup_sqlserver(self):
        pass

    def SvcStop(self):   
        # 先告诉SCM停止这个过程   
        self.logger.error("svc do stop....")  
        self.ReportServiceStatus(win32service.SERVICE_STOP_PENDING)   
        # 设置事件   
        win32event.SetEvent(self.hWaitStop)   
        self.isAlive = False  
  
if __name__=='__main__':   
    win32serviceutil.HandleCommandLine(PythonService)  
```

安装服务：

```
# 加参数--start auto设为自启动
python PythonService.py --startup auto install 
```

注意：

1. 最好将服务设置为延迟启动：

![延迟启动](/images/sqlserver.png)

2. 在注册表里设置依赖与SQLServer服务：

![注册表](/images/reg.png)

![依赖关系](/images/sqlserver2.png)

## 参考链接：

<http://blog.csdn.net/ghostfromheaven/article/details/8604738>

<http://www.cnblogs.com/xienb/archive/2012/07/26/2610643.html>
