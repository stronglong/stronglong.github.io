---
layout: post
title: EventBus初体验
description: 2016-03-30 | 以更新生鲜商品的库存信息为例
category: blog
---

## 概述
因为不太会用EventBus，所以在项目之中，也只能参考前辈的代码，模仿着完成既定的需要。在这里简短的记录一下，利用event处理问题的过程。不了解具体是怎么实现的，所以暂时记录下来，以便后续使用。

###CoreEventDispatcher
事件分发器CoreEventDispatcher

	package io.terminus.parana.common.event;
	import com.google.common.eventbus.AsyncEventBus;
	import com.google.common.eventbus.EventBus;
	import lombok.extern.slf4j.Slf4j;
	import org.springframework.beans.factory.annotation.Autowired;
	import org.springframework.context.ApplicationContext;
	import org.springframework.stereotype.Component;
	import javax.annotation.PostConstruct;
	import java.util.Map;
	import java.util.concurrent.Executors;

	//非泛型实现
	@Slf4j @Component
	public class CoreEventDispatcher {

	    protected final EventBus eventBus;

	    @Autowired
	    private ApplicationContext applicationContext;


	    public CoreEventDispatcher() {
	        this(Runtime.getRuntime().availableProcessors() + 1);
	    }

	    public CoreEventDispatcher(Integer threadCount) {
	        eventBus = new AsyncEventBus(Executors.newFixedThreadPool(threadCount));
	    }

	    @PostConstruct
	    public void registerListeners() {
	        Map<String, EventListener> listeners = applicationContext.getBeansOfType(EventListener.class);
	        for(EventListener eventListener : listeners.values()) {
	            eventBus.register(eventListener);
	        }
	    }

		/**
	     * 发布事件
	     */
	    public void publish(Object event) {
	        log.info("publish an event({})", event);
	        eventBus.post(event);
	    }
	}

###EventListener
事件监听器，具体事件监听需实现该接口

	package io.terminus.parana.common.event;
	public interface EventListener{}

###SuibaoItemUpEvent
事件携带数据

	package com.suibao.item.event;
	import com.suibao.item.module.FreshImportData;
	import io.terminus.pampas.common.BaseUser;
	import io.terminus.parana.shop.model.Shop;
	import lombok.Data;
	import java.util.List;

	@Data
	public class SuibaoItemsUpEvent {

	    private BaseUser user ;
	    private Shop shop;
	    private List<FreshImportData> freshImportDatas;

	    public SuibaoItemsUpEvent(BaseUser user,Shop shop,List<FreshImportData> freshImportDatas){
	        this.user = user;
	        this.shop = shop;
	        this.freshImportDatas = freshImportDatas;
	    }
	}

