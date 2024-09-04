# Uniswap V3 开发手册

<p align="center">
<img src="./src/images/cover.png" alt="Uniswap V3 开发书籍封面" width="360"/>
</p>


<p align="center">
👉&nbsp;<a href="https://uniswapv3book.com/">在线阅读</a>&nbsp;&nbsp;|&nbsp;&nbsp;<a href="https://uniswapv3book.com/print.html">打印或保存为PDF</a>&nbsp;👈
</p>

*本书翻译自[Uniswap V3 Development Book](https://github.com/Jeiwan/uniswapv3-book)，非常感谢作者 Jeiwan 的辛勤工作。现将其翻译成中文，方便大家学习。希望在defi领域有更多的朋友加入，一起学习，一起进步。*

*由于本人水平有限，翻译校对难免有错误，欢迎大家指正。*

本书将教你如何开发一个高级去中心化应用！具体来说，我们将构建一个[Uniswap V3](https://uniswap.org/)的克隆版，这是一个去中心化交易所。

## 为什么选择Uniswap？
- 它实现了一个非常简单的数学概念，`x * y = k`，但这使它非常强大。
- 它是一个高级应用，在简单公式之上有一层厚厚的工程。
- 它是无许可的并且经过实战检验。从一个已经在生产环境中运行多年并处理数十亿美元的应用中学习，将使你成为更好的开发者。

## 我们将构建什么

![前端应用截图](/screenshot.png)

我们将构建一个完整的Uniswap V3克隆版。它**不会是完全相同的副本**，也**不会是生产就绪的**，因为我们会以自己的方式做一些事情，而且我们**肯定**会引入多个bug。所以，不要将其部署到主网！

虽然我们的重点主要是智能合约，但我们也会顺便构建一个前端应用。🙂
我不是前端开发者，我无法做出比你更好的前端应用，但我可以向你展示如何将去中心化交易所集成到前端应用中。

我们将构建的完整代码存储在一个单独的仓库中：

https://github.com/Jeiwan/uniswapv3-code

你可以在以下地址阅读本书：

https://uniswapv3book.com/

### 有问题？

每个里程碑在[GitHub讨论区](https://github.com/Jeiwan/uniswapv3-book/discussions)都有自己的部分。
如果书中有任何不清楚的地方，不要犹豫，尽管提问！

## 目录

- 里程碑 0. 介绍
  1. 市场介绍
  2. 恒定函数做市商
  3. Uniswap V3
  4. 开发环境
  5. 我们将构建什么
- 里程碑 1. 第一次交换
  1. 介绍
  2. 计算流动性
  3. 提供流动性
  4. 第一次交换
  5. 管理合约
  6. 部署
  7. 用户界面
- 里程碑 2. 第二次交换
  1. 介绍
  2. 输出金额计算
  3. Solidity中的数学
  4. Tick位图索引
  5. 通用化铸造
  6. 通用化交换
  7. 报价合约
  8. 用户界面
- 里程碑 3. 跨Tick交换
  1. 介绍
  2. 不同价格范围
  3. 跨Tick交换
  4. 滑点保护
  5. 流动性计算
  6. 关于定点数的更多内容
  7. 闪电贷
  8. 用户界面

- 里程碑 4. 多池交换
  1. 介绍
  2. 工厂合约
  3. 交换路径
  4. 多池交换
  5. 用户界面
  6. Tick舍入
- 里程碑 5. 费用和价格预言机
  1. 介绍
  2. 交换费用
  3. 闪电贷费用
  4. 协议费用
  5. 价格预言机
  6. 用户界面
- 里程碑 6: NFT头寸
  1. 介绍
  2. ERC721概述
  3. NFT管理器
  4. NFT渲染器

## 本地运行

要在本地运行本书：
1. 安装[Rust](https://www.rust-lang.org/)。
2. 安装[mdBook](https://github.com/rust-lang/mdBook)：
    ```shell
    $ cargo install mdbook
    $ cargo install mdbook-katex
    ```
3. 克隆仓库：
    ```shell
    $ git clone https://github.com/Jeiwan/uniswapv3-book
    $ cd uniswapv3-book
    ```
4. 运行：
    ```shell
    $ mdbook serve --open
    ```
5. 访问 http://localhost:3000/（或上一个命令输出的任何URL）！
