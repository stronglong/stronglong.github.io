---
layout: post
title: 会员卡对接(error)
description: 2016-06-01| 项目需要对接会员卡，主要记录绑定线下会员卡的过程
category: project
---

## 概述
此次会员卡对接，采用HTTP协议方式对接。对接主要内容为如果也有线下会员卡，则可以在线绑定会员卡；如果么有，则可以在线申请会员卡。实时查询积分和会员卡消费记录。
对接过程中记录点很多，这里主要记录大概过程，力求自己能明白就好。

###老会员绑定线下会员卡
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
            String ACCESS_TOKEN = getAccessToken();
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
此次对接中，使用redis存放access_token，同时存放第一次存入access_token 的时间，下次使用之前先判断是否在有效期内，在则使，否则重新获取access_token，并存入redis供下次使用。

	private String getAccessToken() throws Exception{
        Response<String> response = userCardReadService.getAccessToken();
        if(!response.isSuccess()){
            log.error("find access token failed,cause:{}",response.getError());
            throw new ServiceException(response.getError());
        }
        return response.getResult();
    }
    ——————————————————————————————————————————————————
	 @Override
    public Response<String> getAccessToken() {
        try{
            Optional<String> access = userCardRedisDao.getAccessToken(ACCESS_TOKEN);
            Optional<String> time = userCardRedisDao.getTime(ACCESS_TOKEN_TIME);
            if(access.isPresent() && time.isPresent()){
                Date old = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").parse(time.get());
                Date now = new Date();
                int hours = Hours.hoursBetween(new DateTime(old),new DateTime(now)).getHours() % 60 ;
                int minutes =  Minutes.minutesBetween(new DateTime(old),new DateTime(now)).getMinutes() % 60 ;
                int total = hours > 0 ? hours * 60 + minutes : minutes;
                if(total <= 115){
                    return Response.ok(userCardRedisDao.getAccessToken(ACCESS_TOKEN).get());
                }
                return Response.ok(getToken());
            }
            return Response.ok(getToken());
        } catch (Exception e){
            log.error("failed to get access token ,cause:{}",Throwables.getStackTraceAsString(e));
            return Response.fail(e.getMessage());
        }
    }

`UserCardRedisDao`：存入access_token，只存2个小时，后被覆盖。

	@Repository
	public class UserCardRedisDao {
    @Autowired
    private JedisTemplate jedisTemplate;
    public void setAccessToken(final String key, final String value){
        jedisTemplate.execute(new JedisTemplate.JedisActionNoResult() {
            @Override
            public void action(Jedis jedis) {
                jedis.set(key,value);
            }
        });
    }
    public void setTime(final String key,final String time){
        jedisTemplate.execute(new JedisTemplate.JedisActionNoResult() {
            @Override
            public void action(Jedis jedis) {
                jedis.set(key,time);
            }
        });
    }
    public Optional<String> getAccessToken(final String key){
        String value  = jedisTemplate.execute(new JedisTemplate.JedisAction<String>() {
            @Override
            public String action(Jedis jedis) {
                return jedis.get(key);
            }
        });
        return Optional.fromNullable(Params.trimToNull(value));
    }
    public Optional<String> getTime(final String key){
        String time  = jedisTemplate.execute(new JedisTemplate.JedisAction<String>() {
            @Override
            public String action(Jedis jedis) {
                return jedis.get(key);
            }
        });
        return Optional.fromNullable(Params.trimToNull(time));
    }
	}

存入access_token的时候同时存入了当前时间，以供计算时间差用。
得到access_token之后，即封装接口参数，采用POST方式连接，参数需要以json格式。
如果链接成功，返回json格式的字符串。按需要取其中的数据即可。

<font color=#A9A9A9>对接还有新会员注册线下会员卡、查询会员卡所有积分和查询会员卡消费小票，过程和注册如出一辙，改动的只是接口和参数。SO这里就不记录了。</font>




[StrongL]:    http://stronglong.me  "StrongL"
