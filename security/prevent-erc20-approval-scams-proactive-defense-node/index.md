# Prevent ERC20 Approval Scams: Building a Proactive Ethereum Defense RPC Node With Cloudflare Workers


随着区块链应用的普及,欺诈行为日益频繁.据 Chainalysis 报告,2024年加密资产犯罪活动导致损失超过40亿美元,其中因授权类诈骗损失达20%.这类欺诈通常涉及诱导用户签署恶意授权交易,而对于普通用户而言,这类授权功能往往是非必要的.

## 常见的欺诈类型及典型案例

1. **钓鱼网站诈骗**：攻击者伪装成知名项目或钱包的官网,引诱用户进行无限授权.如著名的MetaMask假网站案件,数百万美元资产因此被盗.
2. **空投诈骗**：虚假的免费代币或NFT空投,引导用户签署无限授权合约,如近期的Blur假空投诈骗导致大量用户资产被窃取.
3. **虚假挖矿投资**：承诺高额收益,实际骗取用户授权后转移资产,如 "Luna Mining" 诈骗事件.
4. **DEX授权诈骗**：攻击者在去中心化交易所内误导用户进行无限授权,如 PancakeSwap 假合约事件.

由于普通用户日常使用钱包并不经常涉及需要授权的场景,因此通过设置主动防御节点,能够有效降低此类风险.这种节点会在用户签署交易之后;真正广播到区块链网络之前,自动检查交易内容并过滤掉可能存在危险的授权交易.这种机制不仅适用于以太坊,也适用于其他基于EVM的区块链网络(例如币安智能链 BSC; Polygon; Tron等).

## 如何有效预防

- 谨慎访问未知链接,避免轻易授权.
- 签署钱包交易前仔细核对合约地址与交易详情.
- 使用主动防御节点或自定义安全节点对所有交易进行自动检查.
- 定期审查钱包授权状态,撤销不必要的授权.

## 技术实现原理与操作细节

为了防范上述欺诈风险,本文采用 **Cloudflare Worker** 与 **Infura** 搭建主动防御节点，对高风险授权交易进行拦截。该节点作为交易检查中继，工作流程如下：

1. 用户在钱包中签署交易后,交易首先发送至 Cloudflare Worker 进行安全检查
2. 只有通过检查的交易,才会被转发至以太坊主网节点(例如 Infura)进行广播

此外,由于 Cloudflare Worker 部署在边缘网络,还能有效隐藏用户真实 IP,从而增强隐私保护.

具体拦截规则包括:

- 拦截高风险授权函数签名（例如`approve`;`increaseAllowance`;`setApprovalForAll`）.
- 阻止与黑名单合约地址进行任何交互.
- 设置资产转移白名单,只允许向信任地址进行转账.

## Cloudflare Worker详细代码实现

```js
// 此代码未经测试;实际部署中需要测试
const BLOCKED_SELECTORS = [
  '0x095ea7b3', // approve
  '0x39509351', // increaseAllowance
  '0xa22cb465'  // setApprovalForAll
];

const BLACKLIST_CONTRACTS = new Set([
  '0xdeaddeaddeaddeaddeaddeaddeaddeaddeadbeef',
  '0x1234567890abcdef1234567890abcdef12345678'
]);

const ENABLE_WHITELIST = true;
const WHITELIST_RECIPIENTS = new Set([
  '0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa',
  '0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb'
]);

export default {
  async fetch(request) {
    if (request.method !== 'POST') {
      return new Response('Only POST allowed', { status: 405 });
    }

    const body = await request.json();
    const tx = body?.params?.[0];

    const to = tx?.to?.toLowerCase();
    const data = tx?.data?.toLowerCase();

    if (data && BLOCKED_SELECTORS.some(sig => data.startsWith(sig))) {
      return new Response(JSON.stringify({ error: '已拦截危险授权交易' }), { status: 403 });
    }

    if (to && BLACKLIST_CONTRACTS.has(to)) {
      return new Response(JSON.stringify({ error: '目标地址属于黑名单合约' }), { status: 403 });
    }

    if (ENABLE_WHITELIST && to && !WHITELIST_RECIPIENTS.has(to)) {
      return new Response(JSON.stringify({ error: '非白名单地址禁止转账' }), { status: 403 });
    }

    const response = await fetch('https://mainnet.infura.io/v3/YOUR_INFURA_PROJECT_ID', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body)
    });

    return response;
  }
};
```

## 具体部署操作步骤

1. 登录[Cloudflare Workers](https://workers.cloudflare.com/)后台.
2. 创建新的Worker,复制上述代码到编辑器中.
3. 替换代码中的`YOUR_INFURA_PROJECT_ID`为你个人的Infura项目ID.
4. 点击"保存并部署",获得Worker节点URL,如 `https://your-worker.workers.dev`.

## 钱包配置实例参考

- 进入钱包设置 --> 网络 --> 添加新网络.
- 网络名称填入"主动防御节点".
- RPC URL 填入上述 Worker URL.
- Chain ID填写为1,符号ETH.(有些钱包只需要填写prc地址)

设置后,钱包发出的交易都会先经过此节点检查再上链.

## 实际使用注意事项

如果你经常使用去中心化交易所(DEX),直接使用此节点可能导致交易失败,这是因为DEX需要授权函数调用.

建议方案包括：

- 将安全且频繁使用的DEX授权合约加入白名单.
- 对于需要频繁授权的交易,使用单独的钱包,以减少资产风险.

## 总结与扩展建议

通过主动防御节点,我们能有效防范区块链授权类诈骗,保护资产安全.本方案除了以太坊外,同样适用于币安智能链(BSC);Polygon等EVM兼容链,实际使用时只需更换相应RPC地址.

未来还可考虑集成诈骗地址实时数据库;报警通知系统（如Telegram或钉钉）,进一步提升安全性.


---

> 作者: <no value>  
> URL: https://www.lucas6.xyz/security/prevent-erc20-approval-scams-proactive-defense-node/  

