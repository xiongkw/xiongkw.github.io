---
layout: post
title: JavaMail转发并删除已发送邮件
categories: [编程, java]
tags: [JavaMail]
---

> 使用`JavaMail`查收邮件并转发到指定邮箱

#### 1. 收信

使用`imap`协议收信

```java
Properties prop = new Properties();
prop.put("mail.imap.host", "imap.xx.com");
prop.put("mail.store.protocol", "imap");
prop.put("mail.smtp.auth", "true");
prop.put("mail.smtp.host", "smtp.xx.com");
Session session = Session.getDefaultInstance(prop);
Store store = session.getStore("imap");
store.connect("imap.xx.com", "test@xx.com", "123456");
Folder folder = store.getFolder("INBOX");
folder.open(Folder.READ_ONLY);
Message[] messages = folder.getMessages();
for (int i = messages.length - 1; i > 1; i--) {
    
}
folder.close(false);
store.close();
}
```

> 也可以使用`pop3`协议，但是`pop3`协议只能读取收件箱

#### 2. 转发

转发所有主题包含`xxx`的邮件到指定邮箱

```java
if (msg.getSubject().contains("xxx")) {
    MimeMessage forward = new MimeMessage(session);
    forward.setFrom(new InternetAddress(email));
    forward.setRecipient(Message.RecipientType.TO, new InternetAddress("aaa@xx.com"));
    String subject = "Fwd: " + msg.getSubject();
    forward.setSubject(subject);

    MimeBodyPart mainBody = new MimeBodyPart();
    mainBody.setText( "邮件转发" );
    //创建Multipart的容器
    Multipart multipart = new MimeMultipart();
    multipart.addBodyPart(mainBody);
    // 被转发的文字邮件体部分
    MimeBodyPart originBody = new MimeBodyPart();
    originBody.setDataHandler(msg.getDataHandler());
    // 添加到Multipart容器
    multipart.addBodyPart(originBody);
    forward.setContent(multipart);
    forward.saveChanges();

    Transport ts = session.getTransport("smtp");
    try {
        ts.connect(email, password);
        ts.sendMessage(forward, forward.getAllRecipients());
    } finally {
        ts.close();
    }
}
```

#### 3. 删除已发送邮件

读取`已发送`邮件，删除刚刚转发的邮件

```java
String subject = "Fwd: xxx";
Folder sentFolder = store.getFolder("已发送");
sentFolder.open(Folder.READ_WRITE);
Message[] messages = sentFolder.getMessages();
for(Message m: messages){
    if(subject.equals(m.getSubject())){
        m.setFlag(Flags.Flag.DELETED, true);
    }
}
sentFolder.close(true);
```