# 用户界面

在这个里程碑中，我们添加了从池子中移除流动性和收集累积费用的能力。因此，我们需要在用户界面中反映这些变化，以允许用户移除流动性。

## 获取头寸

为了让用户选择要移除多少流动性，我们首先需要从池子中获取用户的头寸。为了使这更容易，我们可以在Manager合约中添加一个辅助函数，该函数将返回用户在特定池子中的头寸：

```solidity
function getPosition(GetPositionParams calldata params)
    public
    view
    returns (
        uint128 liquidity,
        uint256 feeGrowthInside0LastX128,
        uint256 feeGrowthInside1LastX128,
        uint128 tokensOwed0,
        uint128 tokensOwed1
    )
{
    IUniswapV3Pool pool = getPool(params.tokenA, params.tokenB, params.fee);

    (
        liquidity,
        feeGrowthInside0LastX128,
        feeGrowthInside1LastX128,
        tokensOwed0,
        tokensOwed1
    ) = pool.positions(
        keccak256(
            abi.encodePacked(
                params.owner,
                params.lowerTick,
                params.upperTick
            )
        )
    );
}
```

这将使我们在前端免于计算池子地址和头寸密钥。

然后，在用户输入头寸范围后，我们可以尝试获取头寸：

```js
const getAvailableLiquidity = debounce((amount, isLower) => {
  const lowerTick = priceToTick(isLower ? amount : lowerPrice);
  const upperTick = priceToTick(isLower ? upperPrice : amount);

  const params = {
    tokenA: token0.address,
    tokenB: token1.address,
    fee: fee,
    owner: account,
    lowerTick: nearestUsableTick(lowerTick, feeToSpacing[fee]),
    upperTick: nearestUsableTick(upperTick, feeToSpacing[fee]),
  }

  manager.getPosition(params)
    .then(position => setAvailableAmount(position.liquidity.toString()))
    .catch(err => console.error(err));
}, 500);
```

## 获取池子地址

由于我们需要在池子上调用`burn`和`collect`，我们仍然需要在前端计算池子的地址。回想一下，池子地址是使用`CREATE2`操作码计算的，这需要一个盐值和合约代码的哈希。幸运的是，Ether.js有`getCreate2Address`函数，允许在JavaScript中计算`CREATE2`：

```js
const sortTokens = (tokenA, tokenB) => {
  return tokenA.toLowerCase() < tokenB.toLowerCase ? [tokenA, tokenB] : [tokenB, tokenA];
}

const computePoolAddress = (factory, tokenA, tokenB, fee) => {
  [tokenA, tokenB] = sortTokens(tokenA, tokenB);

  return ethers.utils.getCreate2Address(
    factory,
    ethers.utils.keccak256(
      ethers.utils.solidityPack(
        ['address', 'address', 'uint24'],
        [tokenA, tokenB, fee]
      )),
    poolCodeHash
  );
}
```

然而，池子的代码哈希必须硬编码，因为我们不想在前端存储其代码来计算哈希。所以，我们将使用Forge来获取哈希：

```shell
$ forge inspect UniswapV3Pool bytecode| xargs cast keccak 
0x...
```

然后在JS常量中使用输出值：
```js
const poolCodeHash = "0x9dc805423bd1664a6a73b31955de538c338bac1f5c61beb8f4635be5032076a2";
```

## 移除流动性

在获得流动性数量和池子地址后，我们准备调用`burn`：

```js
const removeLiquidity = (e) => {
  e.preventDefault();

  if (!token0 || !token1) {
    return;
  }

  setLoading(true);

  const lowerTick = nearestUsableTick(priceToTick(lowerPrice), feeToSpacing[fee]);
  const upperTick = nearestUsableTick(priceToTick(upperPrice), feeToSpacing[fee]);

  pool.burn(lowerTick, upperTick, amount)
    .then(tx => tx.wait())
    .then(receipt => {
      if (!receipt.events[0] || receipt.events[0].event !== "Burn") {
        throw Error("Missing Burn event after burning!");
      }

      const amount0Burned = receipt.events[0].args.amount0;
      const amount1Burned = receipt.events[0].args.amount1;

      return pool.collect(account, lowerTick, upperTick, amount0Burned, amount1Burned)
    })
    .then(tx => tx.wait())
    .then(() => toggle())
    .catch(err => console.error(err));
}
```
如果燃烧成功，我们立即调用`collect`来收集在燃烧过程中释放的代币数量。
