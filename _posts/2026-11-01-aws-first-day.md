---
layout: post
title: "AWS 新账号第一天：根账号保护、Organizations 与账单告警"
description: "第一天把根用户收进保险柜，用带 MFA 的管理员应付日常，建好组织，并用预算和 CloudWatch 账单告警盯住花费。"
date: 2026-11-01 09:00:00 +0800
categories: [安全, 账号与账单]
tags: [AWS, 安全, Organizations, 账单]
checked: 2026-11-03
---

新的 AWS 账号最容易在两件事上出事：每天用根用户点控制台，以及没有任何花费告警。第一台 EC2 可以等半天，这两件事不要等。下面按当天能做完的顺序写。控制台菜单位置会变，以链接的文档为准。

## 根用户只做恢复，不做日常操作

根用户拥有账号里不可被 IAM 策略缩小的权限。文档要求各类账号（独立账号、组织管理账号、成员账号）都为根用户配置多因素认证。若还没配置，从第一次尝试登录控制台算起，需要在 14 天内注册 MFA，否则会影响控制台访问。不要拿这 14 天当缓冲，注册当天就绑。

支持的类型是通行密钥和安全密钥、虚拟验证器应用，以及硬件 TOTP 令牌。绑定之前先确认根用户邮箱和电话是你自己能收到的：设备丢失后，恢复要靠它们。步骤见[为根用户启用 MFA](https://docs.aws.amazon.com/IAM/latest/UserGuide/enable-mfa-for-root.html)。

同时做到这三件：

- 不要为根用户创建访问密钥。控制台即使仍允许创建，也不要创建。
- 根用户密码放进有第二位可信管理员能够取用的密码管理器，不要放在聊天记录里。这是破玻璃用的，不是每天的登录方式。
- 根用户日常不登录。需要根用户才能做的少数操作（例如某些账号级设置）做完就退出。

更完整的清单见[根用户最佳实践](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html)。

## 给自己一条带 MFA 的日常入口

一个人、当天就要进控制台时，可以先建一个 IAM 管理员用户：只给这个人使用，挂 `AdministratorAccess`，强制 MFA，并设置好记的账号别名。之后从 `https://别名.signin.aws.amazon.com/console` 登录，而不是从根用户入口登录。用无痕窗口试一次，确认 MFA 生效，再把根用户密码收起来。

这只是过渡。一旦有第二个人要进 AWS，就改用 IAM Identity Center 管理员工身份，不要继续扩散长期 IAM 用户和长期访问密钥。Identity Center 可以和本账号的组织一起用。产品说明见[什么是 IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html)。

程序不要使用这个管理员的密钥。计算任务优先用实例角色或任务角色，拿临时凭证。

## 建组织，把业务和工作账号分开

AWS Organizations 本身不另外收费，费用来自成员账号里使用的资源。创建组织后，你当前这个账号成为管理账号，并负责支付组织内全部账号的用量。账单不会按你画的组织单元自动分栏，要看清成本，靠成员账号本身，或靠成本分配标签。见[Organizations 的费用](https://docs.aws.amazon.com/organizations/latest/userguide/pricing.html)。

服务控制策略 SCP 不作用于管理账号。所以「在管理账号上跑生产，再用 SCP 限制它」这条路不成立。官方的多账号实践是：管理账号只做账单和组织，工作负载放到成员账号。见[多账号环境的最佳实践](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices.html)。

第一天建议做到：

1. 根用户 MFA 和日常管理员已经可用。先保证自己不会被关在门外。
2. 用当前账号创建组织。它成为管理账号。
3. 再创建一个成员账号，实验和生产资源放进去。成员账号的根用户邮箱必须是你能收信的独立邮箱。
4. 先不要写拒绝类 SCP。默认的 `FullAWSAccess` 足够开始。等有第二名管理员、并且成员账号里的权限集已经验证过，再收紧。
5. 下一步再考虑集中管理成员账号的根访问：新成员账号可以不保留可登录的根用户凭证。这是安全加固，不是创建组织的前置条件。

只有一个账号、当天下午必须开一台试验机时，可以暂时开在当前账号。一旦这台机器要变成生产，后面把它迁出管理账号，比第一天就放进成员账号更麻烦。

## 先做预算，再做 CloudWatch 账单告警

免费套餐不是消费上限。两层告警都做，收件人用有人每天会看的邮箱，不要只发给没人打开的注册邮箱。

**AWS Budgets。** 打开账单与成本管理控制台里的 Budgets，建一个月度成本预算。通知阈值至少放 50%、80% 和 100%。预算看的是已发生费用，也可以另加预测通知。第一次创建预算时，如果还没开过 Cost Explorer，图表可能是空的，数据最长大约 24 小时后才出现，预算本身可以先保存。单账号、管理账号和成员账号都可以建预算。见[创建预算](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-create.html)。

**CloudWatch 账单告警。** 它只对「已经超过阈值的估算费用」报警，不做预测，所以代替不了预算，但阈值一过就会响。

1. 用有账单权限的身份打开 Billing and Cost Management，进入 Billing preferences。
2. 在告警首选项里打开 Receive CloudWatch Billing Alerts 并保存。文档写明这个数据收集之后不能关掉，告警本身可以删除。
3. 等待大约 15 分钟。
4. 把控制台区域切到美国东部（弗吉尼亚北部）`us-east-1`。账单指标只发布在这里，数值代表全球费用，不是只代表弗吉尼亚。
5. 创建告警：命名空间 Billing，指标 EstimatedCharges，统计值 Maximum，周期 6 小时，静态阈值，条件为大于你设定的美元金额。接一个你能收到的 SNS 邮件，并点开确认信。

如果在 `us-east-1` 仍看不到 Billing 指标，先确认第 2 步是在付款账号（组织的管理账号）上打开的。通过 Amazon Partner Network 结算的账号，文档写明不会把账单指标发到 CloudWatch。这种账号以 Budgets 和合作伙伴侧的额度通知为准，不要在错误的区域里反复刷新。步骤见[用 CloudWatch 监控估算费用](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/monitor_estimated_charges_with_cloudwatch.html)。

另外在账号页面加上备用账单联系人，让发票和付款失败邮件不只有根用户邮箱能收。付款方式本身在五家云之间差别很大，见[付款与账单对比]({% post_url 2026-10-30-five-clouds-billing %})。

## 当天验收

- 根用户已绑定 MFA，邮箱和电话可用，没有根用户访问密钥。
- 日常从管理员身份登录，并且这个身份有 MFA。
- 组织已创建。若已经有生产计划，成员账号已存在，管理账号里没有业务资源。
- 预算邮件发得出、收得到。CloudWatch 告警要么已在 `us-east-1` 建好，要么你已经确认该账号属于不会发布账单指标的合作伙伴结算。

根账号、组织单元和付款方式最好在开第一台 EC2 之前对齐。若需要协助开通 AWS 账号，开通后根用户和 MFA 仍应留在你指定的人手里，见[大牛云的开户说明](https://www.daniucloud.com/open)。
