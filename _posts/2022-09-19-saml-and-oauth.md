---
layout: post
title: '了解一下SAML2.0'
date: 2022-09-19
author: Calvin Wang
cover: '/assets/img/202209/saml.drawio.png'
tags: aws sso saml oauth
---

> 文章分成三个部分：<br />
> 第一部分简单介绍在AWS上配置以Azure AD作为认证方的关键配置<br />
> 第二部分介绍一下SAML2.0认证流程<br />
> 第三部分来讲一下SAML2 和 OAuth2 协议的区别。（不会涉及OIDC协议）<br />
> <br />
> 文章均是个人理解，恳请扶正<br />


# 背景
## 为什么写这篇文章呢?
家里用docker安装了很多软件，每个软件都有自己独立的账密。用起来是真的麻烦。好在绝大部分软件都是支持LDAP认证的。那上个OpenLDAP家庭内部的SSO就“实现”了。同样的在企业内部也会有这样的问题，他们是怎么解决的呢， 就蛮好奇的查询了下。就以熟悉的AWS为例进行学习时了解到了SAML2 协议。但是几乎没有什么文章解释清楚了它和OAuth协议的异同。尝试写一篇文章来回答学习时的各个疑问。希望能温故知新，也为了年度目标再进一步。

# 以AWS作为认证提供方来配置AWS的SSO
我只会把一些关键节点的配置列出来。 详细的文档请参考：[AWS IAM Identity Center](https://aws.amazon.com/cn/iam/identity-center/)

## Azure 平台配置：
* 在Azure AD中创建“AWS Single-Account Access” Application
* 配置“AWS Single-Account Access” Application， 选择SAML 验证。
* 在Base SAML Configuration中配置Entity ID 和 Reply URL(回调URL)。如：https://signin.aws.amazon.com/saml#1
* AWS SAML 认证需要通用的属性信息，如用户姓名邮箱的 和 允许承担的Role相关的信息。 详细的解释请参考：[Configuring SAML assertions for the authentication response](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/id_roles_providers_create_saml_assertions.html#saml_role-attribute)

```html
  <! -- Role 相关的格式--> 
  <Attribute Name="https://aws.amazon.com/SAML/Attributes/Role">
    <AttributeValue>arn:aws:iam::account-number:role/role-name1,arn:aws:iam::account-number:saml-provider/provider-name</AttributeValue>
    <AttributeValue>arn:aws:iam::account-number:role/role-name2,arn:aws:iam::account-number:saml-provider/provider-name</AttributeValue>
    <AttributeValue>arn:aws:iam::account-number:role/role-name3,arn:aws:iam::account-number:saml-provider/provider-name</AttributeValue>
  </Attribute>
```
* `重要`：配置签名证书

## AWS 平台配置
* 在IAM中配置新的Identity Providers
  ![](https://learn.microsoft.com/zh-cn/azure/active-directory/saas-apps/media/amazon-web-service-tutorial/ic795034.png)
* 创建需要的IAM， 注意信赖刚刚创建的Identity Providers
  ![](https://learn.microsoft.com/zh-cn/azure/active-directory/saas-apps/media/amazon-web-service-tutorial/ic795025.png)
* 创建一个IAM User 用来List Role去允许Azure 进行绑定。
* 后续步骤略

## 使用方式：
* 用户用“浏览器”打开在上一步在Azure Applition 中提供的 Login URL
* 进行登陆(已经登陆过会跳过验证)， 部分情况下会存在2FA验证
* 登陆成功后会调用在上一步配置的Replay URL使用户成功登陆AWS。

# 介绍下SAML协议
在 SAML 协议中，涉及两个主体：
* Service Provider 服务提供方，简称 SP。就是上文中的AWS。
* Identity Provider 身份提供方，简称 IdP。就是上文中提到的Azure。
两个主体是通过用户的浏览器进行信息交换。

## SAML 流程
![saml](/assets/img/202209/saml.drawio.png)
在第一步的例子中: 
* 用户想使用AWS， 需要先打开在Azure AD中创建的Application生成的Login URL
* 用户在Azure完成验证之后， Azure会生成SAML Response，并让浏览器通过`POST`方法，将SAML Response 发送给创建Azure时填写的Reply URL。 也就是发送给了AWS。
* AWS 校验完SAML Response的`签名`后, 会让用过选择登陆Role（如果存在多个Role的话）。
* 确定Role之后，AWS 会生成一个临时Token供用户操作AWS。

到这里熟悉OAuth的话， 会感觉这两个协议特别的相像。 下面一节就来看看两个协议的差异。

# SAML2 和 OAuth2
这里仅仅讨论OAuth2协议， 不会涉及在其之上的OIDC协议。
## 目的: 认证和授权
* SAML2是一个“认证”协议，偏重于证明你是你。就像是家里的开门用的钥匙， 谁有这把钥匙，谁就是这个家的主人，就可以进来这个家。
* OAuth2是一个“授权”标准，偏重于说明你有权限。就像是和朋友合租了一个房子，我们都有大门的钥匙， 但是我们只对自己的卧室和公共空间有使用权。

## 适用场景：
* SAML2 适用于Web程序。就像上面的流程图中标红的步骤，IdP登陆通过之后会生成一个SAML Reponse， 但是因为SAML Reponse包含的信息面面俱到，会比较大。只能选择以POST的方式与SP进行交互(GET 方法有长度信息， 这个长度是多少没人知道...)。据我所知在手机App唤醒中，仅能获取GET 请求，对POST请求乏力。也就导致了SAML2 不适用与App场景。 
* OAuth2 适用于Web程序、App程序等等。OAuth2协议使用固定长度Token与服务提供商进行交互。 规避了SAML2的缺点。

## 对接/实现难度
* OAuth2是一个“授权标准”，可以参考[RFC 6479](https://www.rfc-editor.org/rfc/rfc6749)。它定义了一种资源提供方和客户端之间授权的四种方式。并没有详细定义每种对应方式的细节。也就导致各家在实现OAuth2协议时都有细微的不同。
* SAML2相比于OAuth2来说定义是面面俱到， 交互协议、交互信息格式内容都有详细定义。

基于以上内容，就导致SAML2对接起来会更容简单，但是实现一个SAML2 IdP就过于麻烦。

## 安全
* OAuth2 没有额外定义安全协议，纯靠HTTPS。
* SAML2 基于数字签名方案来增加安全性。


# 最后
准备写的时候发现已经没有Azure和AWS的使用环境， 写第一部分的时候就挺烦的。原本计划直接删掉的。 各位就凑合着看吧。<br />
AWS 蛮贴心的提供了[AssumeRoleWithSAML](https://docs.aws.amazon.com/zh_cn/STS/latest/APIReference/API_AssumeRoleWithSAML.html) API, 所以只要获取了SAML Response，就能以程序替代人来完成一些工作了。<br />
就这么多， 辛苦你看到最后！