###SuibaoItemsListener
具体的事件监听

	package com.suibao.web.controller.event；
	import com.google.common.collect.Lists;
	import com.google.common.eventbus.Subscribe;
	import com.suibao.item.event.SuibaoItemsUpEvent;
	import com.suibao.item.module.FreshImportData;
	import com.suibao.item.service.SuibaoItemReadService;
	import io.terminus.pampas.common.BaseUser;
	import io.terminus.pampas.common.Response;
	import io.terminus.parana.category.model.Spu;
	import io.terminus.parana.common.event.EventListener;
	import io.terminus.parana.item.dto.ItemDto;
	import io.terminus.parana.item.dto.SkuWithLvPrice;
	import io.terminus.parana.item.model.*;
	import io.terminus.parana.item.service.ItemWriteService;
	import io.terminus.parana.shop.model.Shop;
	import lombok.extern.slf4j.Slf4j;
	import org.springframework.beans.factory.annotation.Autowired;
	import org.springframework.stereotype.Component;

	import java.util.List;
	import java.util.Objects;

	@Slf4j
	@Component
	public class SuibaoItemsListener implements EventListener {
	    @Autowired
	    private SuibaoItemReadService itemReadService;
	    @Autowired
	    private ItemWriteService itemWriteService;

	    @Subscribe
	    public void onSuitaoItems(SuibaoItemsUpEvent suibaoItemsUpEvent){
	        BaseUser user = suibaoItemsUpEvent.getUser();
	        Shop shop = suibaoItemsUpEvent.getShop();
	        List<FreshImportData> freshImportData = suibaoItemsUpEvent.getFreshImportDatas();
	        for(FreshImportData fresh : freshImportData){
	            if(Objects.equals(shop.getId() , fresh.getShopId())){
	                //只允许更改自家店铺库存
	                updateFresh(user , shop, fresh);
	            }
	        }
	    }

	    /**
	     * 更新生鲜商品的库存信息
	     * @param user              用户
	     * @param shop              店铺
	     * @param freshImportData   生鲜商品数据
	     * @return  Boolean
	     */
	    private Boolean updateFresh(BaseUser user , Shop shop, FreshImportData freshImportData){
	        ItemDto itemDto = new ItemDto();

	        Response<Spu> spuRes = itemReadService.querySpuByUpc(freshImportData.getSpuCode());
	        if(!spuRes.isSuccess() || spuRes.getResult() == null){
	            log.error("Query spu failed, spuCode={}", freshImportData.getSpuCode());
	            return false;
	        }

	        List<Long> spuList = Lists.newArrayList();
	        spuList.add(spuRes.getResult().getId());
	        Response<List<Item>> itemRes = itemReadService.findItemsByIds(spuList, shop.getId());
	        if(!itemRes.isSuccess() || itemRes.getResult().isEmpty()){
	            log.error("Query item failed, spuCode={}", freshImportData.getSpuCode());
	            return false;
	        }

	        Item item = itemRes.getResult().get(0);
	        itemDto.setItem(item);
	        //更改库存数据
	        item.setStockQuantity(freshImportData.getStockQuantity());
	        //更改商品名称
	        item.setName(freshImportData.getItemName());

	        //是否需要状态更改
	        if(!Objects.equals(item.getStatus() , freshImportData.getUpItem())){
	            List<Long> itemIds = Lists.newArrayList();
	            itemIds.add(item.getId());
	            item.setStatus(freshImportData.getUpItem());
	            Response<Boolean> upStatusRes = itemWriteService.updateItemsStatus(user , item.getStatus(), itemIds);
	            if(!upStatusRes.isSuccess()){
	                log.error("Update item status failed with user={}, status={}, itemIds={}", user, item.getStatus(), itemIds);
	                return false;
	            }
	        }

	        Response<ItemDetail> detailRes = itemReadService.findItemDetailByItemId(item.getId());
	        if(!detailRes.isSuccess()){
	            log.error("Query item detail info failed, itemId={} error code={}", item.getId(), detailRes.getError());
	            return false;
	        }
	        itemDto.setItemDetail(detailRes.getResult());

	        Response<List<Sku>> skuRes = itemReadService.findSkusByItemId(item.getId());
	        if(!skuRes.isSuccess() || skuRes.getResult().isEmpty()){
	            log.error("Query sku info failed, itemId={} error code={}", item.getId(), skuRes.getError());
	            return false;
	        }

	        List<SkuWithLvPrice> skuWithLvPriceList = Lists.newArrayList();
	        for(Sku sku : skuRes.getResult()){
	            //同步sku的库存信息
	            sku.setStockQuantity(freshImportData.getStockQuantity());

	            SkuWithLvPrice skuWithLvPrice = new SkuWithLvPrice();
	            skuWithLvPrice.setSku(sku);

	            SkuPrice skuPrice = new SkuPrice();
	            skuPrice.setPrice(sku.getPrice());
	            skuPrice.setLv(0);
	            skuWithLvPrice.setPrices(Lists.<SkuPrice>newArrayList(skuPrice));
	            skuWithLvPriceList.add(skuWithLvPrice);
	        }

	        itemDto.setSkus(skuWithLvPriceList);
	        itemDto.setShipFee(new ShipFee());

	        Response<Boolean> updateRes = itemWriteService.updateItem(user , itemDto);

	        if(!updateRes.isSuccess()){
	            log.error("Update item failed, user={}, itemDto={}, error code={}", user, itemDto, updateRes.getError());
	            return false;
	        }

	        return updateRes.getResult();
	    }
	}

###导入商品处使用event

	/**
     * 导入生鲜商品数据
     * @param file  数据文件
     * @return  Boolean
     */
    @RequestMapping(value = "/freshObject", method = RequestMethod.POST)
    @ResponseBody
    public Boolean syncFreshObject(MultipartFile file){

        try {
            BaseUser user = UserUtil.getCurrentUser();
            if(user == null){
                throw new JsonResponseException("user.not.login");
            }

            if(!Objects.equals(user.getType() , User.TYPE.SELLER.toNumber())){
                throw new JsonResponseException("user.id.invalid");
            }

            Response<Shop> shopRes = shopReadService.findByUserId(user.getId());
            if(!shopRes.isSuccess()){
                log.error("Query shop info failed, userId={}", user.getId());
                throw new JsonResponseException(shopRes.getError());
            }

            List<FreshImportData> freshList = FreshImportAnalyzer.analyze(file.getInputStream());

            if(!freshList.isEmpty()){
                `SuibaoItemsUpEvent event = new SuibaoItemsUpEvent(user,shopRes.getResult(),freshList)`;
                `coreEventDispatcher.publish(event)`;
            }
        }catch (Exception e){
            log.error("Analyze fresh excel failed, error code={}", com.google.api.client.repackaged.com.google.common.base.Throwables.getStackTraceAsString(e));
            throw new JsonResponseException("analyze.fresh.failed");
        }
        return true;
    }

[StrongL]:    http://stronglong.me  "StrongL"
