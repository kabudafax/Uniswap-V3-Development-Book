# NFT 管理合约

我们不会在池合约中添加 NFT 相关功能——我们需要一个单独的合约来合并 NFT 和流动性头寸。回想一下，在我们实现的过程中，我们构建了 `UniswapV3Manager` 合约以便于与池合约交互（使一些计算更简单，并启用多池交换）。这个合约很好地展示了如何扩展核心 Uniswap 合约。我们将进一步推进这个想法。

我们需要一个管理合约，它将实现 ERC721 标准并管理流动性头寸。该合约将具有标准的 NFT 功能（铸造、销毁、转移、余额和所有权跟踪等），并允许向池提供和移除流动性。合约需要成为池中流动性的实际所有者，因为我们不希望用户在不铸造代币的情况下添加流动性，也不希望在不销毁代币的情况下移除全部流动性。我们希望每个流动性头寸都与一个 NFT 代币相关联，并且我们希望它们保持同步。

让我们看看新合约中将会有哪些函数：

1. 由于它将是一个 NFT 合约，它将包含所有 ERC721 函数，包括 `tokenURI`，该函数返回 NFT 代币图像的 URI；

2. `mint` 和 `burn` 用于同时铸造和销毁流动性和 NFT 代币；

3. `addLiquidity` 和 `removeLiquidity` 用于在现有头寸中添加和移除流动性；

4. `collect`，用于在移除流动性后收集代币。

好的，让我们开始编码。

## 最小合约

