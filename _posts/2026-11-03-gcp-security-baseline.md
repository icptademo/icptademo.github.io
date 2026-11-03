---
layout: post
title: "Google Cloud 新项目上线前的安全基线：组织、IAM、预算与审计日志"
description: "生产项目上线前核对四件事：项目挂在哪个组织、谁拥有 Owner、预算会不会只是发信，以及默认网络有没有把 22 端口对全世界打开。"
date: 2026-11-03 18:00:00 +0800
categories: [安全]
tags: [Google Cloud, IAM, 安全, 预算, 审计]
checked: 2026-11-03
---

Google Cloud 没有一把叫「根账号」的钥匙，但项目 Owner、账单账号管理员和组织管理员加在一起，权限并不比根账号小。新项目在创建虚拟机之前，把组织位置、身份、默认防火墙、预算和审计日志对一遍。按钮名称以控制台为准，文档链接在每节末尾。

## 先确认项目挂在哪

资源从组织、文件夹到项目，再到底层资源。一个资源只属于一个项目。项目 ID 创建后不能改，项目名称可以改，项目编号由系统分配。账单、API 和 IAM 都以项目为边界。层级说明见[资源层次结构](https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy)。

有 Cloud Identity 或 Google Workspace 域名时，项目应放在组织下面，必要时再用文件夹把生产和测试分开。没有组织时，项目会落在「无组织」。组织策略、集中看审计日志这些能力都用不了。个人 Gmail 建的项目可以做试验，不适合作为公司唯一的生产项目。

创建之后立刻看项目关联的 Cloud Billing 账号是不是你们要付钱的那一个。关联错了，预算和发票都会跑到别人的账单上。付款方式随国家和账单账号变化，结构上的差别见[五家云的付款与账单]({% post_url 2026-10-30-five-clouds-billing %})。

## 人进群组，程序不要下载密钥

打开 IAM 与管理 → IAM，看这个项目上的绑定。基础角色 Owner、Editor、Viewer 覆盖的权限很宽，不适合当日常授权。Owner 还能改 IAM 和账单关联。日常操作用预定义角色，授给 Google 群组，不要按人头一个个贴在项目上。

程序使用服务账号，并且尽量让虚拟机、Cloud Run 或 GKE 挂上这个服务账号，运行时拿临时凭证。能不创建 JSON 密钥就不创建。密钥文件一旦进了笔记本或代码仓库，就等于一份长期有效的密码。有组织时，可以用组织策略约束 `iam.disableServiceAccountKeyCreation` 禁止再新建密钥。IAM 的用法见[IAM 概述](https://cloud.google.com/iam/docs/overview)。

有组织时，在 Google 管理控制台给成员强制两步验证。没有组织时，至少给每一个拥有 Owner 或 Editor 的 Google 账号自己打开两步验证。

用命令把绑定打出来，比在控制台里一页页翻更不容易漏：

```bash
gcloud projects get-iam-policy PROJECT_ID
```

重点看 `roles/owner`、`roles/editor`，以及 `user:` 开头、却不属于公司域名的成员。

## 默认网络对全世界开了 22 和 3389

新建项目时，除非组织策略禁止，否则会自带一张名为 `default` 的 VPC，并预置几条入站防火墙规则。其中 `default-allow-ssh` 允许 `0.0.0.0/0` 访问 TCP 22，`default-allow-rdp` 允许同样的来源访问 TCP 3389。规则列表见[VPC 防火墙规则](https://cloud.google.com/firewall/docs/firewalls)。

上线前在 VPC 网络 → 防火墙里找到这两条。试验项目可以直接删除。生产项目改成只允许公司出口 IP，或改走 Identity-Aware Proxy，不要留着「方便先连上」。组织级可以设置约束 `compute.skipDefaultNetworkCreation`，让以后的新项目不再自动创建这张网络。已经存在的项目不会因此被改掉，还是要逐个处理。

```bash
gcloud compute firewall-rules list --project=PROJECT_ID
```

输出里只要还有 `default-allow-ssh` 或 `default-allow-rdp`，而且来源是 `0.0.0.0/0`，就还没做完。

## 预算默认只发邮件

预算建在账单账号上，也可以只圈定一个项目。路径是结算 → 预算和提醒 → 创建预算。范围选这个生产项目，金额填你们自己的月度上限。阈值至少放实际费用的 50%、90% 和 100%，有需要再加一条预测费用。邮件要进有人看的邮箱。只勾选账单管理员，而那个邮箱没人打开，等于没有告警。

官方写明：只告警的预算不会自动停用服务，也不会拦住继续扣费。部分账号另外提供支出上限类预算，打开之后可能影响项目计费。生产项目不要拿这个开关做试验。说明见[创建预算和预算提醒](https://cloud.google.com/billing/docs/how-to/budgets)。

## 确认管理活动日志在记

Cloud Audit Logs 里，管理活动日志和系统事件日志始终开启，不能关掉，写入 `_Required` 日志桶，保留 400 天。谁改了 IAM、谁创建了虚拟机，上线当天就应该能查到。

数据访问日志除 BigQuery 的部分服务外默认关闭。它记录读配置和读写用户数据，量可能很大，打开后会计入日志费用。上线清单里不要求第一天就全开。先确认管理活动日志有记录，再按需要打开 `ADMIN_READ` 或数据读写。概述见[Cloud Audit Logs](https://cloud.google.com/logging/docs/audit)。

在日志记录 → 日志浏览器中查询：

```text
logName:"clouactivity"
```

能看到刚才修改 IAM 或防火墙的操作，再进入下一步。查不到时，先确认右上角选中的是这个项目，而不是个人试验项目。

## 上线前勾选

- 项目在组织内，或你们明确接受它暂时没有组织。
- 关联的账单账号正确，预算邮件有人收。
- 项目上没有不认识的 Owner，日常权限在群组上。
- 没有为了图方便下载的服务账号 JSON 密钥。
- `default-allow-ssh` 和 `default-allow-rdp` 已删除或收紧来源。
- 日志浏览器里能看到一条管理活动记录。

人离开公司时，Owner 和账单管理员要按[团队交接清单]({% post_url 2026-11-03-account-handover-checklist %})收回。项目该挂在哪个账单账号上，应在创建生产项目之前定下来。需要按企业主体协助开通 Google Cloud 的，看[大牛云的开户说明](https://www.daniucloud.com/open)。
