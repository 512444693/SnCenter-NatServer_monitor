# SnCenter-NatServer_monitor

## 配置
在**sncenter_list.csv**文件中配置好sncenter的ip、端口、注释等信息；例如
![image](https://github.com/512444693/resources/blob/master/SnCenter-NatServer_monitor/1.jpg)

## 功能
### JMeter
- jmeter会遍历sncenter_list.csv中所有的sncenter，向其发送协议取得所有的natserver的信息后存储到**natserver_list.csv**中，并将natserver去重

![image](https://github.com/512444693/resources/blob/master/SnCenter-NatServer_monitor/2.jpg)
- jmeter会遍历natserver_list.csv中的natserver向其发送各种协议并做断言
- 向某个sncenter发送协议获取natserver信息是一组协议；向某个natserver发送协议并断言是一组协议
- 若某一组协议发送失败(一个协议失败即视为失败)，则可重试，总共可重试**6次**(可配)
- 若某一组协议重试了所有次数，但都失败，则视为服务器异常，在控制台打印出"XXX.XXX.XXX.XXX FAIL!!!!!"

### jenkins
- 定时构建执行
- 若JMeter监控到有服务器(sncenter或natserver)异常，发送邮件给相关人员



## 具体
### 检测异常并发送邮件
- 增加构建后操作步骤"Extended E-mail Notification"
- 点击"Add Trigger"添加"Script_After Build"
- 若在构建日志中发现"FAIL!!!!!"字符串，则发送邮件
![image](https://github.com/512444693/resources/blob/master/SnCenter-NatServer_monitor/11.jpg)

### 线程组
![image](https://github.com/512444693/resources/blob/master/SnCenter-NatServer_monitor/3.jpg)
- setUp Thread Group只有一个功能，**删除natserver_list.csv**
- work 线程组的功能是，遍历sncenter_list.csv中的**sncenter**
- tearDown Thread Group的功能是，遍历natserver_list.csv中的**natserver**

### 遍历
以work线程组为例，演示如何遍历sncenter_list.csv
- 线程组中循环为Forever

![image](https://github.com/512444693/resources/blob/master/SnCenter-NatServer_monitor/4.jpg)
- 在CSV Data Set Config中，配置Recycle on EOF 为false，配置Stop thread on EOF为true
![image](https://github.com/512444693/resources/blob/master/SnCenter-NatServer_monitor/5.jpg)


### 失败重试
![image](https://github.com/512444693/resources/blob/master/SnCenter-NatServer_monitor/7.jpg)

我们在work和tearDown线程组中均实现了失败重试；以work线程组为例来看一下
- 通过"初始化循环条件"、"循环次数减一"**BeanShell Sampler**来控制可循环次数
- 通过"监听是否失败"**BeanShell Listener**来判断是否失败，是否需要重试
- 通过**While Controller**来控制循环
- 总的来说原理就是对变量的存取、判断来实现失败重试
- "判断是否失败" BeanShell Sampler来根据失败次数判断是否该服务器失败

### 验证sncenter
![image](https://github.com/512444693/resources/blob/master/SnCenter-NatServer_monitor/8.jpg)
- 只发送一个获取natserver列表的协议，并断言

### 验证natserver
![image](https://github.com/512444693/resources/blob/master/SnCenter-NatServer_monitor/10.jpg)
- 模拟用户A发送ping包，并比较返回的peerid和从sncenter得到的peerid是否一致
- 模拟用户B发送打孔协议，并断言用户A在线
- 模拟用户A发送logout包
- 模拟用户B发送打孔协议，并断言用户A不在线

## 其它
### 参数
- 通过-Jtimer=X 来指定协议之间的思考时间
- 通过-Jtry_times=X 来指定一组协议的尝试次数
![image](https://github.com/512444693/resources/blob/master/SnCenter-NatServer_monitor/12.jpg)

### 报表
- 如果某个ip的失败次数大于0，将该数据写入report.csv中，格式如下

        192.168.1.1,192.168.1.2,192.168.1.3,
        3,4,2
- 下载"Plot Plugin"插件，添加构建后步骤"Plot build data"，并配置

### 说明
![image](https://github.com/512444693/resources/blob/master/SnCenter-NatServer_monitor/13.jpg)
之所以先用Jmeter测试生成文件，再用Ant通过文件生成报告，而不是直接用Ant调用Jmeter，是由于两个原因
- 由于某些原因，Ant调用Jmeter生成的natserver_list.csv文件不在当前目录，而在Jmeter的bin目录下
- Ant调用Jmeter无法通过-J指定参数

