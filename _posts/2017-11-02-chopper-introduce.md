---
layout: post
title: 小型插件类模块Chopper
---

## 小型插件类模块Chopper

## 模块作用

##１.用户填写项校验记录 储存
用户在支付时展示给用户并且需要填写的项目，当用户提交支付时，我们会检查风控＋路由综合下来的填写项用户是否都填写了

## 2.支付宝,微信支付记录 储存

用户每次用支付宝或微信支付完，需要将相关数据储存下来，当用户下次进入收银台时，会根据这些数据去决定是否给用户展示支付宝

## 主要修改

完成risk模块逻辑的拆分（废除原有无线风控系统以及相关定时任务）
建立新的数据库，并使用QMHA

## 修改效果

## 去除冗余订单信息的存储(orderDetail字段及相关表)

##缩减risk库,逻辑相关的表迁入chopper新库 (约100G):
qmp_risk_white_list_did  8.7G
qmp_risk_white_list_userId  3.3G
qmp_risk_order_detail_XXXX月表3张  约130G
其余约５G左右

## 完成逻辑整合，去除风控降级等相关逻辑，并废除老的dubbo接口


## 暴露dubbo接口

## PayRiskFacade接口提供服务

`postProcessRiskItems,持久化风控填写项`

`queryRiskItemsByCondition,查询风控填写项`

##PluginAssistFacade接口提供服务

`isUserInPluginWhitePaper`,查询用户是否在白名单中

`isUserBlockPlugin`,(待确定,可以干掉)

## 暴露https接口

`/pluginAssist/persistWhitePaperRecord,持久化支付宝白名单记录`

`/pluginAssist/deleteWhitePaperRecord,删除3个月前支付宝白名单记录`



## 使用全新数据源QMHA


半同步复制

3M慢慢迁移



## 接入交易监控系统API



	<dubbo:consumer filter="-qaccesslogconsumer" retries="0">

        <dubbo:parameter key="currentSystemName" value="nami" />

        <dubbo:parameter key="qloglevel" value="4"/>

        <dubbo:parameter key="server" value="netty"/>

        <!--因为考虑client不会同时升级，兼容netty3，不然会出异常-->

        <dubbo:parameter key="client" value="netty"/>

    </dubbo:consumer>



    <dubbo:provider>

        <dubbo:parameter key="currentSystemName" value="nami" />

        <dubbo:parameter key="qloglevel" value="10" /><!-- qloglevel < 5 无日志输出 -->

        <dubbo:parameter key="threads" value="600"/>

        <dubbo:parameter key="provRCFetchCls" value="com.qunar.pay.trade.commons.dubbo.QReponseErrorCodeFetcher"/>

    </dubbo:provider>

对于被调用的系统,可以通过currentSystemName对调用系统名进行感知

￼￼



即监控节点下的dubbo特定节点,会安装provider的系统名进行归类,方便按系统进行监控



    public final class Metrics {



    	public static void countAllTime(String groupName, String name)



    	public static void countAllTime(String groupName, String name, int delta)



    	public static void countByTime(String groupName, String name, long delta)



    	public static void status(String groupName, String name, long val)



    	public static void status(String groupName, String name, IStatusFetcher fetcher)



    	public static void avgTime(String groupName, String name, long timeCost)



    	public static TimeContext startAvgTime(String groupName, String name)

    }

