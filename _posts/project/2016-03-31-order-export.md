---
layout: post
title: Excel导出数据
description: 2016-03-31 | 以订单导出为例
category: project
---

## 概述
订单信息导出excel

###编写orderExportFields.yaml文件
excel导出字段

	 #每个项目都需要定制化一个orderExportFields.yaml文件
	 #fieldName:导出字段名称
	 #needTrans:导出字段是否需要翻译(默认是order.id 这种形式的)

	exportFields:
	  - fieldName: order.id
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.buyer.id
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.buyer.name
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.seller.id
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.seller.name
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.shop.id
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.shop.name
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.type
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.status
	    needTrans: true
	    contentTrans: true
	  - fieldName: order.fee
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.origin.fee
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.ship.fee
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.discount
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.pay.type
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.channel
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.deliver.type
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.created.at
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.cancel.at
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.buyer.note
	    needTrans: true
	    contentTrans: false
	  - fieldName: item.id
	    needTrans: true
	    contentTrans: false
	  - fieldName: item.name
	    needTrans: true
	    contentTrans: false
	  - fieldName: tradeinfo.name
	    needTrans: true
	    contentTrans: false
	  - fieldName: tradeinfo.phone
	    needTrans: true
	    contentTrans: false
	  - fieldName: tradeinfo.address
	    needTrans: true
	    contentTrans: false
	  - fieldName: deliver.coupon
	    needTrans: true
	    contentTrans: false
	  - fieldName: order.picker.name
	    needTrans: true
	    contentTrans: false
	  - fieldName: deliver.name
	    needTrans: true
	    contentTrans: false
	  - fieldName: dispatch.id
	    needTrans: true
	    contentTrans: false
	  - fieldName: delivery.created.at
	    needTrans: true
	    contentTrans: false
	  - fieldName: delivery.done.at
	    needTrans: true
	    contentTrans: false

`fieldName`：需要导出字段名
`needTrans`：该字段是否需要翻译
`contentTrans`：该字段对应的内容是否需要翻译
fieldName对应的类似`order.id`，需要编写对应的翻译文件(中文|英文)

###ExportFields
ExportFields类用于存放取出的导出字段

	@Data
	public class ExportFields implements Serializable {
	    private static final long serialVersionUID = 4026095984436270393L;
	    List<ExportField> exportFields;
	    @Data
	    public static class ExportField {
	        private String fieldName;
	        private Boolean needTrans;
	        private Boolean contentTrans;
	        public boolean needTrans() {
	            return needTrans;
	        }
	        public boolean contentTrans(){
	            return contentTrans;
	        }
	    }
	}


###编写getExportFieldFromYaml()
方法用于取出yaml文件里的导出字段

	 private List<ExportFields.ExportField> getExportFieldFromYaml() throws Exception {
	        String file = Resources.toString(Resources.getResource("auth/orderExportFields.yaml"), Charsets.UTF_8);
	        ExportFields fields = new Yaml().loadAs(file, ExportFields.class);
	        return fields.getExportFields();
	    }


