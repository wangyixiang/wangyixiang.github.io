*电子邮件确实是非常好的de-centralized的方案,internet的核心思想之一
 就是去中性化,但是搭好和管理好email服务器,在今天这个互联网环境之下,
 已经没有这么简单了.要考虑和处理的方面是比当年相对纯洁的互联网环境下
 要多的多了*

虽然有[qmail], [postfix]和[sendmail]几家分天下, 但是现在大家基本
都会选择[postfix]作为MTA, 而IMAP和POP3 email server, 我就选择
[dovecot], 两个个很重要的原因就是:

1.  这个组合在现实使用中可以所说是久经考验,而且维护的力量也很持续
2.  [postfix]可以直接使用[dovecot]的authentication backend

但是这只是基本技术的选定,面对着邮箱域名的注册和管理,邮件传输加密,
反垃圾邮件组件的选择, 备份方案, 防火墙的configuration, 
还有入侵保护, 系统监控, 邮件系统管理, 如果要提供web邮件客户端的话,
那么相关的系统又有一块. 可以看到确实是一个比较辅助的过程.


[dovecot]: http://dovecot.org/ "dovecot official site"
[intodns]: http://www.intodns.com "intodns"
[postfix]: http://www.postfix.org "postfix official site"
[qmail]: http://cr.yp.to/qmail.html "qmail official site"
[sendmail]: http://www.sendmail.com/sm/open_source "sendmail official site"
[mailinabox]: https://mailinabox.email "mail in a box official site" 