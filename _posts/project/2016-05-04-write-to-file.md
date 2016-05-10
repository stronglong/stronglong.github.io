---
layout: post
title: 商品名称写入user.dict，以供搜索使用
description: 2016-05-04 | 根据指定格式导出写入文件，只保留汉字后跟搜索权重
category: project
---

## 概述
本来这次导出应该使用Python来实现，由于本人对Python不熟，so只能使用Java来写了。导出需要用到一个方法去获取需要导出的数据，此处不编写，直接使用已经写好的method。

###流程
取出需要导出的数据，按照导出的数据格式要求对数据进行处理，拆分字符串，去除特殊字符只留中文汉字，且去掉重复数据。使用Guava Files进行写入文件，文件名为user.dict

###代码
####现成Dao

	/**
     * 根据条件查询商品列表
     * @param criteria 查询条件
     * @return 商品列表
     */
	public List<Item> loadsBy(Map<String,Object> criteria){
		return getSqlSession().selectList(sqlId("loadsBy"),criteria);
	}


####service
编程一个service得到数据，调用相应方法处理数据，最后调用写入文件方法写数据。

service

	Response<Boolean> getItemsNames();

serviceImpl

	 @Override
    public Response<Boolean> getItemsNames() {
        Response<Boolean> response = new Response<>();
        try{
            Map<String,Object> map = new HashMap<>();
            map.put("status",Item.Status.ON_SHELF.value());
            List<Item> items = itemDao.loadsBy(map);
            Set<String> names = Sets.newHashSet();
            for(Item item : items ){
                if(!Strings.isNullOrEmpty(item.getName())){
                    names.add(item.getName().trim());
                }
            }
            Set<String> contents = new HashSet<>();
            for(String str : names){
              contents.addAll(resolveDatas(str));
            }
            for(String content : contents){
                writeToFile(content);
            }
            response.setResult(Boolean.TRUE);
            log.error("********************************");
        } catch (Exception e){
            log.error("failed to get item names,cause:{}",Throwables.getStackTraceAsString(e));
            response.setError(e.getMessage());
        }
        return response;
    }

####Other Method

	/**
     * 操作商品名称
     * @param str
     * @return
     */
    private  Set<String> resolveDatas(String str){
        Set<String> set = new HashSet<>();
        if(!Strings.isNullOrEmpty(str)){
            int len = str.length();
            for (int i = 0; i < len; i++) {
                for (int j = i + 1; j <= len; j++) {
                    if(!Strings.isNullOrEmpty(dealStr(String.valueOf(str.substring(i,j))))) {
                        set.add(dealStr(String.valueOf(str.substring(i, j))).concat(" 5"));
                    }
                }
            }
        }
        return set;
    }

	  /**
     * 只保留中文
     * @param str
     * @return
     */
    private  String  dealStr(String str) {
        StringBuffer sb = new StringBuffer();
        for (int i = 0; i < str.length(); i++) {
            if ((str.charAt(i) + "").getBytes().length > 1) {
                sb.append(str.charAt(i));
            }
        }
        String ss = String.valueOf(sb).replaceAll("（","").replaceAll("）","").replaceAll("【","").replaceAll("】","");
        return ss.trim();
    }



####写入method

	 private void writeToFile(String content) {
        File fileName = new File("/Users/terminus/Desktop/user.dict");
        if (!fileName.exists()) {
            try {
                fileName.createNewFile();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        try {
            Files.append(content + "\r\n", fileName, Charsets.UTF_8);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

####controller
最后还有一个controller

	/**
     * 写入文件
     * @return
     */
    @RequestMapping(value = "/write-items-name",method = RequestMethod.GET)
    @ResponseBody
    public Boolean writeItemsName(){
        try{
            return RespHelper.or500(suibaoItemReadService.getItemsNames());
        } catch (Exception e){
            log.error("failed to write to file,cause:{}", Throwables.getStackTraceAsString(e));
            throw new JsonResponseException(500,e.getMessage());
        }
    }

最后写入文件的内容大体如下图所示：
![write to file](/images/other/write-to-file.png)











[StrongL]:    http://stronglong.me  "StrongL"
