# 第一节-AWVS

## 1. AWVS简介

​		AWVS (Acunetix Web Vulnerability Scanner)是一款知名的网络漏洞扫描工具，通过网络爬虫测试网站安全，检测流行的Web应用攻击，如跨站脚本、sql注入等。据统计，75%的互联网攻击目标是基于Web的应用程序

## 2. 为什么要用AWVS

​		在今天，网站的安全是容易被忽视的。黑客具备广泛的攻击手段，例SQL注入，XSS，文件包含，目录遍历，参数篡改，认证攻击等。虽然你配置了正确的防火墙和WAF，但是这些安全防御软件仍然存在策略性的绕过。因此，需要您定期的扫描你的web应用，但是手动检测你所有的web应用是否存在安全漏洞比较复杂和费时，所以您需要一款自动化的web漏洞扫描工具来检测您的web应用是否存在安全漏洞。

# 第二节-扫描Web应用程序

## 1. 账户密码登录扫描

​		（1）点击【Targets】，点击【add Target】

​		（2）输入扫描地址和扫描描述,点击【save】

​		（3）点击【Site Login】

​		（4）选择【try to auto-login into the site】，输入登录地址，用户名，密码，重复密码

​		（5）点击【HTTP Authentication】的开启按钮

​		（6）输入用户名，密码，重复密码

​		（7）点击【save】，然后点击【Scan】按钮

​		（8）选择【Scan Profile】、【Report】、【Schedule】和日期，点击【create Scan】

​		（9）点击扫描目标，查看扫描具体内容

## 2. 利用录制登录序列脚本扫描

​		（1）点击【Targets】，点击【add Target】

​		（2）输入扫描地址和扫描描述,点击【save】

​		（3）点击【Site Login】

​		（4）选择【Use pre-recorded login sequence 】,点击【New】

​		（5）进行登录操作

​		（6）检查登录脚本流程是否完整，点击【Next】

​		（7）点击【Next】，然后点击【确定】

​		（8）点击【Finish】，然后点击【Scan】

## 3.利用定制cookie扫描

扫描过程会遇到网站 存在 手机验证码，图形验证码，滑动验证等等，这时候想要深度扫描时，就需要进行登录绕过。最长用的手段就是定制cookie绕过。

​		（1）点击【Targets】，点击【add Target】

​		（2）输入扫描地址和扫描描述,点击【save】

​		（3）点击【Advanced】

​		（4）点击【Custom Cookies】，输入被测网站网址

​		（5）切换到其他浏览器，获取网站的cookie值

​		（6）切换回AWVS，输入cookie的值，点击【+】

​		（7）点击【save】，然后点击【Scan】

# 第三节-扫描报告分析

## 1. AWVS报告类型

​		Standard Reports：标准报告

​		Affected Items：受影响项目

​		Comprehensive (new)：综合（新）

​		Developer：开发者

​		Executive Summary：执行摘要

​		Quick：快速报告

## 2. Compliance Reports：合规报告

​		CWE / SANS Top 25：SANS (SysAdmin, Audit, Network, Security) 研究所是美国一家信息安全培训与认证机构

​		DISA STIG：DISA STIG 是指提供技术指南（STIG — 安全技术实施指南）的组织（DISA — 国防信息系统局）

​		HIPAA：HIPAA标准

​		ISO 27001：国际标准

​		NIST SP 800-53：联邦信息系统标准

​		OWASP Top 10 2013：开放式Web应用程序安全项目 2013标准

​		OWASP Top 10 2017：开放式Web应用程序安全项目 2017标准

​		PCI DSS 3.2：即支付卡行业数据安全标准

​		Sarbanes Oxley：萨班斯法案标准

​		WASC Threat Classification：WASC 组织标准

## 3. 最常用的报告类型：

​		Executive Summary：执行摘要 给公司大领导看，只关注整体情况，不关注具体细节

​		Comprehensive (new)：综合（新）：一般给QA和产品经理看

​		Developer：开发者：给开发人员看

​		OWASP Top 10 2017 行业报告的代表

​		WASC Threat Classification 行业报告的代表

# 第四节-Goby+AWVS 联动

## 1. Goby简介

​		Goby 是针对目标企业梳理最全面的工具，通过 goby 可以清晰的扫描出 ip 地址开放的端口，以及端口对应的服务，于此同时会根据开放的端口及应用进行实战化的测试，并不在乎中低危害漏洞，而更在乎的是能直接 getshell 的漏洞。

​		AWVS 是针对web的轻量级的漏洞扫描工具。也就是根据我们提供的被扫描地址，快速的扫描出其所有的漏洞，包含高中低及信息泄露等漏洞。

​		Goby 探测出 ip 地址开放的所有服务及应用，然后直接丢给AWVS，那么AWVS是不是就可以直接进行扫描了，然后存在的网站存在的漏扫是不是一幕了然了，还需要我们去手动挖么，很显然了啊，这俩工具一联动，躺着收漏洞呗。

## 2. Goby安装

## 3. 安装npcap-0.9995.exe

## 4. Goby+AWVS联动扫描

​		（1）点击【扫描】

​		（2）输入ip进行扫描,点击【开始】

​		（3）等待扫描结果

​		（4）点击【Web检测】)

​		（5）点击【awvs】扫描

​		（6）切换到【AWVS】，点击【Scans】，点击【Goby传过来的任务】

​		（7）切回到【Goby】，点击【扩展程序】,点击【awvs】

​		（8）选择报告模板，点击【Generate】,生成报告

​		（9）点击【Export】可以导出报告