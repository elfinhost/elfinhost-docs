===================================
Elfin DEX: Crosschain Exchange with Elfin Authorizers as Oracles
===================================

If you want to exchange two tokens on the same blockchain, such as UNI and USDC, DEX is the best choice. It has plentiful liquidity and good user experience. And the most important, it is very secure for users' funds.

If you want to exchange two tokens on two different blockchains, such as ETH and BNB, there is no ideal solution yet. Existing solutions are:

1. Atomic swap. Its user experience is bad, because it does not support AMM and requires both parties are online. And it is not secure for normal users (we'll explain this later). 

2. Centralized Exchanges. Although they work well in most days, black swan events may damage all your deposit assets. Example: MtGox, FCoin, FTX, etc.

3. Bridged tokens + DEX. You send X token through a cross-chain bridge to Y token's native chain, and then use DEXes to swap Y token out. Although this method works well in most days, black swan events may damage all the bridged tokens.

Here we propose yet another solution: EA-DEX (Elfin-Authorized Decentralized EXchange). It emphasizes security while keeping good enough user experience.

How it works
---------------------

EA-DEX uses the following components to build a trading pair `P_token@X_Chain / Q_token@Y_Chain`:

1. A reception contract on X chain accepts all the requests (buy token/sell token/mint liquidity/withdraw liquidity), and emits them as cross-chain request events. If necessary, it also locks the requestors' P token for some kinds of requests. 

2. A business contract on Y chain processes the cross-chain request events in sequence, and emit the response as cross-chain response events. It also manages a pool containing Q tokens.

3. One or more relayers relay the cross-chain request/response events, which are signed by Elfin authorizers.

A cross-chain transaction has the following steps:

1. You sent a request to the reception contract, which may lock some P tokens from you.

2. A relayer relays your request event to the business contract for processing, which may sent Q tokens to your or lock Q tokens from you, before emitting the response events.

3. A relayer relays the response event to the request contract for finalizing, which may release some P tokens to you.

The trading fee collected by EA-DEX are distributed to three parties: the DEX operator, the liquid providers and the Elfin authorizers.

Becides the trading fee, you must pay relay fee as well. Some of the locked P tokens will be used to pay for relaying events. The response event specifies how much P tokens will be sent to the relayers. These relay fees compensate the gas fee paid by relayers.

Existing DEX contracts, such as Uniswap V2, ContinueCash and Gridex, can all be modified to be used for EA-DEX.

Security First
---------------------

Compared to the existing solutions, EA-DEX's major advantage is its security.

1. Atomic swap is secure if and only if the envolved parties can get correct on-chain state. Normal users usually choose to believe a blockchain explorer. However, your can be fooled by a browser with virus or an OS with polluted DNS. Elfin authorizers are experienced NaaS providers and hard to be fooled.

2. Bridged tokens and centralized exchanges have large risk exposures, which means the hackers and the CEX operators may get a large amount of illicit money by doing evil. EA-DEX carefully controls the risk exposure.

The total supply of a bridged token are all under risk. If the bridge token is used in many lending DApps and many UniswapV2-style DApps, the total supply can be very large.

EA-DEX's risk are limited to tokens deposited by the liquid proviers. To further reduce the tokens deposited by LPs, it prefers concentrated market making (such as Gridex and UniswapV3) over UniswapV2-style AMM. 

Since the authorizers can get some of the trading fee, they take more responsibilities. If an authorizer incorrectly signs an events and caused damages to the LPs, it will be slashed to compensate them.

To further isolate the risk, an Elfin authorizer use a dedicated private key for each EA-DEX it serves.

With all these measures, the illicit money that a hacker can get by attacking an EA-DEX is limited to a small value. Even after a successful attack, the users' loss can be compensate anyway.
