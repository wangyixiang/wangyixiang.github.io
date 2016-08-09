电子邮件确实是非常好的de-centralized的方案, internet的核心思想之一就是去中性化, 但是搭好和管理好email服务器, 在今天这个互联网环境之下, 已经没有这么简单了. 要考虑和处理的方面是比当年相对纯洁的互联网环境下要多的多了

虽然有[qmail], [postfix]和[sendmail]几家分天下, 但是现在大家基本都会选择[postfix]作为MTA, 而IMAP和POP3 email server, 我就选择[dovecot], 两个很重要的原因就是:

1.  这个组合在现实使用中可以所说是久经考验,而且维护的力量也很持续
2.  [postfix]可以直接使用[dovecot]的authentication backend

但是这只是基本技术的选定,面对着邮箱域名的注册和管理,邮件传输加密,反垃圾邮件组件的选择, 备份方案, 防火墙的configuration, 还有入侵保护, 系统监控, 邮件系统管理, 如果要提供web邮件客户端的话, 那么相关的系统又有一块. 可以看到确实是一个比较复杂的过程.

下面我们就对应于技术组件来一个一个梳理一下,我们要用到的技术
1.  SMTP: [postfix]  
做为MTA, [postfix]要完成别的MTA对它的投递工作, 也要完成对其他MTA的投递工作.  

    **在接受其他MTA的投递的时候:**  
    对于此MTA我们如何来进行甄别, 这里多管齐下是最有效的, 比如查询黑名单, 看看sender的域名或者发送IP是否在黑名单里或者白名单里, 这里的黑名单不只是本地的黑名单, 一般使用网上的可以提供在线查询的online黑名单系统,  
    **在向其他MTA投递的时候:**  
    

2.  [Greylisting][greylist]:   
When a request for delivery of a mail is received by Postfix via SMTP, the triplet CLIENT_IP/SENDER/RECIPIENT is built. 如果这个3元组是第一次见, 或者这个3元组是在第一次出现后的五分钟内出现,那么邮件将会被以一个临时性错误的理由来拒绝掉, 如果是spammer或者viruses在面对这个错误时一般不会再重试, 而正常的邮件服务器会来重试, 从而把spam邮件发送者和正常邮件发送者区分出来.  
[postgrey]是这么一个Postfix policy server的实现, 这是因为Postfix SMTP server has a number of built-in mechanisms to block or accept mail at specific SMTP protocol stages. In addition, the Postfix SMTP server can delegate decisions to an external policy server (Postfix 2.1 and later).

3.  spam filtering: [spamassassin]

4.  IMAP, POP3: [dovecot]


