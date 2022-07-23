---
title: 2021搬家记
abbrlink: 44fb5d01
date: 2021-09-11 03:02:08
tags:
    - 收纳
    - 搬家
    - 条形码
categories: 生活
---
{% cq %}
生活，就是理解。生活，就是面对现实微笑，就是越过障碍注视将来。生活，就是自己身上有一架天平，在那上面衡量善与恶。生活，就是有正义感、有真理、有理智，就是始终不渝、诚实不欺、表里如一、心智纯正，并且对权利与义务同等重视。生活，就是知道自己的价值，自己所能做到的与自己所应该做到的。生活，就是理智。——雨果
{% endcq %}

## 背景
没啥好说的，就是，我又双叒叕要搬家了。**起因就是因为，最近要涨房租**
与其不如涨价，反正都是涨，我寻思干脆住远一点，直接自己住得了。
以前搬家，稍微不那么简单一点，自己会花好长时间去收拾一下。收拾呢，其实也就是找一堆纸箱子，完了去打包收纳而已。
在过往的收纳经验来看，最后的结论就是：**大件的我能知道，放到具体哪个里面了。但是，小件，emm，死活最后就铁定不知道在什么地方了。**最真实的可能就是，在日后的生活过程中，扒拉扒拉找些东西的时候，哦？原来我还有这个东西......
<!-- more -->
尤其像我，电子产品众多，小东西也是众多，搬过几次家之后，我都不知道我到底还有哪些东西被遗忘进角落了。

综上，这次就来一次大收纳吧。目的就一个：**搬家前后，我知道我有哪些东西，并且快速准确的知道在哪个箱子里面**

## 准备过程
### 思考
想法其实很简单，就是日常中，拿超市为例子：超市上的每个上架货品（除了生鲜散称之外）基本都是有一个唯一的条形码，其次货架也有对应的货架编号。今天上了什么货物，要上到哪个架子上。这个是在后台一清二楚的。
那么换到我这次搬家场景，我要怎么做呢？
### 类比
搬家虽然不是上架售卖物品，但是在这个收纳的过程中，不也就是类似么？收纳的每个纸箱子，就是一个小“货架”，每个待收纳到纸箱子里面的物品，就是对应的一个个“商品”
### 思考
在需求设计之前，我需要对这个分类收纳过程中，涉及到的两个关键点分别阐述一下：
- 分类：
    + 涉及到分类，**第一个就是如何快速识别并记录下物品信息，供后续使用？** 我不可能自己一个个去录入，那是一个非常庞大冗杂的事情。光打标签分类，就是一个非常繁杂的事情。
    + 既然要识别：就会遇到两个场景，一种是确实有现成二维码（如物品外壳上的），还有一大类是零碎的物品怎么办？
    + 有了识别信息之后，如何快速分类呢？
- 收纳：
    + 有了分类信息之后了，怎么快速收纳？并且我还要快速准确的知道在哪个箱子里面？

其实想的有点繁琐了，其实正常收纳流程是：
- 收纳前：先分类 -> 收集物品信息 -> 分类打码 -> 录系统 -> 收纳
- 收纳中：关联收纳物与收纳箱之间的关系
- 收纳后：就是搬家后拿东西了，**反向检索**物品在哪个箱子里。

### 流程初探
针对上面的思考的若干个点，来一个个解决：
- 快速识别：
    + 有条形码的：我选择使用 **条码枪** 来快速扫描，来获取条形码信息。你可能会有疑问，为什么要用条码枪？其实用手机，有很多app借助手机摄像头就能扫描，识别率都还不错。那为什么还要用呢？原因就是：**我要做一个整体的收纳系统！**光靠app，但是数据之间怎么实时传递，那还是有点烦人。最后决定，就是一次性投入一点，买个设备吧，日后还能经常用。
    + 没有条形码零碎的：先进行初步的归纳整理，先在后台自行录入物品集合信息，再进行统一的分类并打码（做条形码），这样就能够供快速识别所需。
