---
title: 金融核心系统设计原则
date: 2017-08-27 10:13:20
categories: Design
tags: [financial system]
comments: false
description: "准确，安全，合规，可追溯，一致，补偿，幂等，识别，内外控"
---
### 1 正确
正确时金融系统设计的第一原则，不能保证正确其他原则都无从谈起。

### 2 安全
对金融数据做到防篡改，防解析，防获取。默认不信任任何外来请求，数据...没有用户会愿意使用一个不安全的金融系统。

### 3 合规
在国内的大环境下，对于有一定规模或有一定追求的金融公司，监管是不得不面对的问题。尤其是对互联网金融企业，针对一些国家，地方的互联网监管规范要求，如《互联网金融统计制度》和其他一些报送要求。

### 4 可追溯
对所有的登陆，退出，请求，响应，业务处理，状态变更都要记录历史信息。方便风控，审计，追溯，定位。

### 4 一致
领域（服务）内强一致性与领域（服务）间最终一致性相结合使用。系统间完善对账机制与业务监控。

### 6 补偿
无论服务内还是服务间，带超时的同步接口，查询/异步通知接口，撤销接口，配合定时补偿机制，结合使用。

### 7 幂等
保证服务调用的幂等性，业务处理的幂等性，不能保证幂等的地方需要请求去重，反馈去重，业务去重，对每一次变化，才能保证结果正确。

### 5 识别
对每一次引起变化的事件，都要可识别，request要有RequestId，responseId，业务要有businessId...对有状态的业务要识别尽可能多的状态（有人提到MECE分析法，深以为然），并记录详细的状态迁移信息。作到一切事件，状态皆识别。

### 9 内外控
风控针对所有用户。不仅包括所有客户，也包括所有内部用户。



欢迎在GitHub上关注FinTx开源金融系统：[FinTx](https://github.com/fintx)
