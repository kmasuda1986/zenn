---
title: "ERC20: TwitFiのスマコンを見てみた"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Blockchain", "SmartContract", "Ethereum", "仮想通貨"]
published: true
---
## 概要

TwitFiNFTのスマートコントラクトがどのような実装をされているか見ていきます。

### TwitFiとは ？

Tweet to Earn "TwitFi"

TwitFiは、ツイートをしてBirdNFTを育てて、TWTトークンを稼いでいく新感覚のGameFiです。

公式 <https://twitfi.com/>

TwitFi (ERC20)<https://etherscan.io/token/0xd4df22556e07148e591b4c7b4f555a17188cf5cf#code>

TwitFiNFT (ERC721) <https://etherscan.io/token/0x94cce07f299945cfe80e309c85cb0a784b3ee6c2#code>

### TwitFi

Compiler Version 0.8.17

### インポートされているライブラリ

- @openzeppelin/contracts/access/Ownable.sol
- @openzeppelin/contracts/security/Pausable.sol
- @openzeppelin/contracts/utils/math/SafeMath.sol

### 関連アドレス一覧

- TwitFi(TWT): 0xd4Df22556e07148e591B4c7b4f555a17188CF5cF
- TwitFiオーナー: 0x2340accae6c964059e0ffcf9e2bb4fca9cfde653
- Wrapped Ether(WETH): 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2

### 関数一覧

書き込みしている関数にフォーカスしてピックアップしました。

#### onlyOwner

コントラクトの所有者(※1)のみ実行ができる関数。

- pause: 送金をストップする関数
- unpause: 送金をスタートする関数
- mint: TWTをミントする関数
- addPairs: ペアを追加、削除する関数
- setLiquidityFeePercent: 流動性手数料を設定する関数
- setBurnFee: 破棄する手数料を設定する関数
- manualswap: コントラクト上で保持しているTWTを全てETHにスワップする関数
- manualBurn: TWTを破棄する関数
- openTrading: トレードを開始する関数
- withdraw: コントラクト上で保持しているETHを全てオーナーに送金する関数

※1. コントラクト作成時に使用されたアカウントがオーナーとなる。

※1. [コントラクト作成時のトランザクション](https://etherscan.io/tx/0x506ffb8e80724507fd87f3de42e7e2939655748171e25b762297944659156905)：Fromが当該コントラクトのオーナーです。

#### private

- swapAndLiquify: コントラクト上で保持している残高からETHにスワップしてLPトークンを生成する関数
- swapTokensForEth
  - 内部で呼び出している関数: [swapExactTokensForETHSupportingFeeOnTransferTokens](https://github.com/Uniswap/v2-periphery/blob/0335e8f7e1bd1e8d8329fd300aea2ef2f36dd19f/contracts/UniswapV2Router02.sol#L379)
- addLiquidity: TWTとETHのLPトークンを生成する関数

#### internal

- _transfer: TWTを送金処理する際の内部関数
- _beforeTokenTransfer: TWTをmint,burnを処理する際の内部関数

### 関数のソースコード

#### pause

指定の関数を実行できなくする関数です。

```solidity
function pause() public onlyOwner {
    // bool _paused = false となる。
    _pause();
}
```

#### unpause

指定の関数を実行再開する関数です。

```solidity
function unpause() public onlyOwner {
    // bool _paused = true となる。
    _unpause();
}
```

#### mint

TWTトークンをミントする関数です。

```solidity
/// @param _to ミントされたトークンの送り先
/// @param _amount ミントするトークン量
function mint(address _to, uint256 _amount) public onlyOwner {
    _mint(_to, _amount);
}
```

#### addPairs

ペアを追加、削除する関数です。

```solidity
/// @param toPair ペアに追加するアドレス
/// @param _enable 有効フラグ
function addPairs(address toPair, bool _enable) public onlyOwner {
    // 既に登録済みのアドレスはエラー
    require(!pairs[toPair], "This pair is already excluded");

    // ペアを追加
    pairs[toPair] = _enable;
}
```

#### setLiquidityFeePercent

流動性手数料を設定する関数です。

```solidity
/// @param liquidityFee 新しく設定する手数料
function setLiquidityFeePercent(
    uint256 liquidityFee
)
    external
    onlyOwner
{
    // 手数料を更新
    _liquidityFee = liquidityFee;
}
```

#### setBurnFee

破棄する手数料を設定する関数です。

```solidity
/// @param burnFee 新しく設定する手数料
function setBurnFee(uint256 burnFee) external onlyOwner {
    // 手数料を更新
    _burnFee = burnFee;
}
```

#### manualswap

コントラクト上で保持しているTWTを全てETHにスワップする関数です。

```solidity
function manualswap() external onlyOwner {
    // コントラクトで保持しているTWT残高を取得
    uint256 contractBalance = balanceOf(address(this));
    // 取得した残高をETHにスワップ
    swapTokensForEth(contractBalance);
}
```

#### manualBurn

TWTを破棄する関数です。

```solidity
/// @param amount 破棄するトークン量
function manualBurn(uint256 amount) public virtual onlyOwner {
    // 指定されたトークン量を破棄する
    _burn(address(this), amount);
}
```

#### openTrading

トレードを開始する関数です。

onlyOwnerではなくonlyOwner()となってる。


```solidity
function openTrading() external onlyOwner() {
    // 既に開始されている場合はエラー
    require(!tradingOpen, "Trading is already open");

    // TwitFiからUniswapを送金することをApprove
    _approve(address(this), address(uniswapV2Router), balanceOf(address(this)));

    // TwitFiとWETHのペアを作成
    uniswapV2Pair = IUniswapV2Factory(uniswapV2Router.factory()).createPair(address(this), uniswapV2Router.WETH());

    // LPトークンを生成
    uniswapV2Router.addLiquidityETH{value: address(this).balance}(address(this), balanceOf(address(this)), 0, 0, owner(), block.timestamp);

    // トレードをオープン
    tradingOpen = true;

    // ペアにuniswapV2Pair(LPトークン)を追加
    pairs[uniswapV2Pair] = true;
}
```

#### withdraw

コントラクト上で保持しているETHを全てオーナーに送金する関数です。

```solidity
function withdraw() public onlyOwner {
    // コントラクトに保持されているETHを取得
    uint amount = address(this).balance;

    // 残高をオーナーに送金
    (bool success, ) = payable(owner()).call {
        value: amount
    }("");

    // 送金チェック
    require(success, "Failed to send Ether");
}
```

### まとめ

非常にシンプルなコントラクトですが、送金 (transfer) に停止機能が実装されていたので、トレードされる方は注意が必要です。