- 分类：结合上面快速识别之后，会产生下面几个问题：
    + 通过条码枪扫描的物品，你拿到的就是一串数字而已，你怎么知道对应的是什么东西？知道了什么东西之后，你又要怎么去分类？
        * 扫码之前，我是知道是什么东西的，此时我就已经可以分类好了。
        * 光有条码，扫完之后，我要怎么知道是什么东西？这个找到一个现成的免费的，通过条形码去检索物品信息的API接口
    + 没有条码的，这个可以在录入物品集合信息期间，自行分类好，是厨房用品，是小家电，还是衣物、鞋帽什么的。
    + 做收纳箱的分配条码
- 扫码打包
    + 遵从扫描单个或多个物品之后，最后扫描分配好的收纳箱子的二维码/条形码，进行封箱登记
- 后台信息处理
    + 条码分配：
        * 收纳箱：前缀统一为BOX，后续分配自增序号，总长度待研究设备之后再定义修改
        * 自定义物品：前缀统一为ITEM，后续分配自增序号
    + 封箱登记流程：
        * 在扫描常规正常条形码、自定义物品信息后，~~扫描结果传送至服务器~~ 扫码枪，与pc通信是借助于一个usb收发器来完成的，传输到PC端是向PC输入窗口来实现的。
        * 当扫描到统一前缀BOX的条码时，后台进行封箱登记操作：将前面的所有扫描的物品进行打包，登记当前物品所在的箱子是在当前条码箱子内
    + 统一信息处理后台：这个后台，集中处理所有物品信息，条码信息，封箱信息等后台管理平台。
## 需求设计
家庭收纳信息处理后台(TakeInEverything, TIE)
### 技术栈选型
一个后台，多种前端。
后台核心相关主要：
- 数据库：MySQL 5.7
- SpringBoot
- Shrio（统一认证，权限管理等）
- Dubbo（接入，供后续可能的服务订阅使用）
- Zookeeper

web前端：
- Dubbo
- Springboot


## 详细设计
### 一期
先把核心功能做出来
#### 后台表
- 分类表(tie_classification)
    + id
    + 一级大类(top_sub)
    + 二级小类(second_sub)
    + 添加时间(create_time)
    + 修改时间(update_time)
    + 添加人(add_user)
    + 修改人(update_user)
    + 备注(remark)
- 物品明细表(tie_item_detail)
    + id
    + 物品条码(item_barcode)
    + 物品名称(item_name)
    + 物品品牌(item_brand)
    + 物品供应商(item_supplier)
    + 物品类别(item_class)
    + 添加时间(create_time)
    + 修改时间(update_time)
    + 添加人(add_user)
    + 修改人(update_user)
    + 备注(remark)
- 收纳明细表(tie_box_detail)
    + id
    + 收纳箱条码(box_barcode)
    + 收纳箱名称(box_name)
    + 收纳箱描述(box_description)
    + 收纳箱分类(box_class)
    + 添加时间(create_time)
    + 修改时间(update_time)
    + 添加人(add_user)
    + 修改人(update_user)
    + 备注(remark)
- 物品收纳表(tie_item_boxing)
    + id
    + 物品条码(item_barcode)
    + 收纳箱条码(box_barcode)
    + 收纳时间(take_in_time)
    + 创建时间(create_time)
- 收纳流水表(tie_takein_log)
    + id
    + 条码(barcode)
    + 创建时间(create_time)
- 用户信息表(tie_user_info)(TODO)
    + id
    + 用户名(username)
    + 用户密码(password)
    + 

#### 收纳流程
1. 分配
1. 采集流程
2. 


## 落地

## 未来
在思考过程中，也有提到，现在采取扫描枪的方式录入信息，那么怎么快速读取信息？比如，从收纳场景来说，我怎么快速知道，当前收纳箱里面有哪些东西？
当前最方便的就是，通过手机来读取，那么读取的可能性就很多了。
通过微信小程序？通过独立App？等等

这个就是一个大的联动相关了，这个就需要系统的再想想。

## 关于 