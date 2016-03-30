---
layout: post
title: 项目中redis简易使用
description: |2016-03-30|redis存入一个id，在使用时去除此id
category: project
---

## 概述
新建一个redis dao接口类WechatRedisDao，里面编写两个方法：`setSaveOpenId(final Long userId,final String openId)`、`getSaveOpenId(final Long userId)`.

###WechatRedisDao

	@Repository
	public class WechatRedisDao {
	    @Autowired
	    private JedisTemplate jedisTemplate;

	    /**
	     * 保存openId
	     * @param userId 用户id
	     * @param openId 第三方用户id
	     */
	    public void setSaveOpenId(final Long userId,final String openId){
	        jedisTemplate.execute(new JedisTemplate.JedisActionNoResult() {
	            @Override
	            public void action(Jedis jedis) {
	                jedis.set(User.keyOfSaveOpenId(userId),openId);
	            }
	        });
	    }

	    /**
	     *  取得保存的openId
	     * @param userId
	     * @return
	     */
	    public Optional<String> getSaveOpenId(final Long userId){
	        String openId = jedisTemplate.execute(new JedisTemplate.JedisAction<String>() {
	            @Override
	            public String action(Jedis jedis) {
	                return jedis.get(User.keyOfSaveOpenId(userId));
	            }
	        });
	        return Optional.fromNullable(Params.trimToNull(openId));
	    }
	}


###保存openId的key
需要设置一个key，放置在redis之中，用做key-value绑定

	public static final String keyOfSaveOpenId(Long userId){
		return "pay:" + userId + ":Wechat:openId";
	}

###取出redis中放置的id
在需要取出的地方注入

	@Autowired
	private WechatRedisDao wechatRedisDao;

取出openId

	Optional<String> data = wechatRedisDao.getSaveOpenId(payment.getUserId());
	String openId = null;
	if (data.isPresent()){
		openId = data.get();
	}










[StrongL]:    http://stronglong.me  "StrongL"
