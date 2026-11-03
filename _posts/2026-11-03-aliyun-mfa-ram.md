---
layout: post
title: "阿里云国际站注册后必做的安全设置：MFA 与 RAM 子账号"
description: "国际站主账号的权限不能被缩小。注册当天就绑定 MFA、停掉主账号 AccessKey，并用各自绑定 MFA 的 RAM 用户接管日常操作。"
date: 2026-11-03 09:00:00 +0800
categories: [安全]
tags: [阿里云国际, MFA, RAM, 安全]
checked: 2026-11-03
---

阿里云国际站的主账号（文档中的 Alibaba Cloud account）对名下全部资源拥有不可缩小的权限。注册完成后如果还用它每天登录、把密码发到群里，或者创建一把主账号 AccessKey 交给程序，任何一次泄露都等于整站失守。下面按注册当天能做完的顺序写。按钮名称以你账号中心里看到的为准。

## 先分清两种身份

主账号是资源所有者。资源访问管理 RAM 里的用户默认没有任何权限，授了什么才能做什么。RAM 用户本身不另外计费，资源费用记在主账号上。

人、管理员和程序不要共用同一个登录名。

<div class="table-wrap" tabindex="0">
<table>
  <caption>三种身份，对应三种凭证。不要混用。</caption>
  <thead>
    <tr><th>谁在用</th><th>用什么</th><th>不要做什么</th></tr>
  </thead>
  <tbody>
    <tr><td>几乎没有人</td><td>主账号，只用于恢复和个别必须由主账号完成的设置</td><td>日常登录，创建 AccessKey，共享密码</td></tr>
    <tr><td>负责开通资源的同事</td><td>单独的 RAM 用户，只开控制台登录</td><td>再给这个用户发一把长期 AccessKey</td></tr>
    <tr><td>跑在 ECS 或其他云产品上的程序</td><td>实例 RAM 角色，或单独的 RAM 用户</td><td>把密钥写进镜像、工单或聊天记录</td></tr>
  </tbody>
</table>
</div>