###Terms
1. [MTA](https://en.wikipedia.org/wiki/Message_transfer_agent)  
    Messenger Transfer Agent, 又名Mail Exchanger
2. [MSA](https://en.wikipedia.org/wiki/Message_submission_agent)  
    Messenger Submission Agent, 
3. [MUA](https://en.wikipedia.org/wiki/Email_client)  
    Mail User Agent, 又名email client
4. [SMTP Authentication](https://en.wikipedia.org/wiki/SMTP_Authentication)
5. [Email authentication](https://en.wikipedia.org/wiki/Email_authentication)
6. [Milter](https://en.wikipedia.org/wiki/Milter)
    MTA Extension, 
7. [DKIM](https://en.wikipedia.org/wiki/DomainKeys_Identified_Mail)  
    DomainKey Identified Mail, 这是用来防止email-spoofing的一个技术, 这个主要是一个由infrastructure来进行verify的技术, 所以她不同于end-to-end的digital signature技术. 这个技术主要是依靠非对称加密技术, HASH技术和DNS技术来完成, 在发送的邮件里会有一个DKIM-Signature字段, 里面有对邮件Header和Body的HASH值记录,而HASH值又用了与发布在DNS上的公钥对应的密钥加密,infrastructure就可以利用这个来检查邮件内容是否被篡改. 例子如下, 在邮件中加入的字段:
    >DKIM-Signature: v=1; a=rsa-sha256; c=simple/simple; d=box.cidanash.com;  
	s=mail; t=1470409800;  
	bh=bx8+5JGJKxuLiww2d2UALYTPGDMxjbDtBNQ2FYrXkEo=;  
	h=Date:From:To:Subject:From;  
	b=3QYQc/ARWNVjqIJzhOq3LhKii/L8rgF3UCIA86yABwFU2CR8OB/Cae2s0S6XWRa3+
	 TK8Ohaqk0yH+tR59pACZN8ysiBEw3wzoZor4/DWtLi0YzfdWCXdeYkZF/+H4Twe78p
	 Trb2aSddtwP7cZ+YCUrNFq/JRNhWVA7CJj8q/yUybmkOFCTWNP+sGedBPsP5uIbNXy
	 AyOpEETU3Rx+Y38l1axGuiP9sZDXgVMCKwuToKgtgzsHB95Lrw2ewHgrDyUBBg7RzY
	 /R7aqhtogmUwn51wUVBwa/ghqXQzi2F3UeqfVPp9cp1C2kjyc90rnU2OpExda1ZBmY
	 DtKjf44dT9B0w==
	 
	 在DNS上:
	 >mail._domainkey.box.cidanash.com	TXT	 
	 v=DKIM1; k=rsa; s=email; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA4VVlwUrLpsPJ6UVTaG5a/1IAPtqsj7AcmhpLeDIsnXJp0uU89XxapOZXFqG8AhRD3hFjV0blGxP2diVCRE1Grjz8jwDuQ/v/NRVbfZbSic0EkU9wK4mf/DFJxQWT4pW7EJP2GZazewkUGmKEdz33YXMX1OXp42dlzvyKrY4OeW6LoBQ/eCnRRfu67o+lRwQDGIUpqYUnWmZvaee4M8Crmz0XFQRhW/dbnSb8AxcYoyjIhn4stXMtNFBm+LfrNMNQuBaKPv9B+pIsGqk0nTfV2y288izR0xZF34ihnYIBw8OEwxSjK7m73/jTfaTNsQ8N9AUNu7eCxugX6WaI8NmBkQIDAQAB  
	 Recommended. Provides a way for recipients to verify that this machine sent @box.cidanash.com mail.
	 
8. [SPF](https://en.wikipedia.org/wiki/Sender_Policy_Framework)
    Sender Policy Framework, 这个技术是利用DNS系统来发布授权和禁止的hosts的TXT类型的record, 有的DNS就直接支持SPF RR这种类型, receiving mail exchanger可以根据在DNS中query出来的信息来check邮件中是否有forged的"from"信息, 详细的可以读wikipedia.例子如下:   
    >box.cidanash.com	TXT	v=spf1 mx -all  
Recommended. Specifies that only the box is permitted to send @box.cidanash.com mail.

    MX: If the domain name has an MX record resolving to the sender's address, it will match (i.e. the mail comes from one of the domain's incoming mail servers).  
    这个表示version是spf1, 根据上面定义, 如果sender的地址与DNS里解析出来的MX的记录match, 那么就说明了sender的合法性, 而-all就表示所有不符合的sender都应该reject
    >ns1.box.cidanash.com	TXT	v=spf1 -all  
Recommended. Prevents use of this domain name for outbound mail by specifying that no servers are valid sources for mail from @ns1.box.cidanash.com. If you do send email from this domain name you should either override this record such that the SPF rule does allow the originating server, or, take the recommended approach and have the box handle mail for this domain (simply add any receiving alias at this domain name to make this machine treat the domain name as one of its mail domains).  
    
    上面这个ns1.box.cidanash.com是用来作为Nameserver glue record来把domain name registrar中的NS定向到我们自己搭建的name server上来, 但是我们禁止outbound mail以@ns1.box.cidanash.com来作为域名来进行发送.
9. [DMARC](https://en.wikipedia.org/wiki/DMARC)  
    Domain-based Message Authentication, Reporting and Conformance (DMARC) 
    >_dmarc.box.cidanash.com	TXT	v=DMARC1; p=quarantine
Recommended. Specifies that mail that does not originate from the box but claims to be from @box.cidanash.com or which does not have a valid DKIM signature is suspect and should be quarantined by the recipient's mail system.

    >_dmarc.ns1.box.cidanash.com	TXT	v=DMARC1; p=reject
Recommended. Prevents use of this domain name for outbound mail by specifying that the SPF rule should be honoured for mail from @ns1.box.cidanash.com.
10. [Bounce Message](https://en.wikipedia.org/wiki/Bounce_message)


###References
1. [各大免费邮箱邮件群发账户SMTP服务器配置及SMTP发送量限制情况](https://www.freehao123.com/mail-smtp/)
2. [Demystifying SPF, DKIM, and DMARC](https://blog.returnpath.com/demystifying-spf-dkim-and-dmarc/)

[dovecot]: http://dovecot.org/ "dovecot official site"
[intodns]: http://www.intodns.com "intodns"
[postfix]: http://www.postfix.org "postfix official site"
[qmail]: http://cr.yp.to/qmail.html "qmail official site"
[sendmail]: http://www.sendmail.com/sm/open_source "sendmail official site"
[spamassassin]: https://spamassassin.apache.org "spamassassin official site"
[greylist]: http://projects.puremagic.com/greylisting "greylisting explanation"
[postgrey]: http://postgrey.schweikert.ch "postgrey official site"
[mailinabox]: https://mailinabox.email "mail in a box official site" 
[mxtoolbox]: https://mxtoolbox.com/SuperTool.aspx
[OpenDkim]: http://www.opendkim.org
[SPF]: https://en.wikipedia.org/wiki/Sender_Policy_Framework
[DMARC]: https://en.wikipedia.org/wiki/DMARC
[nsd]: https://www.nlnetlabs.nl/projects/nsd/
[DNSSEC]: https://en.wikipedia.org/wiki/Domain_Name_System_Security_Extensions
[DANE TLSA]: https://en.wikipedia.org/wiki/DNS-based_Authentication_of_Named_Entities
[SSHFP]: https://tools.ietf.org/html/rfc4255
[fail2ban]: http://www.fail2ban.org/wiki/index.php/Main_Page