由于我们不想从头实现 ERC721 标准，我们将使用一个库。我们的依赖项中已经有 [Solmate](https://github.com/transmissions11/solmate)，所以我们将使用[它的 ERC721 实现](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC721.sol)。

> 使用 [OpenZeppelin 的 ERC721 实现](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/master/contracts/token/ERC721) 也是一个选择，但我更喜欢 Solmate 的气体优化合约。

这将是 NFT 管理合约的最基本实现：


```solidity
contract UniswapV3NFTManager is ERC721 {
    address public immutable factory;

    constructor(address factoryAddress)
        ERC721("UniswapV3 NFT Positions", "UNIV3")
    {
        factory = factoryAddress;
    }

    function tokenURI(uint256 tokenId)
        public
        view
        override
        returns (string memory)
    {
        return "";
    }
}
```
`tokenURI` 将返回一个空字符串，直到我们实现元数据和 SVG 渲染器。我们添加了这个存根，以便在我们处理合约其余部分时 Solidity 编译器不会失败（Solmate ERC721 合约中的 `tokenURI` 函数是虚拟的，所以我们必须实现它）。

## 铸造

如我们之前讨论的，铸造将涉及两个操作：向池中添加流动性和铸造 NFT。

为了保持池流动性头寸和 NFT 之间的链接，我们需要一个映射和一个结构：

```solidity
struct TokenPosition {
    address pool;
    int24 lowerTick;
    int24 upperTick;
}
mapping(uint256 => TokenPosition) public positions;
```

要找到一个头寸，我们需要：

1. 池地址；

2. 所有者地址；

3. 头寸的边界（下限和上限刻度）。

由于 NFT 管理合约将成为通过它创建的所有头寸的所有者，我们不需要存储头寸的所有者地址，我们只需存储其余数据。`positions` 映射中的键是代币 ID；该映射将 NFT ID 链接到查找流动性头寸所需的头寸数据。

让我们来实现铸造：

```solidity
struct MintParams {
    address recipient;
    address tokenA;
    address tokenB;
    uint24 fee;
    int24 lowerTick;
    int24 upperTick;
    uint256 amount0Desired;
    uint256 amount1Desired;
    uint256 amount0Min;
    uint256 amount1Min;
}

function mint(MintParams calldata params) public returns (uint256 tokenId) {
    ...
}
```

铸造参数与 `UniswapV3Manager` 的参数相同，但增加了 `recipient`，这将允许为另一个地址铸造 NFT。

在 `mint` 函数中，我们首先向池中添加流动性：


```solidity
IUniswapV3Pool pool = getPool(params.tokenA, params.tokenB, params.fee);

(uint128 liquidity, uint256 amount0, uint256 amount1) = _addLiquidity(
    AddLiquidityInternalParams({
        pool: pool,
        lowerTick: params.lowerTick,
        upperTick: params.upperTick,
        amount0Desired: params.amount0Desired,
        amount1Desired: params.amount1Desired,
        amount0Min: params.amount0Min,
        amount1Min: params.amount1Min
    })
);
```

`_addLiquidity` 与 `UniswapV3Manager` 合约中 `mint` 函数的主体内容相同：它将刻度转换为 $\sqrt(P)$，计算流动性数量，并调用 `pool.mint()`。

接下来，我们铸造一个 NFT：

```solidity
tokenId = nextTokenId++;
_mint(params.recipient, tokenId);
totalSupply++;
```
`tokenId` 被设置为当前的 `nextTokenId`，然后后者递增。`_mint` 函数由 Solmate 的 ERC721 合约提供。在铸造新代币后，我们更新 `totalSupply`。

最后，我们需要存储有关新代币和新头寸的信息：

```solidity
TokenPosition memory tokenPosition = TokenPosition({
    pool: address(pool),
    lowerTick: params.lowerTick,
    upperTick: params.upperTick
});

positions[tokenId] = tokenPosition;
```

这将在后续帮助我们通过代币 ID 找到流动性头寸。

## 添加流动性

接下来，我们将实现一个函数，用于向现有头寸添加流动性，适用于我们想要在已有一些流动性的头寸中增加更多流动性的情况。在这种情况下，我们不想铸造 NFT，而只是增加现有头寸中的流动性数量。为此，我们只需要提供代币 ID 和代币数量：


```solidity
function addLiquidity(AddLiquidityParams calldata params)
    public
    returns (
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    )
{
    TokenPosition memory tokenPosition = positions[params.tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    (liquidity, amount0, amount1) = _addLiquidity(
        AddLiquidityInternalParams({
            pool: IUniswapV3Pool(tokenPosition.pool),
            lowerTick: tokenPosition.lowerTick,
            upperTick: tokenPosition.upperTick,
            amount0Desired: params.amount0Desired,
            amount1Desired: params.amount1Desired,
            amount0Min: params.amount0Min,
            amount1Min: params.amount1Min
        })
    );
}
```

这个函数确保存在一个已有的token，并使用现有position的参数调用`pool.mint()`。

## 移除流动性

回想一下，在`UniswapV3Manager`合约中，我们没有实现`burn`函数，因为我们希望用户成为流动性position的所有者。现在，我们希望NFT管理器成为所有者。我们可以在其中实现流动性销毁：

```solidity
struct RemoveLiquidityParams {
    uint256 tokenId;
    uint128 liquidity;
}

function removeLiquidity(RemoveLiquidityParams memory params)
    public
    isApprovedOrOwner(params.tokenId)
    returns (uint256 amount0, uint256 amount1)
{
    TokenPosition memory tokenPosition = positions[params.tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    IUniswapV3Pool pool = IUniswapV3Pool(tokenPosition.pool);

    (uint128 availableLiquidity, , , , ) = pool.positions(
        poolPositionKey(tokenPosition)
    );
    if (params.liquidity > availableLiquidity) revert NotEnoughLiquidity();

    (amount0, amount1) = pool.burn(
        tokenPosition.lowerTick,
        tokenPosition.upperTick,
        params.liquidity
    );
}
```

我们再次检查提供的token ID是否有效。我们还需要确保position有足够的流动性可以销毁。

## 收集代币

NFT管理器合约也可以在销毁流动性后收集代币。注意，收集的代币会被发送给`msg.sender`，因为合约是代表调用者管理流动性的：


```solidity
struct CollectParams {
    uint256 tokenId;
    uint128 amount0;
    uint128 amount1;
}

function collect(CollectParams memory params)
    public
    isApprovedOrOwner(params.tokenId)
    returns (uint128 amount0, uint128 amount1)
{
    TokenPosition memory tokenPosition = positions[params.tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    IUniswapV3Pool pool = IUniswapV3Pool(tokenPosition.pool);

    (amount0, amount1) = pool.collect(
        msg.sender,
        tokenPosition.lowerTick,
        tokenPosition.upperTick,
        params.amount0,
        params.amount1
    );
}
```

## 销毁

最后是销毁。与合约的其他函数不同，这个函数不会对池子做任何操作：它只是销毁一个NFT。要销毁NFT，底层position必须为空，并且代币必须已被收集。因此，如果我们想要销毁NFT，我们需要：

1. 调用`removeLiquidity`并移除整个position的流动性；

2. 调用`collect`来收集销毁position后的代币；

3. 调用`burn`来销毁token。


```solidity
function burn(uint256 tokenId) public isApprovedOrOwner(tokenId) {
    TokenPosition memory tokenPosition = positions[tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    IUniswapV3Pool pool = IUniswapV3Pool(tokenPosition.pool);
    (uint128 liquidity, , , uint128 tokensOwed0, uint128 tokensOwed1) = pool
        .positions(poolPositionKey(tokenPosition));

    if (liquidity > 0 || tokensOwed0 > 0 || tokensOwed1 > 0)
        revert PositionNotCleared();

    delete positions[tokenId];
    _burn(tokenId);
    totalSupply--;
}
```

就是这样！
