# ERC721 概述

让我们从概述 [EIP-721](https://eips.ethereum.org/EIPS/eip-721) 开始，这是定义 NFT 合约的标准。

ERC721 是 ERC20 的一个变体。它们之间的主要区别在于 ERC721 代币是*非同质化的*，也就是说：一个代币与另一个代币并不完全相同。为了区分 ERC721 代币，每个代币都有一个唯一的 ID，这几乎总是代币铸造时的计数器。ERC721 代币还有一个扩展的所有权概念：每个代币的所有者都被跟踪并存储在合约中。这意味着只有由代币 ID 标识的独特代币可以被转移（或批准转移）。

Uniswap V3 流动性头寸和 NFT 的共同点就是这种非同质性：NFT 和流动性头寸都是不可互换的，并且由唯一的 ID 标识。正是这种相似性使我们能够合并这两个概念。

ERC20 和 ERC721 之间最大的区别是后者的 `tokenURI` 函数。作为 ERC721 智能合约实现的 NFT 代币，有链接到存储在外部（而不是在区块链上）的资产。为了将代币 ID 链接到存储在区块链外部的图像（或声音，或其他任何东西），ERC721 定义了 `tokenURI` 函数。该函数预期返回一个指向定义 NFT 代币元数据的 JSON 文件的链接，例如：

```json
{
    "name": "Thor's hammer",
    "description": "Mjölnir, the legendary hammer of the Norse god of thunder.",
    "image": "https://game.example/item-id-8u5h2m.png",
    "strength": 20
}
```

这个例子取自 OpenZeppelin 上的 [ERC721 documentation on OpenZeppelin](https://docs.openzeppelin.com/contracts)

这样的 JSON 文件定义了代币的名称、集合的描述、代币图像的链接以及代币的属性。

另外，我们也可以将 JSON 元数据和代币图像存储在链上。这当然非常昂贵（在链上保存数据是以太坊中最昂贵的操作），但如果我们存储模板，我们可以使其更便宜。同一集合中的所有代币都有相似的元数据（大部分相同，但每个代币的图像链接和属性不同）和视觉效果。对于后者，我们可以使用 SVG，这是一种类似 HTML 的格式，而 HTML 是一种很好的模板语言。

当在链上存储 JSON 元数据和 SVG 时，`tokenURI` 函数不是返回链接，而是直接返回 JSON 元数据，使用[数据 URI 方案](https://en.wikipedia.org/wiki/Data_URI_scheme#Syntax) 对其进行编码。SVG 图像也会被内联，无需进行外部请求来下载代币元数据和图像。
