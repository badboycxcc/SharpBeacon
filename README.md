## 20240929 更新

**增加Https协议支持**

![image](https://github.com/user-attachments/assets/4d08289b-5915-4fbd-90de-a635dc6604d4)


![image](https://github.com/user-attachments/assets/e118567f-23fe-47c4-baee-60aeb96f821f)


**修复指定Hosts地址无法上线BUG**
![image](https://github.com/user-attachments/assets/11c6ae93-7188-4f81-97f8-59e97aedd79b)



--------------------------------------------------------- 

# 如何使用
启动teamserver后会生成beacon_keys文件，我们需要提取出里面的RSA 公私密钥后，复制到config.cs中，并在config.cs修改回传的URL和生成hash的随机数。
# How to use
Firstly, starting TeamServer and you got .cobaltstrike.beacon_keys meanwhile configurate listener etc.
Secondly, compiling SharpBeacon with VisualStudio after you changed url and RSA private key and public key in config.cs
And Then click sharpbeacon.exe. Once you got one beacon session and have fun!
BTW this project as just a beacon which depends on CobaltStrike.   -- bopin2020 

<br>
关于使用syscall注入的问题 bopin2020师傅在win11测试calc.exe 创建线程没有成功；其他notepad,powershell都没有问题。感谢 bopin2020 师傅。
<br>

---
# SharpBeacon
CobaltStrike Beacon written in .Net 4  用.net重写了stager及Beacon，其中包括正常上线、文件管理、进程管理、令牌管理、结合SysCall进行注入、原生端口转发、关ETW等一系列功能

# 一、概述
这次我们一起用C#来重写stager及其Beacon中的大部分常用功能，帖子主要介绍该项目的运行原理（LolBins->Stager->Beacon）及相应的功能介绍及展示。LolBins部分是由GadgetToJs使Stager转换为js、vba、hta文件后，再结合相应的csript、mshta等程序来运行；Stager功能包括从网络中拉取Beacon的程序集并在内存中加载及AMSIBypass；Beacon部分主要有包括正常上线、文件管理、进程管理、令牌管理、结合SysCall进行注入、原生端口转发、关ETW等一系列功能。<br>
项目基于.net4.0，暂支持cs4.1（更高版本待测试），感谢M大、WBG师傅、SharpSploit、Geason的分享。另因最近出去广州找工作没时间弄，就暂时写到这里，开发进度比较赶致使封装不是很好、设计模式也没有用，但每个实现功能点都有较详细注释，等后续工作安定后会进行重构及完善更多功能。若有错误之处还请师傅指出，谢谢大家。<br>

**详见帖子：魔改CobaltStrike_重写Stager和Beacon https://bbs.pediy.com/thread-269115.htm**

# 二、LolBins
LOLBins，全称“Living-Off-the-Land Binaries”，直白翻译为“生活在陆地上的二进“，我大概将其分为两大类：<br>
* 1、带有Microsoft签名的二进制文件，可以是Microsoft系统目录中二进制文件。
* 2、第三方认证签名程序。
LolBins的程序除了正常的功能外，还可以做其他意想不到的行为。在APT或红队渗透常用，常见于海莲花等APT组织所使用。下图是较常见的LolBins，还有很多就不一一列出了：<br>
![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%871.png)
 
而GadgetToJS项目则可以把源码cs文件动态编译再base64编码后，保存在js、vba、vbs、hta文件，而在其相关文件中文件利用了当 BinaryFormatter属性在进行反序列化时，可以触发对 Activator.CreateInstance() 的调用，，从而实现 .NET 程序集加载/执行。<br>
![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%872.png)
 
但这需要在.net程序集中把相应的功能写在默认/公共构造函数，这样才能触发 .NET 程序集执行。下面以实例程序为例：<br>
![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%873.png)
 
在相应文件夹下执行如下命令：<br>
.\GadgetToJScript.exe -w js -c Program.cs -d  System.Windows.Forms.dll -b -o gg
其中命令参数解析如下：
* -w js表示所生成的是js文件，可以生成其他形式的文件
* -c Program.cs是所选择的cs文件
* -d  System.Windows.Forms.dll  cs文件所用到的dll
* -b  会在js文件中的引入第一个stager，因为当在.NET4.8+的版本中引入了旁路类型检查控件，默认值为false，如果所生成的脚本要在.NET4.8+的环境中运行，则设置为true（--Bypass/-b）。生成的stager1就是bypass这个检查的。
* -o gg生成文件名
生成js、hta、vbs等文件后默认是会被杀的：<br>
![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%874.png)<br>

而我们只需要简单修改下单引号为/就行了:<br>
![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%875.png)<br>
![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%876.png)<br>

最后执行所生成的js或hta：<br>
![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%877.png)<br>
![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%878.png)<br>

# 三、Stager
Stager部分的功能可以包括下图几项： <br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%879.png)
我主要实现了从网络中拉取Beacon的程序集并在内存中加载及AMSIBypass，沙箱及虚拟机检测的方式有挺多方式的，师傅可以自行添加。
拉取程序集及内存加载这个较为简单，就不细说了：<br>
![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8710.png)<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8711.png)<br>
 
下面说说bypassAMSI，这里一开始找的不是AmsiScanBuffer，而是找DllCanUnloadNow的地址：<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8712.png)<br>
 
