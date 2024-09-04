# Factory合约

Uniswap的设计假设存在多个独立的Pool合约，每个池子处理一对代币的交换。当我们想要交换两个没有池子的代币时，这看起来是个问题——如果没有池子，就无法进行交换。然而，我们仍然可以进行中间交换：首先交换到一个与两个代币中任一个有配对的代币，然后将这个代币交换到目标代币。这个过程可以更深入，涉及更多的中间代币。不过，手动执行这个过程很麻烦，幸运的是，我们可以通过在智能合约中实现这个过程来简化它。

Factory合约是一个服务于多个目的的合约：

1. 它作为Pool合约的中央注册表。使用Factory，你可以找到所有已部署的池子、它们的代币和地址。

2. 它简化了Pool合约的部署。EVM允许从智能合约部署智能合约——Factory使用这个特性使池子部署变得轻而易举。

3. 它使池子地址可预测，并允许在不调用注册表的情况下计算它们。这使得池子易于发现。

让我们来构建Factory合约！但在此之前，我们需要学习一些新东西。

## `CREATE`和`CREATE2`操作码

EVM有两种部署合约的方式：通过`CREATE`或通过`CREATE2`操作码。它们之间的唯一区别是新合约地址的生成方式：

1. `CREATE`使用部署者账户的`nonce`来生成合约地址（伪代码）：

    ```
    KECCAK256(deployer.address, deployer.nonce)
    ```

    `nonce`是一个账户特定的交易计数器。在新合约地址生成中使用`nonce`使得在其他合约或链下应用中计算地址变得困难，主要是因为要找到合约部署时的nonce，需要扫描历史账户交易。

2. `CREATE2`使用自定义的*salt*来生成合约地址。这只是开发者选择的任意字节序列，用于使地址生成具有确定性（并减少冲突的机会）。

    ```
    KECCAK256(deployer.address, salt, contractCodeHash)
    ```

我们需要知道这个区别，因为Factory在部署Pool合约时使用`CREATE2`，这样池子就能获得唯一且确定性的地址，可以在其他合约和链下应用中计算。具体来说，对于salt，Factory使用这些池子参数计算哈希：

```solidity
keccak256(abi.encodePacked(token0, token1, tickSpacing))
```
token0和token1是池子代币的地址，而tickSpacing是我们接下来要学习的内容。

## Tick间距

回想一下swap函数中的循环：

```solidity
while (
    state.amountSpecifiedRemaining > 0 &&
    state.sqrtPriceX96 != sqrtPriceLimitX96
) {
    ...
    (step.nextTick, ) = tickBitmap.nextInitializedTickWithinOneWord(...);
    (state.sqrtPriceX96, step.amountIn, step.amountOut) = SwapMath.computeSwapStep(...);
    ...
}
```

这个循环通过向任一方向迭代来找到具有一些流动性的已初始化的刻度。然而，这种迭代是一个昂贵的操作：如果一个刻度距离很远，代码需要经过当前刻度和目标刻度之间的所有刻度，这会消耗gas。为了使这个循环更加gas高效，Uniswap池子有`tickSpacing`设置，它设置了刻度之间的距离：距离越宽，交换的gas效率就越高。

然而，刻度间距越宽，精度就越低。低波动性对（例如稳定币对）需要更高的精度，因为这类对中的价格变动较小。中等和高波动性对需要较低的精度，因为这类对中的价格变动较大。为了处理这种多样性，Uniswap允许在部署对时选择刻度间距。Uniswap允许部署者从以下选项中选择：10、60或200。为了简单起见，我们只使用10和60。

从技术角度来说，刻度索引只能是`tickSpacing`的倍数：如果`tickSpacing`是10，那么只有10的倍数才能作为有效的刻度索引（10、20、5000、5010，但不能是8、12、5001等）。然而，这一点很重要，这并不适用于当前价格——它仍然可以是任何刻度，因为我们希望它尽可能精确。`tickSpacing`只应用于价格范围。

因此，每个池子都由这组参数唯一标识：

1. `token0`，
2. `token1`，
3. `tickSpacing`；

> 是的，可以存在具有相同代币但不同刻度间距的池子。

