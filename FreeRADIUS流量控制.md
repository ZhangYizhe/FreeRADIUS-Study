# FreeRADIUS流量控制

建立流量控制表格


```
INSERT INTO radgroupreply (groupname,attribute,op,VALUE) VALUES ('user','Auth-Type',':=','Local');
INSERT INTO radgroupreply (groupname,attribute,op,VALUE) VALUES ('user','Service-Type',':=','Framed-User');
INSERT INTO radgroupreply (groupname,attribute,op,VALUE) VALUES ('user','Framed-IP-Address',':=','255.255.255.255');
INSERT INTO radgroupreply (groupname,attribute,op,VALUE) VALUES ('user','Framed-IP-Netmask',':=','255.255.255.0');
INSERT INTO radgroupreply (groupname,attribute,op,VALUE) VALUES ('user','Acct-Interim-Interval',':=','600');
INSERT INTO radgroupreply (groupname,attribute,op,VALUE) VALUES ('user','Max-Monthly-Traffic',':=','5368709120');
INSERT INTO radgroupcheck (groupname,attribute,op,VALUE) VALUES ('user','Simultaneous-Use',':=','1');
```

acct-interim-interval是计算流量的间隔（600秒），意味着每隔10分钟记录当前流量。倒数第二行是每月最大流量，这里是5G（单位是字节）。最后一行是允许同时连接数目。

创建完表格，编辑`/usr/local/etc/raddb/dictionary`，到最后一行，添加`ATTRIBUTE Max-Monthly-Traffic 3003 integer`。

进入radius -X调试模式，看看是否正常，如果正常，继续看下面操作：

编辑`/usr/local/etc/raddb/sites-enabled/default`，找到`authorize`字节，添加下面内容：


```
update request {
Group-Name := "%{sql:SELECT groupname FROM radusergroup WHERE username='%{User-Name}' ORDER BY priority}"
}
if ("%{sql: SELECT SUM(acctinputoctets+acctoutputoctets) FROM radacct WHERE username='%{User-Name}' AND date_format(acctstarttime, '%Y-%m-%d') >= date_format(now(),'%Y-%m-01') AND date_format(acctstoptime, '%Y-%m-%d') <= last_day(now());}" >= "%{sql: SELECT value FROM radgroupreply WHERE groupname='%{Group-Name}' AND attribute='Max-Monthly-Traffic';}") {
reject
}
```

至此，完事，如果流量超限了，用户登录则无法通过验证，会提示691的错误。