###导出Controller

	@Slf4j
	@Controller
	@RequestMapping("/api/orders")
	public class NewOrderExprots {

    @Autowired
    private SuibaoOrderExporter suibaoOrderExporter;

    @Autowired
    private MessageSources messageSources;

    @Autowired
    private OrderExportFactory orderExportFactory;

    private static final String ORDER_EXCEL_NAME = "ExportOrders";

    /**
     * 导出订单
     */
    @RequestMapping(value = "/new-export-xls", method = RequestMethod.GET)
    @ResponseBody
    public void exportXls(
            @RequestParam(value = "statuses",required = false) String statuses,
            @RequestParam(value = "status", required = false) Integer status,
            @RequestParam(value = "orderId", required = false) Long orderId,
            @RequestParam(value = "nickName", required = false) String nickName,
            @RequestParam(value = "mobile", required = false) String mobile,
            @RequestParam(value = "email", required = false) String email,
            @RequestParam(value = "shopName", required = false) String shopName,
            @RequestParam(value = "start", required = false) String start,
            @RequestParam(value = "end", required = false) String end,
            HttpServletResponse response) {

        BaseUser baseUser = UserUtil.getCurrentUser();
        Response<List<OrderExportDto>> resp = suibaoOrderExporter.loadByAllParams(baseUser, statuses,status, orderId, nickName,
                mobile, email, shopName, start, end);
        if (!resp.isSuccess()) {
            throw new JsonResponseException(500, resp.getError());
        }

        try {
            String xlsFileName = URLEncoder.encode(ORDER_EXCEL_NAME, "UTF-8") + ".xls";
            response.setContentType("application/x-download");
            String headerKey = "Content-Disposition";
            String headerValue = String.format("attachment; filename=\"%s\"", xlsFileName);
            response.setHeader(headerKey, headerValue);

            Workbook wb = new SXSSFWorkbook();
            Sheet s = wb.createSheet();

            // yaml 文件订单excel所有的列
            List<ExportFields.ExportField> fields = getExportFieldFromYaml();

            // title row
            Row titleRow = s.createRow(0);
            for (int i=0; i<fields.size(); ++i) {
                Cell cell = titleRow.createCell(i);
                ExportFields.ExportField field = fields.get(i);
                if (field.needTrans()) {
                    cell.setCellValue(messageSources.get(field.getFieldName()));
                }else {
                    cell.setCellValue(field.getFieldName());
                }
            }
            // content rows
            int rowNum = 1;
            for (OrderExportDto exportDto : resp.getResult()) {
                Row contentRow = s.createRow(rowNum);
                Map<String, String> fieldsMap = orderExportFactory.composeOrderExportFieldsMap(exportDto);
                for (int i=0; i<fields.size(); ++i) {
                    ExportFields.ExportField field = fields.get(i);
                    Cell cell = contentRow.createCell(i);
                    if (field.contentTrans()) {
                        cell.setCellValue(messageSources.get(fieldsMap.get(field.getFieldName())));
                    } else {
                        cell.setCellValue(fieldsMap.get(field.getFieldName()));
                    }
                }
                ++rowNum;
            }
            wb.write(response.getOutputStream());

        } catch (Exception e) {
            log.error("export order failed, cause:{}", Throwables.getStackTraceAsString(e));
            throw new JsonResponseException(500, "order.export.fail");
        }
    }

`ORDER_EXCEL_NAME`：导出的excle名称
导出使用Apache的poi包，使用：`org.apache.poi.xssf.streaming.SXSSFWorkbook`.

###DTO
组装导出数据
OrderExportDto

	@Data
	public class OrderExportDto implements Serializable {
	    private static final long serialVersionUID = -1550854230391323682L;
	    private Order order;
	    private OrderItem orderItem;
	    private OrderTrack orderTrack;
	    private OrderExtra orderExtra;
	    private Spu spu;
	    private Sku sku;
	    private Item item;
	    private OrderItemRefundTrack orderItemRefundTrack;
	}

SuibaoOrderExportDto

	@Data
	public class SuibaoOrderExportDto extends OrderExportDto {
	    private static final long serialVersionUID = -7303116660306471420L;
	    private OrderDiscountDetail orderDiscountDetail;
	    private OrderPickInfo orderPickInfo;
	    private DispatchDto dispatchDto;
	}

###SuibaoOrderExporter
编写loadByAllParams()方法，取出需要导出的数据，给出大体的结构，具体根据实际需求进行。

	@Component @Slf4j @Primary
	public class SuibaoOrderExporter {

    @Autowired
    private OrderPickReadService orderPickReadService;

    @Autowired
    private OrderReadService orderReadService;

    @Autowired(required = false)
    private DispatchReadService dispatchReadService;

    @Override
    public Response<List<OrderExportDto>> loadByAllParams(BaseUser baseUser, String statuses,Integer status, Long orderId, String nickName, String mobile,
                                                          String email, String shopName,
                                                          String start, String end) {
	        Response<Paging<RichOrderSellerView>> viewResp = getDataViaUser(baseUser, statuses,status, 1, 100, orderId, nickName, mobile, email, shopName, start, end);
	        if (!viewResp.isSuccess()) {
	            return Response.fail(viewResp.getError());
	        }
	        try {
	            List<RichOrderSellerView> views = Lists.newArrayList();

	            Long total = viewResp.getResult().getTotal();
	            int size = 5000;
	            for (int i = 1; i * size - size < total; ++i) {
	                viewResp = getDataViaUser(baseUser, statuses,status, i, size, orderId, nickName, mobile, email, shopName, start, end);
	                if (!viewResp.isSuccess()) {
	                    log.warn("paging orders failed, error:{}", viewResp.getError());
	                    return Response.fail(viewResp.getError());
	                }
	                views.addAll(viewResp.getResult().getData());
	            }
            List<OrderExportDto> exports = Lists.newArrayList();

            List<Long> orderIds = new ArrayList<>();
            for (RichOrderSellerView view: views){
                orderIds.add(view.getRichOrder().getOrder().getId());
            }

            Response<List<OrderTrack>> orderTrackResp = orderReadService.findOrderTrackByOrderIds(orderIds);
            if (!orderTrackResp.isSuccess()){
                log.warn("find order track failed,error:{}",orderTrackResp.getError());
                return Response.fail(orderTrackResp.getError());
            }
            List<OrderTrack> orderTracks= orderTrackResp.getResult();
            Map<Long,OrderTrack> groupOrderTrack = groupOrderTrackByOrderId(orderTracks);


            Response<List<OrderExtra>> resp =orderReadService.findOrderExtraByOrderIds(orderIds);
            if (!resp.isSuccess()){
                log.warn("find order extra failed,error:{}",resp.getError());
                return Response.fail(resp.getError());
            }
            List<OrderExtra> orderExtra = resp.getResult();
            Map<Long,OrderExtra> groupOrderExtra = groupOrderExtraByOrderId(orderExtra);

            io.terminus.common.model.Response<List<DispatchDto>> listResponse = dispatchReadService.loadsFullDispatchByOutOrderIds(orderIds);
            if(!listResponse.isSuccess()){
                log.warn("find dispatch dto failed ,error:{}",listResponse.getError());
                return Response.fail(listResponse.getError());
            }
            List<DispatchDto> dispatchDtos = listResponse.getResult();
            Map<Long,DispatchDto> groupDispatchDto = groupDispatchDtoByOrderId(dispatchDtos);

            List<RichOrderItem> rich = Lists.newArrayList();
            for(RichOrderSellerView view: views){
                rich.addAll(view.getRichOrder().getRichOrderItems());
            }

            List<Long> orderItemIds = new ArrayList<>();
            for (RichOrderItem richOrderItem : rich) {
                orderItemIds.add(richOrderItem.getOrderItem().getId());
            }

            Response<List<OrderDiscountDetail>> disResp = orderReadService.findOrderDiscountDetailByItemIds(orderItemIds);
            if (!disResp.isSuccess()){
                log.warn("find order discount detail failed,error:{}",disResp.getError());
                return Response.fail(disResp.getError());
            }
            List<OrderDiscountDetail> details = disResp.getResult();
            Map<Long,OrderDiscountDetail> groupOrderDiscountDetail = groupOrderDiscountDetailByItemIds(details);

            Response<List<OrderPickInfo>> info = orderPickReadService.queryByOrderIds(orderIds);
            if (!info.isSuccess()){
                log.warn("find order pick info failed,error:{}",info.getError());
                return Response.fail(info.getError());
            }
            List<OrderPickInfo> infos = info.getResult();
            Map<Long,OrderPickInfo> groupOrderPickInfo = groupPickInfoByOrderIds(infos);

            SuibaoOrderExportDto exportDto ;
            Long key;
            for (RichOrderSellerView view : views) {
                List<OrderExportDto> orderExportDto = Lists.newArrayList();
                for (RichOrderItem richOrderItem : view.getRichOrder().getRichOrderItems()) {
                    OrderItem orderItem = richOrderItem.getOrderItem();
                    exportDto = new SuibaoOrderExportDto();
                    exportDto.setOrderItem(orderItem);
                    exportDto.setOrder(view.getRichOrder().getOrder());
                    key = richOrderItem.getOrderItem().getOrderId();
                    exportDto.setOrderTrack(getOrderTrack(key,groupOrderTrack));
                    exportDto.setOrderExtra(getOrderExtra(key,groupOrderExtra));
                    exportDto.setDispatchDto(getDispachDto(key,groupDispatchDto));
                    exportDto.setOrderDiscountDetail(getOrderDiscountDetail(richOrderItem.getOrderItem().getId(),groupOrderDiscountDetail));
                    exportDto.setOrderPickInfo(getOrderPickInfo(key,groupOrderPickInfo));
                    orderExportDto.add(exportDto);
                }
                exports.addAll(orderExportDto);
            }
            return  Response.ok(exports);
        } catch (Exception e) {
            log.error("export order failed,cause:{}", Throwables.getStackTraceAsString(e));
            return Response.fail("order.export.fail");
        }
    }


    private Map<Long,DispatchDto> groupDispatchDtoByOrderId(List<DispatchDto> dispatchDtos){
        Map<Long,DispatchDto> result = new HashMap<>();
        for(DispatchDto dto : dispatchDtos){
            result.put(dto.getDispatch().getOutOrderId(),dto);
        }
        return result;
    }

    private Map<Long,OrderDiscountDetail> groupOrderDiscountDetailByItemIds(List<OrderDiscountDetail> details){
        Map<Long,OrderDiscountDetail> result = new HashMap<>();
        for (OrderDiscountDetail detail : details){
            result.put(detail.getOrderItemId(),detail);
        }
        return result;
    }

    private Map<Long,OrderPickInfo> groupPickInfoByOrderIds(List<OrderPickInfo> infos){
        Map<Long,OrderPickInfo> result = new HashMap<>();
        for (OrderPickInfo info : infos){
            result.put(info.getOrderId(),info);
        }
        return result;
    }

    private OrderPickInfo getOrderPickInfo(Long key,Map<Long,OrderPickInfo> map){
        return map.get(key);
    }
    private DispatchDto getDispachDto(Long key,Map<Long,DispatchDto> map){
        return map.get(key);
    }

    private OrderDiscountDetail getOrderDiscountDetail(Long key,Map<Long,OrderDiscountDetail> map){
        return map.get(key);
    }
	}

###OrderExportFactory
 每个项目都可以根据具体业务实现 composeOrderExportFieldsMap() 方法,
 每个composeXXX子方法都可以被重用, 并且可以定制化
 默认的,这个类会提供order, orderItem, orderExtra, orderTrack主要字段的map

	@Component
	public class OrderExportFactory {
    private static final DateTimeFormatter DTF = DateTimeFormat.forPattern("yyyy-MM-dd HH:mm:ss");
    /**
     * 所有的字段都组装成一个map, 生成excel时,根据orderExportField.yaml 中的fieldName, 获取对应的value
     *
     * key 都为英文,例如: order.id
     **/
    public Map<String, String> composeOrderExportFieldsMap(OrderExportDto orderExportDto) {
        Map<String, String> fieldsMap = new HashMap<>();
        composeOrder(fieldsMap, orderExportDto.getOrder());
        composeOrderItem(fieldsMap, orderExportDto.getOrderItem());
        composeOrderExtra(fieldsMap, orderExportDto.getOrderExtra());
        composeOrderTrack(fieldsMap, orderExportDto.getOrderTrack());
        return fieldsMap;
    }

    protected void composeOrder(Map<String, String> fieldsMap, Order order) {
        if (null != order.getId()) fieldsMap.put(ExportConstants.ORDER_ID, order.getId().toString());
        if (null != order.getBuyerId()) fieldsMap.put(ExportConstants.ORDER_BUYER_ID, order.getBuyerId().toString());
        if (!Strings.isNullOrEmpty(order.getBuyerName())) fieldsMap.put(ExportConstants.ORDER_BUYER_NAME, order.getBuyerName());
        if (null != order.getSellerId()) fieldsMap.put(ExportConstants.Order_SELLER_ID, order.getSellerId().toString());
        if (!Strings.isNullOrEmpty(order.getSellerName())) fieldsMap.put(ExportConstants.ORDER_SELLER_NAME, order.getSellerName());
        if (null != order.getShopId()) fieldsMap.put(ExportConstants.ORDER_SHOP_ID, order.getShopId().toString());
        if (!Strings.isNullOrEmpty(order.getShopName())) fieldsMap.put(ExportConstants.ORDER_SHOP_NAME, order.getShopName());
        if (null != order.getBusinessId()) fieldsMap.put(ExportConstants.ORDER_BUSINESS_ID, order.getBusinessId().toString());
        if (null != order.getType()) fieldsMap.put(ExportConstants.ORDER_TYPE, toStringOfOrderType(order.getType()));
        if (null != order.getStatus()) fieldsMap.put(ExportConstants.ORDER_STATUS, toStringOfOrderStatus(order.getStatus()));
        if (null != order.getFee()) fieldsMap.put(ExportConstants.ORDER_FEE, toYuan(order.getFee()).toString());
        if (null != order.getOriginFee()) fieldsMap.put(ExportConstants.ORDER_ORIGIN_FEE, toYuan(order.getOriginFee()).toString());
        if (null != order.getShipFee()) fieldsMap.put(ExportConstants.ORDER_SHIP_FEE, toYuan(order.getShipFee()).toString());
        if (null != order.getDiscount()) fieldsMap.put(ExportConstants.ORDER_DISCOUNT, toYuan(order.getDiscount()).toString());
        if (null != order.getPayType()) fieldsMap.put(ExportConstants.ORDER_PAY_TYPE, toStringOfPayType(order.getPayType()));
        if (null != order.getHasPaid()) fieldsMap.put(ExportConstants.HAS_ORDER_PAID, toStringOfPredicate(order.getHasPaid()));
        if (null != order.getHasComment()) fieldsMap.put(ExportConstants.HAS_ORDER_COMMENT, toStringOfPredicate(order.getHasComment()));
        if (null != order.getChannel()) fieldsMap.put(ExportConstants.ORDER_CHANNEL, toStringOfOrderChannel(order.getChannel()));
    }

    protected void composeOrderItem(Map<String, String> fieldsMap, OrderItem orderItem) {
        if (null != orderItem.getItemId()) fieldsMap.put(ExportConstants.ITEM_ID, orderItem.getItemId().toString());
        if (!Strings.isNullOrEmpty(orderItem.getItemName())) fieldsMap.put(ExportConstants.ITEM_NAME, orderItem.getItemName());
        if (null != orderItem.getType())
            fieldsMap.put(ExportConstants.ORDER_ITEM_TYPE, toStringOfOrderItemType(orderItem.getType()));
        if (null != orderItem.getStatus())
            fieldsMap.put(ExportConstants.ORDER_ITEM_STATUS, toStringOfOrderStatus(orderItem.getStatus()));
        if (null != orderItem.getPayType())
            fieldsMap.put(ExportConstants.ORDER_ITEM_PAY_TYPE, toStringOfPayType(orderItem.getPayType()));
        if (null != orderItem.getFee()) fieldsMap.put(ExportConstants.ORDER_ITEM_FEE, toYuan(orderItem.getFee()).toString());
        if (null != orderItem.getOriginFee()) fieldsMap.put(ExportConstants.ORDER_ITEM_ORIGIN_FEE, toYuan(orderItem.getOriginFee()).toString());
        if (null != orderItem.getShipFee()) fieldsMap.put(ExportConstants.ORDER_ITEM_SHIP_FEE, toYuan(orderItem.getShipFee()).toString());
        if (null != orderItem.getDiscount()) fieldsMap.put(ExportConstants.ORDER_ITEM_DISCOUNT, toYuan(orderItem.getDiscount()).toString());
        if (!Strings.isNullOrEmpty(orderItem.getOuterSkuId()))
            fieldsMap.put(ExportConstants.ORDER_ITEM_OUT_SKU_ID, orderItem.getOuterSkuId());
        if (null != orderItem.getQuantity()) fieldsMap.put(ExportConstants.ORDER_ITEM_QUANTITY, orderItem.getQuantity().toString());
    }

    protected void composeOrderExtra(Map<String, String> fieldsMap, OrderExtra orderExtra) {
        if (!Strings.isNullOrEmpty(orderExtra.getTradeInfo())) {
            UserTradeInfo tradeInfo =
                    JsonMapper.JSON_NON_DEFAULT_MAPPER.fromJson(orderExtra.getTradeInfo(), UserTradeInfo.class);
            if (null != tradeInfo) {
                if (!Strings.isNullOrEmpty(tradeInfo.getName())) fieldsMap.put(ExportConstants.TRADEINFO_NAME, tradeInfo.getName());
                if (!Strings.isNullOrEmpty(tradeInfo.getPhone())) fieldsMap.put(ExportConstants.TRADEINFO_PHONE, tradeInfo.getPhone());
                fieldsMap.put(ExportConstants.TRADEINFO_ADDRESS, toStringAddress(tradeInfo));
                if (!Strings.isNullOrEmpty(tradeInfo.getZip())) fieldsMap.put(ExportConstants.TRADEINFO_ZIP, tradeInfo.getZip());
            }
        }
        if (!Strings.isNullOrEmpty(orderExtra.getBuyerNotes())) fieldsMap.put(ExportConstants.ORDER_BUYER_NOTE, orderExtra.getBuyerNotes());
        if (!Strings.isNullOrEmpty(orderExtra.getSellerNotes())) fieldsMap.put(ExportConstants.ORDER_SELLER_NOTE, orderExtra.getSellerNotes());
    }

    protected void composeOrderTrack(Map<String, String> fieldsMap, OrderTrack orderTrack) {
        if (null != orderTrack.getCreatedAt()) fieldsMap.put(ExportConstants.ORDER_CREATED_AT, printDate(orderTrack.getCreatedAt()));
        if (null != orderTrack.getPaidAt()) fieldsMap.put(ExportConstants.ORDER_PAID_AT, printDate(orderTrack.getPaidAt()));
        if (null != orderTrack.getDeliverAt()) fieldsMap.put(ExportConstants.ORDER_DELIVER_AT, printDate(orderTrack.getDeliverAt()));
        if (null != orderTrack.getDoneAt()) fieldsMap.put(ExportConstants.ORDER_DONE_AT, printDate(orderTrack.getDoneAt()));
        if (null != orderTrack.getBuyerCancelAt()) fieldsMap.put(ExportConstants.ORDER_CANCEL_AT, printDate(orderTrack.getBuyerCancelAt()));
        if (!Strings.isNullOrEmpty(orderTrack.getWaybillNo())) fieldsMap.put(ExportConstants.ORDER_WAY_BILL_NO, orderTrack.getWaybillNo());
        if (!Strings.isNullOrEmpty(orderTrack.getLogisticsCompany()))
            fieldsMap.put(ExportConstants.ORDER_LOGISTIC_COMPANY, orderTrack.getLogisticsCompany());
    }



    protected String printDate(Date dte) {
        if (dte == null) {
            return null;
        }
        return DTF.print(new DateTime(dte));
    }

    protected String toStringOfOrderType(Integer type) {
        for (OrderType orderType : OrderType.values()) {
            if (Objects.equal(type, orderType.value())) {
                return orderType.toString();
            }
        }
        return null;
    }

    protected String toStringOfOrderStatus(Integer status) {
        Status orderStatus = Status.from(status);
        return orderStatus == null ? null : orderStatus.description;
    }

    protected String toStringOfOrderItemType(Integer type) {
        for (OrderItem.Type orderItemType : OrderItem.Type.values()) {
            if (Objects.equal(type, orderItemType.value())) {
                return orderItemType.description();
            }
        }
        return null;
    }

    protected Double toYuan(Integer fen) {
        return fen == null ? null : fen == 0 ? .0 : (double)fen/100;
    }

    protected Double toYuan(Long fen) {
        return fen == null ? null : fen == 0 ? .0 : (double)fen/100;
    }

    protected String toStringOfPayType(Integer type) {
        for (PaidType paidType : PaidType.values()) {
            if (Objects.equal(type, paidType.value())) {
                return paidType.toString();
            }
        }
        return null;
    }

    protected String toStringOfPredicate(Boolean predicate) {
        return predicate == null ? null : predicate ? "yes" : "no";
    }

    protected String toStringOfOrderChannel(Integer channel) {
        for (Order.CHANNEL chan : Order.CHANNEL.values()) {
            if (Objects.equal(channel, chan.value)) {
                return chan.desc;
            }
        }
        return null;
    }

    protected String toStringOfSkuAttributes(Sku sku) {
        return sku.getAttributeKey1() + ":" + sku.getAttributeName1() + "/" +
                sku.getAttributeKey2() + ":" + sku.getAttributeName2();
    }

    protected String toStringAddress(UserTradeInfo tradeInfo) {
        if (tradeInfo == null) {
            return null;
        }
        return MoreObjects.firstNonNull(tradeInfo.getProvince(), "") +
                MoreObjects.firstNonNull(tradeInfo.getCity(), "") +
                MoreObjects.firstNonNull(tradeInfo.getRegion(), "") +
                MoreObjects.firstNonNull(tradeInfo.getStreet(), "");
    }
	}

###SuibaoOrderExportFactory

	@Component @Primary @Slf4j
	public class SuibaoOrderExportFactory extends OrderExportFactory {

    @Override
    public Map<String, String> composeOrderExportFieldsMap(OrderExportDto orderExportDto) {
        if (! (orderExportDto instanceof SuibaoOrderExportDto)) {
            // log
            log.warn("suibao order export factory type not match, skip");
            return Collections.emptyMap();
        }
        SuibaoOrderExportDto suibaoOrderExportDto = (SuibaoOrderExportDto) orderExportDto;

        Map<String, String> fieldsMap = new HashMap<>();
        composeOrder(fieldsMap, suibaoOrderExportDto.getOrder());
        composeOrderItem(fieldsMap, suibaoOrderExportDto.getOrderItem());
        composeOrderExtra(fieldsMap, suibaoOrderExportDto.getOrderExtra());
        composeOrderTrack(fieldsMap, suibaoOrderExportDto.getOrderTrack());

        composeOrderDeliverType(fieldsMap, suibaoOrderExportDto.getOrder());
        composeOrderDispatchInfo(fieldsMap, suibaoOrderExportDto.getDispatchDto());
        composeOrderPickInfo(fieldsMap, suibaoOrderExportDto.getOrderPickInfo());
        composeDeliverCoupon(fieldsMap, suibaoOrderExportDto.getOrderDiscountDetail());
        return fieldsMap;
    }

    private void composeDeliverCoupon(Map<String, String> fieldMap, OrderDiscountDetail deliverCouponDetail) {
        if (null != deliverCouponDetail) {
            fieldMap.put(SuibaoExportConstants.DELIVER_COUPON, deliverCouponDetail.getPrice().toString());
        }
    }

    private void composeOrderDeliverType(Map<String, String> fieldMap, Order order) {
        if (null != order){
            if (null != order.getDeliverType())
                fieldMap.put(SuibaoExportConstants.ORDER_DELIVER_TYPE, toStringOfOrderDeliverType(order.getDeliverType()));
        }
    }

    private void composeOrderPickInfo(Map<String, String> fieldMap, OrderPickInfo orderPickInfo) {
        if(null != orderPickInfo){
            if (!Strings.isNullOrEmpty(orderPickInfo.getUserName())) fieldMap.put(SuibaoExportConstants.ORDER_PICKER_NAME, orderPickInfo.getUserName());
        }
    }

    private void composeOrderDispatchInfo(Map<String, String> fieldMap, DispatchDto dispatchDto) {
        if (null != dispatchDto){
            if (null != dispatchDto.getUser()) {
                if (!Strings.isNullOrEmpty(dispatchDto.getUser().getName()))
                    fieldMap.put(SuibaoExportConstants.DELIVER_NAME, dispatchDto.getUser().getName());
            }
            if (null != dispatchDto.getDispatch()) {
                if (null != dispatchDto.getDispatch().getId())
                    fieldMap.put(SuibaoExportConstants.DISPATCH_ID, dispatchDto.getDispatch().getId().toString());
            }
            if (null != dispatchDto.getDelivery()) {
                if (null != dispatchDto.getDelivery().getCreatedAt())
                    fieldMap.put(SuibaoExportConstants.DELIVERY_CREATED_AT, printDate(dispatchDto.getDelivery().getCreatedAt()));
                if (null != dispatchDto.getDelivery().getDeliveryAt())
                    fieldMap.put(SuibaoExportConstants.DELIVERY_DONE_AT, printDate(dispatchDto.getDelivery().getDeliveryAt()));
            }
        }
    }

    protected String toStringOfOrderDeliverType(Integer deliverType) {
        for (DeliverType type : DeliverType.values()) {
            if (Objects.equals(type.value(), deliverType)) {
                return type.toString();
            }
        }
        for (SuibaoDeliverType suibaoType : SuibaoDeliverType.values()) {
            if (Objects.equals(suibaoType.value(), deliverType)) {
                return suibaoType.toString();
            }
        }
        return null;
    }

    @Override
    protected String toStringOfOrderStatus(Integer status) {
        if(status == 301 || status == 302 ){
            SuibaoOrderStatus suibaoOrderStatus = SuibaoOrderStatus.from(status);
            return suibaoOrderStatus == null ? null : suibaoOrderStatus.description;
        }
        return super.toStringOfOrderStatus(status);
    }
	}

###结语
导出代码过多，此处记录只显示部分，import不列出。

[StrongL]:    http://stronglong.me  "StrongL"
