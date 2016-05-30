---
layout: post
title: Send Email
description: 2016-05-30 | 单人简易邮件发送记录
category: project
---

## 概述
本来是一个自动发送邮件的需求，这里就不完全记录了。简要记录一下邮件的发送过程。

##代码
###service

	 /**
     * 发送邮件
     * @param subject 主体
     * @param content 内容
     * @param to 收件人
     * @param checkSubscribe 校验用户订阅关系
     * @return 成功TRUE,反之FALSE
     */
    Response<Boolean> send(String subject, String content, String to, Boolean checkSubscribe);

###serviceImpl

	    private static final String USER="stronglong1024@163.com";
    private static final String PASSWORD="strong525529";
    private static final String FROM="stronglong1024@163.com";

    @Override
    public Response<Boolean> send(String subject, String content, String to, Boolean checkSubscribe) {
        Response<Boolean> response = new Response<Boolean>();
        Properties pro = new Properties();
        pro.put("mail.smtp.host","smtp.163.com");
        pro.put("mail.smtp.port","25");
        pro.put("mail.debug","true");
        pro.put("mail.smtp.auth","true");
        pro.put("mail.transport.protocol", "smtp");
        Session session = Session.getInstance(pro);
        try{
            Message message = new MimeMessage(session);
            message.setFrom(new InternetAddress(FROM));
            message.setRecipient(Message.RecipientType.TO,new InternetAddress(to));
            message.setSubject(subject);
            message.setContent(content,"text/html;charset=utf-8");
            message.setSentDate(new Date());
            message.saveChanges();

            Transport transport = session.getTransport();
            transport.connect(USER,PASSWORD);
            transport.sendMessage(message,message.getAllRecipients());
            transport.close();
            response.setResult(Boolean.TRUE);
            return response;
        } catch (Exception e){
            log.info("fail to send mail to user:{} subject:{},cause:{}",to,subject, Throwables.getStackTraceAsString(e));
            response.setError(e.getMessage());
            return response;
        }
    }

###sendEmail

	  private Boolean sendEmail(List<Long> userIds) throws Exception {
        final String subject = "用户余额检查";
        final String to = "1101900763@qq.com";
        final String content = "用户ID为"+userIds.toString()+"的余额存在异常情况!";
        io.terminus.pampas.common.Response<Boolean> response = dispatchEmailService.send(subject, content, to, false);
        if (!response.isSuccess()) {
            throw new ServiceException(response.getError());
        }
        return Boolean.TRUE;
    }

此处的`List<Long> userIds`其实就是需要发送的内容，只不过它是list的。

##总结
发送邮件就上面的代码，接下来就是提前准备好发送数据，调用此方法，即可发送邮件。
与其说是记录，不如说是直接笔记，主要是方面以后使用，保不准哪天又有同样或类似的需求了。


[StrongL]:    http://stronglong.me  "StrongL"
