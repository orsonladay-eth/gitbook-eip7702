# 客户端实现

## 批量转账示例

下面是一个完整的批量转账示例，展示了如何使用EIP-7702执行ETH和代币的批量转账：

```typescript
import { createWalletClient, http, parseEther, parseAbi, createPublicClient, encodeFunctionData, formatEther, formatUnits } from "viem";
import { sepolia } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";
import { eip7702Actions } from "viem/experimental";

const ALICE_PK = "";
const BOB = "";
// 更新为您部署的合约地址
const BATCH_CALL_DELEGATION = "0xb764372fd28c302e29ce6f1972f06d99300de69f";
const SIMPLE_TOKEN = "0xf74f2008215a358d09fa4bd9cf4e3027cc904b1a";

const main = async () => {
    console.log(sepolia.rpcUrls);
    const account = privateKeyToAccount(ALICE_PK);

    const clients = {
        wallet: createWalletClient({ chain: sepolia, transport: http(), account }).extend(eip7702Actions()),
        public: createPublicClient({ chain: sepolia, transport: http() }),
    };

    // 签署委托
    const authorization = await clients.wallet.signAuthorization({
        contractAddress: BATCH_CALL_DELEGATION,
    });

    const tokenAbi = parseAbi([
        "function balanceOf(address) view returns (uint256)",
        "function transfer(address,uint256) returns (bool)",
        "function name() view returns (string)",
        "function symbol() view returns (string)",
        "function decimals() view returns (uint8)",
        "function totalSupply() view returns (uint256)",
    ]);

    // 获取代币信息
    const tokenName = await clients.public.readContract({
        abi: tokenAbi,
        address: SIMPLE_TOKEN,
        functionName: "name",
    });

    const tokenSymbol = await clients.public.readContract({
        abi: tokenAbi,
        address: SIMPLE_TOKEN,
        functionName: "symbol",
    });

    const tokenDecimals = await clients.public.readContract({
        abi: tokenAbi,
        address: SIMPLE_TOKEN,
        functionName: "decimals",
    });

    console.log(`>>> 代币信息: ${tokenName} (${tokenSymbol}), 小数位: ${tokenDecimals}`);

    // 检查转账前的余额
    const ethBalanceBefore = await clients.public.getBalance({ address: BOB });
    console.log(`>>> Bob的ETH余额 (转账前): ${formatEther(ethBalanceBefore)} ETH`);

    const tokenBalanceBefore = await clients.public.readContract({
        abi: tokenAbi,
        address: SIMPLE_TOKEN,
        functionName: "balanceOf",
        args: [BOB],
    });
    console.log(`>>> Bob的${tokenSymbol}余额 (转账前): ${formatUnits(tokenBalanceBefore, tokenDecimals)} ${tokenSymbol}`);

    // 准备代币转账数据
    const tokenTransferAmount = 100n * 10n ** BigInt(tokenDecimals);
    console.log(`>>> 代币转账金额: ${formatUnits(tokenTransferAmount, tokenDecimals)} ${tokenSymbol}`);

    // 编码代币转账函数调用
    const tokenTransferData = encodeFunctionData({
        abi: tokenAbi,
        functionName: "transfer",
        args: [BOB, tokenTransferAmount],
    });

    try {
        // 检查Alice是否有足够的代币
        const aliceTokenBalance = await clients.public.readContract({
            abi: tokenAbi,
            address: SIMPLE_TOKEN,
            functionName: "balanceOf",
            args: [account.address],
        });

        console.log(`>>> Alice的${tokenSymbol}余额: ${formatUnits(aliceTokenBalance, tokenDecimals)} ${tokenSymbol}`);

        if (aliceTokenBalance < tokenTransferAmount) {
            console.log(">>> Alice没有足够的代币。尝试从合约部署者转移代币给Alice...");

            // 直接调用代币合约转账给Alice
            await clients.wallet.writeContract({
                abi: tokenAbi,
                address: SIMPLE_TOKEN,
                functionName: "transfer",
                args: [account.address, tokenTransferAmount * 2n],
            });

            console.log(">>> 代币已从部署者转移给Alice");

            // 更新Alice的代币余额
            const updatedAliceBalance = await clients.public.readContract({
                abi: tokenAbi,
                address: SIMPLE_TOKEN,
                functionName: "balanceOf",
                args: [account.address],
            });

            console.log(`>>> Alice更新后的代币余额: ${formatUnits(updatedAliceBalance, tokenDecimals)} ${tokenSymbol}`);
        }

        // 批量执行ETH和代币转账
        console.log(">>> 执行批量转账 (ETH + 代币)...");

        const batchAbi = parseAbi(["function execute((bytes data,address to,uint256 value)[])"]);

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

        console.log(">>> 批量转账成功完成");
    } catch (error) {
        console.error(">>> 批量转账失败:", error);
    }

    // 检查转账后的余额
    const ethBalanceAfter = await clients.public.getBalance({ address: BOB });
    console.log(`>>> Bob的ETH余额 (转账后): ${formatEther(ethBalanceAfter)} ETH`);
    console.log(`>>> ETH增加了: ${formatEther(ethBalanceAfter - ethBalanceBefore)} ETH`);

    const tokenBalanceAfter = await clients.public.readContract({
        abi: tokenAbi,
        address: SIMPLE_TOKEN,
        functionName: "balanceOf",
        args: [BOB],
    });
    console.log(`>>> Bob的${tokenSymbol}余额 (转账后): ${formatUnits(tokenBalanceAfter, tokenDecimals)} ${tokenSymbol}`);
    console.log(`>>> 代币增加了: ${formatUnits(tokenBalanceAfter - tokenBalanceBefore, tokenDecimals)} ${tokenSymbol}`);
};

main();
``` 