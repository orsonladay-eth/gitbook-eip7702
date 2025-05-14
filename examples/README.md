# 实际应用示例

在本节中，我们将展示EIP-7702的实际应用案例。具体来说，我们将演示如何在一个交易中同时执行ETH和代币转账：

1. **ETH转账**：向接收方直接发送ETH
2. **代币转账**：调用ERC20代币合约的transfer方法
3. **批量执行**：通过BatchCallDelegation合约一次性完成上述操作

## 完整流程示例

### 1. 检查转账前余额
```typescript
const ethBalanceBefore = await clients.public.getBalance({ address: BOB });
const tokenBalanceBefore = await clients.public.readContract({
    abi: tokenAbi,
    address: SIMPLE_TOKEN,
    functionName: "balanceOf",
    args: [BOB],
});
```

### 2. 执行批量转账
```typescript
// 批量执行ETH和代币转账
await clients.wallet.writeContract({
    abi: batchAbi,
    address: account.address,
    functionName: "execute",
    args: [
        [
            // ETH转账
            {
                data: "0x", // 空数据表示简单的ETH转账
                to: BOB,
                value: parseEther("0.0001"), // 转账0.0001 ETH
            },
            // 代币转账
            {
                data: tokenTransferData,
                to: SIMPLE_TOKEN, // 代币合约地址
                value: 0n, // 不发送ETH
            },
        ],
    ],
    authorizationList: [authorization],
});
```

### 3. 验证转账结果
```typescript
const ethBalanceAfter = await clients.public.getBalance({ address: BOB });
const tokenBalanceAfter = await clients.public.readContract({
    abi: tokenAbi,
    address: SIMPLE_TOKEN,
    functionName: "balanceOf",
    args: [BOB],
});
``` 