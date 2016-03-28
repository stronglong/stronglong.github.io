---
layout: post
title: EXCEL导出优化
description: 分离导出代码，组装成工具
category: blog
---

## 导出

###1、controller

####导出以提现为例，给出主要代码

	@RequestMapping(value="/stark-user-balance",method=RequestMethod.GET)
    @ResponseBody
    public void starkUserBalanceExportXls(
          @RequestParam(value = "id",required = false) Long id,
          @RequestParam(value = "userId",required = false) Long userId,
          @RequestParam(value = "status",required = false) Integer status,
          @RequestParam(value = "startAt",required = false) String startAt,
          @RequestParam(value = "endAt",required = false) String endAt,
          HttpServletResponse response){
     Response<List<UserBalanceDetailDto>> resp = starkReportExporter.loadByAllParams(id,userId,status,startAt,endAt);
        if (!resp.isSuccess()){
             throw new JsonResponseException(500,resp.getError());
        }
        try{
	        //封装导出内容为map类型的list
            List<Map<String,String>> mapList =startReportExportFactory.composeStarkReportExportFieldsMaps(resp.getResult());
            List<ExportFields.ExportField> fields = getStarkUserFieldFromYaml(); //取得excel导出字段
           // 调用导出方法
            new ExportModel().export(response,STARK_USER_BALANCE_DETAIL_NAME,
                    fields,mapList,resp.getResult().size(),messageSources);
        } catch (Exception e){
            log.error("export stark user balance detail failed,error:{}", Throwables.getStackTraceAsString(e));
            throw new JsonResponseException(500,"stark.user.balance.export.failed");
        }


####导出Model代码ExportModel

	@Slf4j
	public class ExportModel {

    public void export(HttpServletResponse response, String excelTitle,
                       List<ExportFields.ExportField> fields, List<Map<String, String>> mapList,
                       Integer count,MessageSources messageSources) {
        try {
            Map<Integer, Map<String, String>> groupMap = groupMapById(mapList);
            String xlsFileName = URLEncoder.encode(excelTitle, "UTF-8") + ".xls";
            response.setContentType("application/x-download");
            String headerKey = "Content-Disposition";
            String headerValue = String.format("attachment;filename=\"%s\"", xlsFileName);
            response.setHeader(headerKey, headerValue);
            Workbook wb = new SXSSFWorkbook();
            Sheet s = wb.createSheet();
            Row titleRow = s.createRow(0);
            for (int i = 0; i < fields.size(); ++i) {
                Cell cell = titleRow.createCell(i);
                ExportFields.ExportField field = fields.get(i);
                if (field.needTrans()) {
                    cell.setCellValue(messageSources.get(field.getFieldName()));
                } else {
                    cell.setCellValue(field.getFieldName());
                }
            }
            int rowNum = 1;
            for (int x = 0; x < count; ++x) {
                Row contentRow = s.createRow(rowNum);
                Map<String, String> fieldsMap = getMap(rowNum, groupMap);
                for (int i = 0; i < fields.size(); ++i) {
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
        } catch (Exception e){
            log.warn("export failed,cause:{}", Throwables.getStackTraceAsString(e));
        }
    }

    private  Map<String,String> getMap(Integer num,Map<Integer, Map<String,String>> mapMap){
        return mapMap.get(num);
    }

    private  Map<Integer,Map<String,String>> groupMapById(List<Map<String,String>> mapList){
        Map<Integer,Map<String,String>> result = new HashMap<>();
        int count = 1;
        for(Map<String,String> map : mapList){
            result.put(count,map);
            ++ count;
        }
        return result;
	    }
	}



差点忘了，factory得多加一个method，取得所有的数据

	 public List<Map<String,String>> composeStarkReportExportFieldsMaps(List<UserBalanceDetailDto> dtos){
        List<Map<String,String>> result = Lists.newArrayList();
        Map<String,String> fieldsMap = null ;
        UserBalanceDetail detail = null;
        User user = null;
        for (UserBalanceDetailDto dto : dtos){
            fieldsMap = new HashMap<>();
            if (dto.getDetail() != null){
                detail = dto.getDetail();
                if(detail.getId() != null) fieldsMap.put(StarkExportConstants.STARK_ID,detail.getId().toString());
                if(detail.getCreatedAt() != null) fieldsMap.put(StarkExportConstants.STARK_DATE_AT,printDate(detail.getCreatedAt()));
                if(detail.getUpdatedAt() != null) fieldsMap.put(StarkExportConstants.STARK_USER_ID,detail.getUserId().toString());
                if(detail.getPoundage() != null) fieldsMap.put(StarkExportConstants.STARK_POUNDAGE,toYuan(detail.getPoundage()).toString());
                if(detail.getBalance() != null) fieldsMap.put(StarkExportConstants.STARK_BANLANCE,toYuan(detail.getBalance()).toString());
                if(detail.getBalance() != null) fieldsMap.put(StarkExportConstants.STARK_REAL_PAY,toYuan(detail.getBalance()).toString());
                if(detail.getProvince() != null) fieldsMap.put(StarkExportConstants.STARK_PROVINCE,detail.getProvince());
                if(detail.getCity() != null) fieldsMap.put(StarkExportConstants.STARK_CITY,detail.getCity());
                if(detail.getBankName() != null) fieldsMap.put(StarkExportConstants.STARK_BANK_NAME,detail.getBankName());
                if(detail.getBankOpenName() != null) fieldsMap.put(StarkExportConstants.STARK_BANK_OPEN_NAME,detail.getBankOpenName());
                if(detail.getAccountNo() != null) fieldsMap.put(StarkExportConstants.STARK_ACCOUNT_NO,detail.getAccountNo());
                if(detail.getStatus() != null) fieldsMap.put(StarkExportConstants.STARK_STATUS,toStringOfStarkStatus(detail.getStatus()));
            }
            if (dto.getUser() != null){
                user = dto.getUser();
                if(user.getRealName() != null) fieldsMap.put(StarkExportConstants.STARK_USER_NAME,user.getRealName());
            }
            result.add(fieldsMap);
        }
        return result;
    }

虽然导出结果和原来一样，但还是不能随便放进项目里，所在在此记录一下，以便后续使用。



[StrongL]:    http://stronglong.com  "StrongL"