权限模型的完整说明见[使用 RAM 管理用户权限](https://www.alibabacloud.com/help/en/ram/use-cases/use-ram-to-manage-user-permissions-and-resources)。

## 主账号只保留恢复能力

用主账号登录账号中心，先确认登录邮箱能收信、绑定的手机能收验证码。MFA 设备丢失时，恢复依赖这些联系方式。邮箱如果还在注册服务商手里，先改到自己控制的邮箱，再做后面的步骤。

然后在安全设置里给主账号绑定多因素认证。文档里的入口有两种写法，对应不同时期的控制台：一种是账号保护（Account Protection），一种是虚拟 MFA 旁边的绑定。看到哪个做哪个，最终状态应是已绑定。虚拟 MFA 使用基于时间的一次性口令，文档示例的验证器是 Google Authenticator 或阿里云 App。扫码之后，把密钥同时存进密码管理器，不要只存在一台手机里。

步骤以[为主账号绑定 MFA](https://www.alibabacloud.com/help/en/account/configure-account-mfa)为准。

主账号不要创建 AccessKey。这把密钥拥有全部权限，也不能附加来源 IP 或时间段。官方建议是不要创建、不要使用。已经存在且确认没有程序在调用的，先禁用，观察一段时间再删除。仍有程序在用的，先建一个只授了必要权限的 RAM 用户密钥，在预发环境替换并验证，再改生产，最后禁用主账号密钥。说明见[不要使用主账号 AccessKey](https://www.alibabacloud.com/help/en/ram/avoid-using-the-accesskey-pair-of-an-alibaba-cloud-account)。

## 建一个管理员，然后把主账号收起来

仍用主账号打开 RAM 控制台，创建一个用户，访问方式只选控制台登录。授权系统策略 `AdministratorAccess`。这个管理员权限仍然很大，但可以按人冻结，审计里也能看出是谁。

在 RAM 的设置里把默认域名改成你们能认出来的别名，然后复制控制台给出的 RAM 用户登录地址。国际站不要自己去拼中国站的 `signin.aliyun.com`。登录名一般是「用户名@默认域名」。

用浏览器的无痕窗口登录这个 RAM 用户，确认能打开控制台。确认之后，主账号密码只留在密码管理器里。创建步骤见[创建 RAM 用户作为账号管理员](https://www.alibabacloud.com/help/en/ram/create-admin-user)。

## 让每个 RAM 用户自己绑 MFA

顺序不要反。主账号或 RAM 管理员要先配置全局 MFA 策略，规定允许哪些设备、登录时是否强制。这条策略不会替用户把设备绑上。每个 RAM 用户在自己登录时绑定。

可选方式包括通行密钥、虚拟 MFA 和安全邮箱。虚拟 MFA 不依赖某一国的短信。通行密钥验证的是设备本身。安全邮箱适合做备用，不适合当唯一因素。官方建议至少绑两种，例如通行密钥加安全邮箱，避免手机丢失后整组人进不去。

不要把 MFA 二维码发到群里「方便值班」。那仍然是共享身份。每位需要登录的同事使用自己的 RAM 用户。绑定步骤见[为 RAM 用户绑定 MFA](https://www.alibabacloud.com/help/en/ram/bind-an-mfa-device-to-a-ram-user-1)。

## 业务权限按组发出去

管理员建好之后，其他人不要再挂 `AdministratorAccess`。

1. 按职能建组，例如网络、数据库、只读账单，把策略授给组，而不是授给每一个人。
2. 一个人一个 RAM 用户。离职或换岗时从组里移除，并禁用登录。
3. 系统策略可以起步，例如某个产品的只读策略。能写清动作和资源时，再改成自定义策略。
4. 需要调用 API 的程序单独建用户，只创建 AccessKey，不要同时开控制台密码。Secret 只在创建时显示一次。更稳妥的做法是不用长期密钥，让 ECS 使用 RAM 角色，程序在实例里拿临时凭证。这个区分写在[创建 RAM 用户](https://www.alibabacloud.com/help/en/ram/user-guide/create-a-ram-user)。

国际站和中国站是两套账号，资源、账单和 RAM 都不互通。在中国站控制台里找不到国际站的用户，是正常现象。

## 确认操作审计能查到人

权限做完如果没有日志，出了事只能猜。操作审计 ActionTrail 记录控制台、OpenAPI 和工具发起的操作。管理事件默认在线保留 90 天，不用额外开通就能查登录和改资源，例如谁停了一台 ECS。数据事件（如读取某个 OSS 对象）默认不采集。

需要保存超过 90 天，或要做检索告警时，再创建跟踪，把事件投递到日志服务或 OSS。投递之后按目标产品计费。先在控制台用「最近的登录」查一条 `ConsoleSignin`，确认事件里能看到 RAM 用户名和来源 IP，而不是只看到开通成功。产品说明见[什么是操作审计](https://www.alibabacloud.com/help/en/actiontrail/product-overview/what-is-actiontrail)。

## 当天验收

- 主账号已绑定 MFA，邮箱和手机你自己能收到验证码。
- 没有仍在启用的主账号 AccessKey。
- 管理员 RAM 用户能从控制台给出的地址登录，并且已经绑定 MFA。
- 同事手里没有主账号密码，值班也不共享二维码。
- ActionTrail 里能查到刚才的登录事件。

做完这些再去买 ECS 或充值。账单通知该发给谁，可以对照[五大国际云的付款和账单方式]({% post_url 2026-10-30-five-clouds-billing %})。

账号若由服务商协助开通，交接时请核对主账号邮箱、MFA、管理员 RAM 和账单联系人，不要只收一个登录密码。需要按企业主体办理国际站开户的，看[大牛云的开户说明](https://www.daniucloud.com/open)。
