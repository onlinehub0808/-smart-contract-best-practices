Protocols that rely on external data as inputs (from what's known as an
[oracle](https://medium.com/better-programming/what-is-a-blockchain-oracle-f5ccab8dbd72?source=friends_link&sk=d921a38466df8a9176ed8dd767d8c77d))
automatically execute even if the data is incorrect, due to the nature of smart contracts. If a
protocol relies on an oracle that is hacked, deprecated, or has malicious intent, all processes
that depend on the oracle can now operate with disastrous effects.

For example:

1. Protocol gets price from single Uniswap pool
1. Malicious actor drains one side of the pool with a large transaction
1. Uniswap pool starts responding with a price more than 100x what it should be
1. Protocol operates as if that were the actual price, giving the manipulator a better price

We've seen examples where this will liquidate positions, allow insane arbitrage, ruin DEX
positions, and more.

### Oracle Manipulation Solutions

The easiest way to solve this is to use decentralized oracles such as:

* [Chainlink](https://chain.link/) is the leading decentralized oracle provider, and the Chainlink network can be leveraged to bring decentralized data on-chain.
* [Tellor](https://tellor.io/) is an oracle that provides censorship resistant data, secured by crypto-economic incentives, that ensure data can be provided by anyone, anytime, and checked by everyone.
* [Witnet](https://witnet.io/) leverages state-of-the-art cryptographic and economic techniques to provide your smart contracts with secure data input.

Using a median of multiple oracles provides heighten security since it is harder and more expensive to attack multiple oracles than one and it ensures that your smart contract gets the data it needs even if one oracle or API call fails. 

Another common solution is to use a time-weighted average price feed, so that price is averaged out
over `X` time periods and multiple sources. Not only does this prevent oracle manipulation, but it also reduces the
chance you can be front-run, as an order executed right before yours won't have as drastic an
impact on price. However, always keep in mind that low liquidity assets are generally easier/cheaper to manipulate, even for a period of time. One tool that gathers Uniswap price feeds every thirty minutes is
[Keep3r](https://docs.uniquote.finance/). If you're looking to build a custom solution,
[Uniswap provides a sliding window example](https://github.com/Uniswap/uniswap-v2-periphery/blob/master/contracts/examples/ExampleSlidingWindowOracle.sol).