Factory合约使用这组参数作为池子的唯一标识符，并将其作为salt来生成新的池子合约地址。

> 从现在开始，我们将假设所有池子的刻度间距为60，对于稳定币对我们将使用10。请注意，只有能被这些值整除的刻度才能在刻度位图中被标记为已初始化。例如，当刻度间距为60时，只有-120、-60、0、60、120等刻度可以被初始化并用于流动性范围。

## Factory实现

在Factory的构造函数中，我们需要初始化支持的刻度间距：

```solidity
// src/UniswapV3Factory.sol
contract UniswapV3Factory is IUniswapV3PoolDeployer {
    mapping(uint24 => bool) public tickSpacings;
    constructor() {
        tickSpacings[10] = true;
        tickSpacings[60] = true;
    }

    ...
```

> 我们本可以将它们设为常量，但为了后面的里程碑（不同的刻度间距将有不同的交换费用金额），我们需要将其作为映射。

Factory合约是一个只有一个函数`createPool`的合约。该函数以我们在创建池子之前需要进行的必要检查开始：

```solidity
// src/UniswapV3Factory.sol
contract UniswapV3Factory is IUniswapV3PoolDeployer {
    PoolParameters public parameters;
    mapping(address => mapping(address => mapping(uint24 => address)))
        public pools;

    ...

    function createPool(
        address tokenX,
        address tokenY,
        uint24 tickSpacing
    ) public returns (address pool) {
        if (tokenX == tokenY) revert TokensMustBeDifferent();
        if (!tickSpacings[tickSpacing]) revert UnsupportedTickSpacing();

        (tokenX, tokenY) = tokenX < tokenY
            ? (tokenX, tokenY)
            : (tokenY, tokenX);

        if (tokenX == address(0)) revert TokenXCannotBeZero();
        if (pools[tokenX][tokenY][tickSpacing] != address(0))
            revert PoolAlreadyExists();
        
        ...
```

注意，这是我们第一次对代币进行排序：

```solidity
(tokenX, tokenY) = tokenX < tokenY
    ? (tokenX, tokenY)
    : (tokenY, tokenX);
```

从现在开始，我们也会期望池子代币地址是经过排序的，即当排序时，`token0`在`token1`之前。我们强制执行这一点是为了使salt（和池子地址）的计算保持一致。

> 这个变化也影响了我们在测试和部署脚本中部署代币的方式：我们需要确保WETH始终是`token0`，以使Solidity中的价格计算更简单（否则，我们就需要使用分数价格，比如1/5000）。如果在你的测试中WETH不是`token0`，请更改代币部署的顺序。

之后，我们准备池子参数并部署一个池子：

```solidity
parameters = PoolParameters({
    factory: address(this),
    token0: tokenX,
    token1: tokenY,
    tickSpacing: tickSpacing
});

pool = address(
    new UniswapV3Pool{
        salt: keccak256(abi.encodePacked(tokenX, tokenY, tickSpacing))
    }()
);

delete parameters;
```

