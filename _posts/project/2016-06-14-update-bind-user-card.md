---
layout: post
title: 会员卡对接(update)
description: 2016-06-14| 更新上次会员卡对接，不走redis，直接access_token
category: project
---

## 概述
这次update会员卡对接，是因为上次对接出现了error，error出的也太low，自己打自己脸。文档理解有误，把简单的往复杂处想。最后得出一结论：慎重看文档，一切应从简。

###还是老会员绑定线下会员卡
先给出代码，后做解释：

	/**
     * 老会员绑定线下岁宝会员卡
     * @param cardNum 会员卡号
     * @param mobile 会员卡对应的手机号
     * @return
     */
    @RequestMapping(value = "/bind/user/card",method = RequestMethod.POST,produces = MediaType.APPLICATION_JSON_VALUE)
    @ResponseBody
    public String bingOldUserCard(@RequestParam("cardNum") String cardNum,
                                  @RequestParam("mobile") String mobile,
                                  @RequestParam(value = "code") String code,
                                  HttpServletRequest request){
        validateSmsCode(code,mobile,request);
        String HOST = configCenter.get(USER_CARD_HOST_IP);
        String PORT = configCenter.get(USER_CARD_HOST_PORT);
        String WEB_APP = configCenter.get(USER_CARD_WEB_APP);
        String APP_ID = configCenter.get(USER_CARD_APP_ID);
        String TOKEN = configCenter.get(USER_CARD_TOKEN);
        String COM_ID = configCenter.get(USER_CARD_BIND_ID);
        BaseUser user = UserUtil.getCurrentUser();
        if(user == null){
            throw new JsonResponseException(500,"user.not.login");
        }
        try {
            Response<UserCard> cardResponse = userCardReadService.queryByCardNum(cardNum);
            if(!cardResponse.isSuccess()){
                log.warn("find user card failed cause:{}",cardResponse.getError());
            }
            if(!Arguments.isNull(cardResponse.getResult())){
                throw new JsonResponseException(500,"user.card.is.exist");
            }
            String ACCESS_TOKEN = getToken();
            Map<String, Object> params = new HashMap<>();
            params.put("vcardno", cardNum);
            params.put("vphone", mobile);
            params.put("vaccount",user.getId());
            params.put("Wechatno", mobile);
            String data = getData(TOKEN,params);
            String url = "http://" + HOST + ":" + PORT + "/" + WEB_APP + "/cmd/" + COM_ID + "?appid=" + APP_ID + "&ACCESS_TOKEN=" + ACCESS_TOKEN;
            String body = HttpRequest.post(url).contentType("application/json").send(data).body();
            Map bingResult = mapper.readValue(body,Map.class);
            Map dataResult = (Map) bingResult.get("data");
            Map outVal = (Map) dataResult.get("outVal");
            Integer ret = (Integer) outVal.get("ret");
            if(Objects.equals(ret,0)){
                log.warn("binding user card failed,cause:{}",String.valueOf(outVal.get("retmsg")));
                throw new JsonResponseException(500,String.valueOf(outVal.get("retmsg")));
            }

            UserCard userCard = new UserCard();
            userCard.setUserId(user.getId());
            userCard.setCardNum(cardNum);
            userCard.setStatus(UserCard.Status.BINDING.value());
            userCard.setPhoneNum(mobile);
            Response<Boolean> response = userCardWriteService.createUserCard(userCard);
            if (!response.isSuccess()) {
                log.info("create binding user card info failed,cause:{}", response.getError());
                throw new JsonResponseException(500, response.getError());
            }
            return body;
        } catch (Exception e){
            log.error("bind user card failed,cause:{}",Throwables.getStackTraceAsString(e));
            throw new JsonResponseException(500,e.getMessage());
        }
    }

`validateSmsCode(code,mobile,request)`方法校验手机验证码

	/**
     * 校验手机验证码
     * @param code    输入的验证码
     * @param request 请求
     */
    private void validateSmsCode(String code, String mobile, HttpServletRequest request) {
        HttpSession session = request.getSession(true);
        String codeInSession = (String) session.getAttribute("code");
        checkArgument(notEmpty(codeInSession), "sms.token.error");
        String expectedCode = AT_SPLITTER.splitToList(codeInSession).get(0);
        checkArgument(equalWith(code, expectedCode), "sms.token.error");
        String expectedMobile = AT_SPLITTER.splitToList(codeInSession).get(2);
        checkArgument(equalWith(mobile, expectedMobile), "sms.token.error");

        // 如果验证成功则删除之前的code
        session.removeAttribute("code");
    }

参数：
	`HOST`：对接主机IP、`PORT`：端口号、`WEB_APP`：应用、`APP_ID`：分配的应用ID、`TOKEN`：消息令牌、`COM_ID`：接口编号、`ACCESS_TOKNE`：调用接口凭证

`UserCard`会员卡对象信息

	@Data
	public class UserCard implements Serializable {
    private static final long serialVersionUID = -7793707951734257338L;
    private Long id;
    private Long userId;            //用户编号
    private String realName;        //真实姓名
    private String cardNum;         //岁宝卡号
    private String identityCard;    //绑定的证件号
    private String phoneNum;        //会员卡绑定的手机号
    private Integer status;         //状态(-1:未绑定, 0:待绑定系统帐户, 1:已绑定)
    private Date createdAt;         //创建时间
    private Date updatedAt;         //更新时间

###封装参数为json、取得sign签名
sign签名获得流程：
![get sign](/images/other/get-sign.png)
代码实现如下

	 private String getData(String token , Map<String,Object> params){
        Map<String,Object> map = new HashMap<>();
        String timestamp = String.valueOf(System.currentTimeMillis());
        map.put("timestamp",timestamp);
        map.put("params",params);
        map.put("sign",getSign(token,timestamp));
        return JsonMapper.nonEmptyMapper().toJson(map);
    }
    private String getSign(String token,String timestamp) {
        if (token.compareTo(timestamp) > 0) {
            timestamp = timestamp + token;
            return SHA1(timestamp);
        }
        token = token + timestamp;
        return SHA1(token);
    }
`SHA1()`是一个Sha1加密工具，网上copy的代码。

###获取Access_token
`getAccessToken()`方法实时获取acdess_token，access_token是datahub的调用命令的全局唯一票据，接入系统调用接口命令时都需要使用acces_token。access_token的有效期目前为2个小时，需定时刷新，重复获取将导致上次获取的access_token失效。
`此次对接中，使用redis存放access_token，同时存放第一次存入access_token 的时间，下次使用之前先判断是否在有效期内，在则使，否则重新获取access_token，并存入redis供下次使用。`这是上次的解决方案，这次不使用。
	直接每次都获取access_token

	 private String getToken() throws Exception{
        String HOST = configCenter.get(USER_CARD_HOST_IP);
        String PORT = configCenter.get(USER_CARD_HOST_PORT);
        String WEB_APP = configCenter.get(USER_CARD_WEB_APP);
        String APP_ID = configCenter.get(USER_CARD_APP_ID);
        String SECRET = configCenter.get(USER_CARD_SECRET);
        String url = "http://"+HOST+":"+PORT+"/"+WEB_APP+"/access_token?appid="+APP_ID+"&secret="+SECRET;
        log.info("获取access_token url:"+url);
        String body = HttpRequest.get(url).body();
        log.info("获取access_token返回数据:"+body);
        ObjectMapper mapper = new ObjectMapper();
        Map data =mapper.readValue(body,Map.class);
        String access_token = (String) data.get("access_token");
        return access_token;
    }


[StrongL]:    http://stronglong.me  "StrongL"