然后再通过相关的硬编码找到AmsiScanBuffer后，再进行相应的patch：<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8713.png)<br>

# 四、Beacon
Beacon部分主要有包括正常上线、文件管理、进程管理、令牌管理、结合SysCall进行注入、原生端口转发、关ETW等一系列功能。<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8714.png)<br>

## 4.1文件管理
先从文件管理部分说，包含了cp、mv、upload、download、filebrowse、rm、mkdir上述这七个功能点：<br>
Cp:<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8715.png)<br>
  ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8716.png)<br>
Mv:<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8717.png)<br>
Upload:<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8718.png)<br>
Download:<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8719.png)<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8720.png)<br>
Filebrowse:<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8721.png)<br>
rm:<br>
![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8722.png)<br>
mkdir<br>
  ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8723.png)<br>

## 4.2进程部分
进程部分，已完成的有run、shell、execute、runas、kill，未完成的有runu:<br>
Run:<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8724.png)<br>
shell:<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8725.png)<br>
execute:<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8726.png)<br>
runas:<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8727.png)<br>
ps:<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8729.png)<br>
kill:<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8730.png)<br>
 
## 4.3令牌权限
令牌权限部分，已完成的有getprivs、make_token、steal_token、rev2self：<br>
Getprivs:<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8731.png)<br>
  ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8732.png)<br>
 
make_token：测试时在make_token后执行了cmd.exe /C dir \\10.10.10.165\C$ <br>
![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8733.png)<br>
steal_token：测试时在steal_token后执行了whoami<br>
  ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8734.png)<br>
rev2self：<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8735.png)<br>
 
## 4.4端口转发
端口转发部分，已完成的有rportfwd、rportfwd stop：<br>
Rportfwd，注意这里端口转发teamserver只返回了本地需要绑定的端口，没有返回需转发的ip和port。<br>
在192.168.202.180:22222上新建msf监听：<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8736.png)<br>
在本机地址192.168.202.1的23456端口转发到上述msf的监听<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8737.png)<br>
  ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8738.png)<br>
本地访问23456端口：<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8739.png)<br>
另一个网段访问23456端口：<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8740.png)<br>
rportfwd stop：<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8741.png)<br>
 
## 4.5注入部分
注入部分，cs的shinject、dllinject、inject都用来远程线程注入，我个人机器是win10 x64 1909,shellcode是用cs 的64位c# shellcode，被注入的程序是64位的calc.exe，程序返回的NTSTSTUS均为SUCCESS，且shellcode均已注入在相应的程序中，并新建出线程进行执行，但最后calc.exe都崩了，有点奇怪呀：<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8742.png)<br>
申请rwx内存空间存放shellcode后并在所执行shellcode下断:<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8743.png)<br>
执行NtCreateThreadEx(),被注入的calc.exe新建线程执行此shellcode：<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8744.png)<br>
最后跑起来报的c05，但分配的内存属性是rwx的：<br>
 ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8745.png)<br>
 
## 4.6杂项部分
杂项部分，已完成有sleep、pwd、exit、setenv、drives、cd：<br>
Sleep：<br>
  ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8746.png)<br>
exit：<br>
  ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8747.png)<br>
setenv： <br>
  ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8748.png)<br>
drives： <br>
  ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8749.png)<br>
Pwd：<br>
  ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8750.png)<br>
cd：<br>
  ![](https://github.com/mai1zhi2/SharpBeacon/blob/master/image/%E5%9B%BE%E7%89%8751.png)<br>
  
# 五、完善及改进
后续需要改进的地方还有很多，有如下几点：<br>
* 1、该封装好就封装好，该用设计模式就用
* 2、目前rsa密钥是pem方式就用了BouncyCastle库，要用回Exponent 和 Modulus
* 3、更多的注入方式，APC、傀儡进程等
* 4、更多的通信协议，如DNS、ICMP
* 5、支持spawn**，因为当执行spawn和job后，teamserver端会回传相应的dll，要改ts端
* 6、更多的功能，如mimi、keylogger、portscan、加载pe等
最后谢谢大家观看。