这段代码看起来很奇怪，因为`parameters`没有被使用。Uniswap使用[控制反转](https://en.wikipedia.org/wiki/Inversion_of_control)在部署期间将参数传递给池子。让我们看看更新后的Pool合约构造函数：

```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    ...
    constructor() {
        (factory, token0, token1, tickSpacing) = IUniswapV3PoolDeployer(
            msg.sender
        ).parameters();
    }
    ..
}
```

啊哈！Pool期望其部署者实现`IUniswapV3PoolDeployer`接口（该接口只定义了`parameters()`getter），并在部署期间的构造函数中调用它来获取参数。这个流程看起来是这样的：

1. `Factory`：定义`parameters`状态变量（实现`IUniswapV3PoolDeployer`）并在部署池子之前设置它。

2. `Factory`：部署一个池子。

3. `Pool`：在构造函数中，调用其部署者的`parameters()`函数，并期望返回池子参数。

4. `Factory`：调用`delete parameters;`来清理`parameters`状态变量的槽位并减少gas消耗。这是一个临时状态变量，只在调用`createPool()`期间有值。

在创建池子后，我们将其保存在`pools`映射中（这样可以通过其代币找到它）并发出一个事件：

```solidity
    pools[tokenX][tokenY][tickSpacing] = pool;
    pools[tokenY][tokenX][tickSpacing] = pool;

    emit PoolCreated(tokenX, tokenY, tickSpacing, pool);
}
```

## 池子初始化

正如你从上面的代码中注意到的，我们不再在Pool的构造函数中设置`sqrtPriceX96`和`tick`——这现在在一个单独的函数`initialize`中完成，需要在池子部署后调用：

```solidity
// src/UniswapV3Pool.sol
function initialize(uint160 sqrtPriceX96) public {
    if (slot0.sqrtPriceX96 != 0) revert AlreadyInitialized();

    int24 tick = TickMath.getTickAtSqrtRatio(sqrtPriceX96);

    slot0 = Slot0({sqrtPriceX96: sqrtPriceX96, tick: tick});
}
```

所以这就是我们现在部署池子的方式：

```solidity
UniswapV3Factory factory = new UniswapV3Factory();
UniswapV3Pool pool = UniswapV3Pool(factory.createPool(token0, token1, tickSpacing));
pool.initialize(sqrtP(currentPrice));
```

## `PoolAddress`库

现在让我们实现一个库，它将帮助我们从其他合约计算池子合约地址。这个库只有一个函数，`computeAddress`：

```solidity
// src/lib/PoolAddress.sol
library PoolAddress {
    function computeAddress(
        address factory,
        address token0,
        address token1,
        uint24 tickSpacing
    ) internal pure returns (address pool) {
        require(token0 < token1);
        ...
```

该函数需要知道池子参数（它们用于构建salt）和Factory合约地址。它期望代币是已排序的，我们之前讨论过这一点。

现在，函数的核心部分：

```solidity
pool = address(
    uint160(
        uint256(
            keccak256(
                abi.encodePacked(
                    hex"ff",
                    factory,
                    keccak256(
                        abi.encodePacked(token0, token1, tickSpacing)
                    ),
                    keccak256(type(UniswapV3Pool).creationCode)
                )
            )
        )
    )
);
```

这就是`CREATE2`在底层计算新合约地址时所做的。让我们来解析一下：

1. 首先，我们计算salt（`abi.encodePacked(token0, token1, tickSpacing)`）并对其进行哈希；
2. 然后，我们获取Pool合约代码（`type(UniswapV3Pool).creationCode`）并也对其进行哈希；
3. 接着，我们构建一个字节序列，包括：`0xff`、Factory合约地址、哈希后的salt和哈希后的Pool合约代码；
4. 最后，我们对这个序列进行哈希并将其转换为地址。

这些步骤实现了合约地址生成，正如[EIP-1014](https://eips.ethereum.org/EIPS/eip-1014)中定义的那样，这个EIP添加了`CREATE2`操作码。让我们更仔细地看看构成哈希字节序列的值：

1. `0xff`，如EIP中定义的，用于区分由`CREATE`和`CREATE2`生成的地址；
2. `factory`是部署者的地址，在我们的情况下是Factory合约；
3. salt之前已经讨论过——它唯一标识一个池子；
4. 哈希后的合约代码是为了防止冲突：不同的合约可能有相同的salt，但它们的代码哈希会不同。

因此，根据这个方案，合约地址是唯一标识这个合约的值的哈希，包括其部署者、代码和唯一参数。我们可以从任何地方使用这个函数来找到池子地址，而无需进行任何外部调用，也无需查询工厂。

## Manager和Quoter的简化接口

在Manager和Quoter合约中，我们不再需要向用户询问池子地址！这使得与合约的交互更加容易，因为用户不需要知道池子地址，他们只需要知道代币。然而，用户还需要指定刻度间距，因为它包含在池子的salt中。

此外，我们不再需要向用户询问`zeroForOne`标志，因为现在我们可以通过代币排序来确定它。当"from token"小于"to token"时，`zeroForOne`为真，因为池子的`token0`总是小于`token1`。同样，当"from token"大于"to token"时，`zeroForOne`总是假。

> 地址是哈希，而哈希是数字，所以我们可以在比较地址时说"小于"或"大于"。
