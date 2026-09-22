# 没有 TRX，x402 如何在 TRON 上完成首次 Permit2 支付？

本文面向关注 x402 的开发者，介绍 TRON 上的 TRC-20 授权资源赞助，以及它在 Facilitator 中的实现位置。阅读后，你可以理解这项能力的工作方式，并找到测试和自托管开发的入口。

先明确前提：**付款账户已经激活，代币和支付路径受支持，服务端与 Facilitator 已启用赞助，且赞助方有足够资源与额度。** 在这些条件下，付款钱包可以无需预先持有 TRX，完成首次 Permit2 授权并继续支付。账户激活不包含在本扩展内。

本文是能力介绍与架构导览，不覆盖生产部署和生产安全评估。相关边界见扩展规范的 [Security considerations](https://github.com/BofAI/x402/blob/v1.2.0/specs/extensions/trc20_approval_resource_sponsoring.md#security-considerations)。

## 1. Agent 为什么会卡在首次支付？

一个研究 Agent 找到了一份报价为 0.05 USDT 的数据。钱包里有足够的 USDT，价格也在预算内，但支付仍可能停在授权环节：账户还没有为 Permit2 建立代币授权，可用的能量和带宽又不足以执行这笔交易。

USDT 是支付资产；能量（Energy）和带宽（Bandwidth）是执行链上操作所需的资源。钱包有钱支付服务，不等于已经具备首次授权所需的网络资源。

通常，用户需要先准备 TRX 或补充资源。授权资源赞助将这段准备工作交给 Facilitator 协调，付款钱包仍然负责签名。

## 2. Permit2 首次支付为什么需要 approve？

Permit2 是这条支付路径使用的授权合约。付款钱包先通过 TRC-20 `approve` 允许指定的 Permit2 合约使用代币，再通过单独的付款凭证约束本次支付。

这几个概念承担不同职责：

| 概念 | 在流程中的含义 |
| --- | --- |
| TRC-20 授权 `approve` | 一笔链上交易，为指定的 Permit2 合约设置代币使用额度 |
| 授权额度 `allowance` | 代币合约记录的可用授权额度，用来判断是否还需要 approve |
| 付款凭证 | 钱包对本次支付条件的签名，例如金额、收款方与有效期 |
| 结算 | 根据有效的付款凭证执行链上付款或通道存款 |

**approve 成功只代表授权生效，不代表本次付款已经完成。** 后续调用只要 allowance 仍然足够，就可以跳过 approve。

## 3. 授权资源赞助如何工作？

`trc20ApprovalResourceSponsoring` 是接入现有 x402 流程的扩展。服务端在支付要求中声明赞助能力；客户端需要 approve 时，签署授权交易并随付款凭证一同提交，暂不自行广播。

Facilitator 验证交易与付款凭证后，协调 **Resource Owner**（资源账户）临时向付款账户委托所需的能量，必要时补充带宽。资源可用后，Facilitator 广播钱包签好的原始 approve，确认 allowance 生效，再继续结算。

![首次 Permit2 支付对比：自行准备授权资源，与由 Facilitator 协调资源赞助](trc20-resource-approve-assets/01-before-after.png)

*图 1：资源准备方式发生变化；代币授权与付款凭证仍由付款钱包签署。*

Resource Owner 通过 Stake 2.0 质押获得可委托的资源。委托的是资源使用额度，质押本金仍属于资源账户。它与提供数据或模型的业务服务端（Resource Server）是不同角色。

授权生效后，Facilitator 尽快发起资源撤回，并在后台跟踪恢复；结算不必等待撤回确认。网络资源仍然有成本，具体赞助条件由服务运营方决定。

## 4. 五步支付流程与 0.05 USDT 案例

以 `exact` 固定金额付款为例，一次首次支付可分为获取报价、钱包签名、校验并准备资源、授权后结算、返回结果五步。

![首次支付五步流程：获取报价、钱包签名、校验并提供资源、授权后结算、返回服务结果](trc20-resource-approve-assets/02-payment-flow.png)

*图 2：客户端沿用 x402 请求流程，不需要另行调用资源准备接口。*

假设已激活的钱包有 10 USDT、0 TRX，Permit2 allowance 为零。Agent 接受 0.05 USDT 的报价，服务按已声明的条件赞助授权资源；本例假设没有其他代币费用，也没有并发交易。一次成功调用的状态变化如下：

| 检查项 | 支付前 | 授权与结算成功后 |
| --- | --- | --- |
| 付款账户 USDT | 10 USDT | 9.95 USDT |
| 付款账户 TRX | 0 TRX | 0 TRX，本流程不依赖付款人燃烧 TRX |
| Permit2 allowance | 0，需要 approve | 已建立；后续可按剩余额度判断是否需要再次授权 |
| 付款凭证 | 本次调用尚未签署 | 已签署，并用于本次 0.05 USDT 结算 |
| 数据结果 | 尚未取得 | 已返回，Agent 继续任务 |

*这是用于解释流程的状态示例，不是实测记录，也不是服务费用承诺。*

## 5. 支持范围与使用条件

当前扩展规范覆盖以下 TRON Permit2 路径：

| 支付方案 | 资源赞助作用的位置 |
| --- | --- |
| `exact` | 固定金额付款需要 approve 时 |
| `upto` | 授权支付上限、后续按实际金额结算的路径需要 approve 时 |
| `batch-settlement` | Permit2 通道存款或补款需要 approve 时；后续 Voucher、Claim 等操作不重复使用本扩展 |

付款账户需为已激活、使用默认 owner 权限的单签普通账户。服务端声明、客户端签名能力、Facilitator 配置与资产支持必须匹配。这条路径与 `exact_gasfree` 的 GasFree 账户和中继路径不同。

SDK v1.2.0 签署的是向指定 canonical Permit2 授予 `MaxUint256` 额度的 approve；每笔付款或存款另有签名约束。默认 `zero-first` 策略适用于 allowance 为零的情况；非零但不足的额度需要按代币策略另行处理。具体条件以[扩展规范](https://github.com/BofAI/x402/blob/v1.2.0/specs/extensions/trc20_approval_resource_sponsoring.md)为准。

## 6. Facilitator 架构与精简注册示例

Facilitator 中的协议实现验证授权交易与付款凭证；**runtime** 编排资源赞助；**Resource Owner** 提供资源并签署委托、撤回操作。runtime 完成授权准备后，相应支付方案继续执行结算。

![Facilitator 内的 runtime 架构：verify、sponsor、reconcile，与 Resource Owner 和后台任务的关系](trc20-resource-approve-assets/04-runtime-architecture.png)

*图 3：SDK 提供流程与接口，部署方接入资源账户和运行所需的组件。*

| runtime 入口 | 职责 |
| --- | --- |
| `verify()` | 只读检查账户、allowance、资源需求与赞助条件，不委托资源、不广播交易 |
| `sponsor()` | 预留并委托资源，广播原始 approve，确认授权生效，记录并发起回收 |
| `reconcile()` | 由后台任务调用，处理结果未知的交易以及未完成的资源撤回与恢复 |

下面是**装配骨架**，用于说明注册关系，不是可直接运行的部署脚本。`network`、签名器、协调器、资产地址和权限配置需由应用提供：

```typescript
import { x402Facilitator } from "@bankofai/x402-core/facilitator";
import { createTrc20ApprovalResourceSponsoringExtension } from "@bankofai/x402-extensions";
import { createTrc20ResourceSponsoringRuntime } from "@bankofai/x402-tron";
import { ExactTronScheme } from "@bankofai/x402-tron/exact/facilitator";

const runtime = await createTrc20ResourceSponsoringRuntime({
  network, resourceOwnerSigner, coordinator,
  allowedAssets: [usdtAddress], permissionId,
});
const facilitator = new x402Facilitator()
  .register(network, new ExactTronScheme(settlementSigner))
  .registerExtension(
    createTrc20ApprovalResourceSponsoringExtension(runtime),
  );
// 另由后台任务持续调用 runtime.reconcile()。
```

需要进一步定制时，可从 `policy`、`coordinator`、`chain` 和 `resourceOwnerSigner` 的接口入手。持久化、恢复、远程签名与 HSM 接入的要求见 [SDK Facilitator 接入文档](https://github.com/BofAI/x402/blob/v1.2.0/typescript/packages/extensions/src/trc20-approval-resource-sponsoring/README.md#facilitator)与[模块接口](https://github.com/BofAI/x402/blob/v1.2.0/typescript/packages/mechanisms/tron/src/resource-sponsoring/types.ts)。多资源池调度与第三方能量供应商接入需要额外实现，不属于现成的 SDK 部署能力。

## 7. Nile 体验与自托管入口

**先确认服务是否实际启用扩展。** 截至 2026 年 9 月 22 日，[Official Facilitator 文档](https://docs.bankofai.io/zh-Hans/x402/core-concepts/OfficialFacilitator/)仍注明：BANK OF AI 运营的 Official Facilitator 尚未启用 `trc20ApprovalResourceSponsoring`。同日查询官方 [`/supported`](https://facilitator.bankofai.io/supported)，其扩展列表仅包含 `erc20ApprovalGasSponsoring`。官方服务支持 Nile 网络，不等于已经提供本文的首次授权资源赞助。

本文目前没有列出已核实的第三方公开 Nile 赞助服务。公开体验入口需要同时说明服务名称、运营主体、访问地址、支持资产与赞助条件；应确认 Facilitator 的 `/supported` 能力声明和业务路由返回的扩展信息相互匹配。

如果希望现在开始开发自己的 Facilitator，可以从 [SDK 接入说明](https://github.com/BofAI/x402/blob/v1.2.0/typescript/packages/extensions/src/trc20-approval-resource-sponsoring/README.md#facilitator)开始，在 Nile 配置资源账户与支持的测试代币，运行首次授权和付款流程。[公开的 Nile 集成测试源码](https://github.com/BofAI/x402/blob/v1.2.0/typescript/packages/mechanisms/tron/test/integrations/trc20-approval-resource-sponsoring.nile.test.ts)可作为验证路径的参考；测试源码存在并不代表某个公开服务当前可用，也不构成本次运行结果。

## 8. 开发者参考

本文基于 BANK OF AI x402 SDK **v1.2.0**，对应 `@bankofai/x402-tron@1.2.0` 与 `@bankofai/x402-extensions@1.2.0`。接口与支持范围见 [v1.2.0 发布说明](https://github.com/BofAI/x402/releases/tag/v1.2.0)。

- [BANK OF AI x402：面向 AI 智能体的稳定币支付方案](https://docs.bankofai.io/zh-Hans/devnotes/x402-stablecoin-payments-for-agents/)：协议背景与支付方案。
- [TRC-20 Approval Resource Sponsoring 规范](https://github.com/BofAI/x402/blob/v1.2.0/specs/extensions/trc20_approval_resource_sponsoring.md)：支持范围与授权约束。
- [SDK 接入说明](https://github.com/BofAI/x402/blob/v1.2.0/typescript/packages/extensions/src/trc20-approval-resource-sponsoring/README.md)：客户端、服务端和 Facilitator 接入。
- [Security considerations](https://github.com/BofAI/x402/blob/v1.2.0/specs/extensions/trc20_approval_resource_sponsoring.md#security-considerations)：生产安全评估需要考虑的协议边界。
