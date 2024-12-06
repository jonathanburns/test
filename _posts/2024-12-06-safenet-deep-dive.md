---
title: "Safenet Deep Dive"
date: 2024-12-05T15:34:30-04:00
categories:
  - crypto
tags:
  - intents
---
![safenet](https://docs.safe.global/_next/static/media/safenet-introduction.cf3046fd.png)
Safe launched a new product [Safenet](https://safe.global/safenet) this week. 

I got a chance to meet the Safe team at Devcon chat about the design of the system. A friend of mine asked me to describe the problem they're solving, which spurred this post.
## High Level Overview

The last few years have brought us an explosion of new blockchains. This has helped with scaling, but brought about new challenges. If you have funds on one chain, but you want to interact with an application on a different chain, there's a long and arduous set of steps required to move your funds across the chains and achieve your goal.

There has been a lot effort recently to abstract this complexity for users. The general idea is that users should only need to specify an "intent". This intent describes what the user is trying to do, and abstracts the specific transactions that are needed to get there. The long-term vision of intents is that users should be able to operate across multiple chains so smoothly that it feels like it's all one system.

Early efforts to facilitate intents were high-cost, high-latency, or required centralized trust (more on that below).

Safenet enables low-cost, low-latency intents without trust assumptions. 

##  Detailed Problem Description

The easiest way to understand Safenet is by looking at a specific problem.

Imagine I have some USDC on Optimism. I would like to swap some of it for 1 ETH on Base. 

Until recently, I had two options:

**The Decentralized Path**: Move the funds through a series of trust-minimized bridges and swaps. For example:

1. Bridge the USDC to Ethereum using the Optimism native bridge.
2. Bridge the USDC to Base using the Base native bridge.
3. Swap the USDC for ETH using Uniswap.

There are several disadvantages to this method:

1. The process is tedious. You have to sign a series of transactions. Services like Li.fi are designed to help with this. You can pre-sign the transactions and they'll babysit the process for you for a small fee.
2. The process is slow. The funds must flow through a series of sequential transactions across multiple chains.
3. Costs are high due to the number of blockchain transactions (especially if the transaction needs to flow through Ethereum).
4. You must correctly predict gas costs and swap rates as you're planning the flow. In the best case, you have some left over funds on the destination chain. In the worst case, you under-estimate costs and your plan stalls mid-way through the flow, which leaves you with an asset you don't want on a chain you don't want.


**The Centralized Path**: Send the USDC to a centralized "processor" who promises to fulfill the intent. Here is an snippet from [Rhino.fi bridge docs](https://tech.rhino.fi/rhino.fi/the-details/bridges):

> As mentioned, the bridging process involves the user giving rhino.fi assets on their origin chain and rhino.fi providing their desired assets on their destination chain. In other words, the user gives us tokens and trusts that we will give them tokens on another chain.

This gives you:
1. More Speed: The centralized processor can deliver your funds immediately after they receive your send. They don't need to find a decentralized path of bridges/swaps to the destination.
2. Lower Cost: The centralized processor can leverage off-chain liquidity. This can be especially cost-effective for assets where the on-chain liquidity pools are small.

The tradeoff is that you must send your funds to a central authority, and hope they deliver what you asked.

![cost-vs-trust](https://i.imgflip.com/9cpirz.jpg)



## The Solution

Safenet aims to provide the best of both worlds. Users should be able to leverage the speed and pricing of the centralized processor, without any trust assumptions.

Let's walk through how Safenet handles the operation I described above.

I have some USDC in on Optimism. I want 1 ETH on Base. To optimize speed and cost, I would like a centralized processor (like maybe a centralized exchange) to facilitate the transaction, but without the trust assumptions typically associated with a centralized exchange.

_Safenet refers to the source chain (Optimism in our case) as the "debit chain", and the destination chain (Base in our case) as the "spend chain". I'll use those terms here._

**Prequisite**: Before using Safenet, I'll need to create a smart contract wallet on both the debit and spend chain, and install any safenet modules on those wallets. Once that is set up, I am ready to use Safenet.

**Step 1**: I go to the website of the centralized processor. The processor offers me 1 ETH on Base for 3500 USDC on Optimism (They calculate this offer based on the market price of ETH + some fees for processing).

**Step 2**: I create an off-chain "intent" (Safenet calls these transactions) by signing a message and sending it to the Safenet off-chain transaction pool. This intent denotes my expectations. I expect 1 ETH on Base, and I’m willing to trade 3500 USDC.

![step2](https://emerald-frequent-panther-621.mypinata.cloud/ipfs/bafkreiebzs3owumlgsg5zu6bmz6zjwuam7w22uzua4myghcbg4idns22u4)

**Step 3**: The processor sees the transaction in the pool and accepts the job by posting to the debit chain. This places a “resource lock” on the 3500 USDC in my smart contract wallet. For a set period of time, I won’t be able to touch these funds. If the processor delivers the ETH on Base as promised, they’ll be allowed to withdraw the funds. If they fail to fulfill my intent within the time window, I get the funds back.

![step3](https://emerald-frequent-panther-621.mypinata.cloud/ipfs/bafkreifdfgfcmht43o4jy4fd2mk3vuftpvdl66g5rn2lq23jrfzvz7hvj4)

_The processor must put up some collateral when they accept the job. If the processor does not complete the job in time, not only will I get my funds back, but I’ll get the collateral too (this ensures the processor doesn't needlessly lock up my funds)._

**Step 4**: The processor delivers the 1 ETH to my account on Base, fulfilling the intent.

![step3](https://emerald-frequent-panther-621.mypinata.cloud/ipfs/bafkreiegrhpsuzmjiexccim6oj7nqvimjcgyprmhxcodjny3sqpta77ieu)

_Worth noting: According to Safenet docs, these transfers will happen in 500ms. In a perfect world, the processor would wait for the lock transaction to finalize on the debit chain before delivering the funds on the spend chain. In that case, delivery would happen **500ms after the lock transaction has finalized**. In practice, the processor may accept the risk of re-orgs and deliver the funds prior to finality. This improves speed at a risk to the processor._

**Step 5**: The processor notifies the smart contract wallet on the debit chain (Optimism) that the fulfillment is complete.

![step5](https://emerald-frequent-panther-621.mypinata.cloud/ipfs/bafkreia275hwqm2fjiqfxpnxsijo2l2iroz6ax23bxkkbuw4ohm6ycfi4q)

_At this point, the processor has notified the debit chain that the fulfillment is complete, but has not provided any proof. Verifying the fulfillment is very easy to do off-chain, but slow and expensive to do on-chain. Instead of providing the proof on-chain, the processor makes an on-chain claim that the intent is fulfilled (without providing any proof). The processor adds some collateral with this claim, which essentially says “If I’m lyin', I’m dyin'”. If the processor is later shown to be lying, they will lose the collateral._

**Step 6**: A challenge period ensues, during which time, the claim can be disputed. If there are no challenges during the period, the processor can withdraw the locked funds from the smart contract wallet at the end of the period.

![step6](https://emerald-frequent-panther-621.mypinata.cloud/ipfs/bafkreidil5vyylwnce2ipyvbhqej3bs24mlodvw4bbsdchp7nyoliewoau)

**Claim Disputes**

As mentioned, it’s easy to verify off-chain that order was fulfilled. If the processor is lying, anyone can easily detect this and issue a challenge on-chain. By issuing the challenge, the [validator](https://docs.safe.global/safenet/core-components/validator) is accusing the processor of lying. Validators are incentivized to challenge false claims. A validator who challenges a false claim will receive the processor's collateral if the validator is shown to be correct. It's important to note that the validator must _also_ put up collateral to issue the challenge.

When a claim is challenged, it means either:

* The processor is lying (the processor claimed the intent was fulfilled when it was not)
* The validator is lying (the validator claimed the intent was not fulfilled, but it was). 

At this point, the processor must undergo the long and expensive process to prove on-chain that the funds were delivered. It's important to note that both parties involved in the challenge have to put up collateral, and whichever party is ultimately shown to be lying will lose their collateral to the party that is **not** lying. In practice, this means it doesn't pay for either party to lie. Because lying doesn't pay, there shouldn't be many challenges in practice.

## Generalizing The Problem

Now imagine I have USDC on Optimism, but instead of wanting ETH on Base, what if I wanted to stake MATIC on Polygon, or purchase an NFT on Avalanche?

There are an unlimited number of things I might want to do, and each type of intent requires specialized logic to fulfill that intent. Safe expects that specialized processors will fulfill specific types of intents. They call these **application-specific processors**. 

![application-specific-processors](https://images.ctfassets.net/1i5gc724wjeu/3m0dDipKPFrXmsrMCjLCQX/393a059576819c8c31c191c06196b4bb/safenet_blog_06.png)

We can picture how the original intent described in this post could be fulfilled by the OKX co-processor (OKX is a centralized exchange who can leverage their liquidity to fulfill this intent). 


## Additional Benefits To Processor

Aside from trust benefits to users, this architecture is helpful to processors.

The resource lock in the smart wallet works somewhat like an escrow. Without this escrow, the centralized processor must receive funds from the user, and then subsequently deliver the desired tokens on the destination chain. During the a period of time _after_ the processor receives the funds, but _before_ the funds have been sent to the destination chain, centralized processor is holding funds which belong to the user. This exposes the business to a set of regulatory concerns around "safeguarding".

Because of the escrow in the Safenet protocol, the processor never takes custody of the user's funds. They don't receive a payment until _after_ they've delivered their promise to the user. In this case, the processor never holds funds which belong to the user. This alleviates a class of regulatory concerns. 

## An Intents Marketplace

In the flow I described, the user expects a specific processor to fulfill their intent. In future versions of the protocol, this won't be required. Users will be able to create processor-agnostic intents on-chain. Processor's will bid to fulfill the intent, and the processor with the winning bid will get the job. This creates a Google-Ads-like marketplace which increases competition between processors and results in better pricing for users.

## Kicking The Tires

In this section I'll try to provide a balanced perspective of the protocol by listing potential hurdles.

### Security 

**Risks For The Processor**

Safenet aims to deliver funds extremely quickly. The most obvious risk is that the processor delivers the funds on the spend chain in 500ms, and then the debit chain re-orgs, reverting the funds lock and allowing the user to withdraw their funds. The re-org could be a natural occurrence, or a group of users who control 51% of the debit chain orchestrating an attack on the processors.

It's important to mention that the victims of such an attack would be the processors. This is not as bad as lost end-user funds IMO, since the processors are tech-savvy participants who knowingly sign up for this risk. For riskier transactions, the processor may introduce delays before fulfilling the intent.

**Risks For The User**

When a dispute happens, data must be propagated from the spend chain to the debit chain so that the dispute can be resolved and the locked funds can be released.

Depending on the chain, oracles may be required to propagate this data. A set of users controlling 51% of the oracles could generate a false proof that the assets were delivered. This would allow them to withdraw the user's funds, and take collateral from the validator.

### Price Risk & Opportunity Cost

After the processor delivers the asset on the spend chain, they are entitled to withdraw the asset on the debit chain, but not until the challenge period is complete. From the processor's perspective, there are two downsides to the wait period.

1. These funds cannot be used to fulfill more intents until the challenge period is over.
2. During the challenge period, the processor is exposed to fluctuations in price of the locked asset.

These two factors may have some impact on transaction cost (compared to skipping Safenet and using the centralized processor directly with trust)

### Competing Protocols

From this post, it may seem like smart contract wallets are critical to providing low-cost, low-latency intents without trust assumptions. As a leader in smart contract wallets, Safe appears uniquely positioned to build this platform, but are smart contract wallets _truely_ required to support this system?

Technically, no. In fact, there are already protocols using similar algorithms **without** smart contract wallets. [UniswapX](https://blog.uniswap.org/uniswapx-protocol) and [Across-Settlement](https://across.to/across-settlement) are two such protocols that exist today and have a head start on adoption. There are some differences between these protocols, but the most noticeable distinction is that Safenet locks funds in a user's smart contract wallet, whereas in the the other systems, funds are sent to an escrow smart contract where they are locked. 

**Does escrowing the funds in the user's smart contract wallet offer any advantages over the external escrow?**

This is a question I've been musing about for days. While it certainly _feels_ safer to keep the funds in your wallet, the risk is theoretically the same in both approaches. At the end of the day, there's a smart contract which escrows your funds, and your ability to get those funds back is dictated by the logic of that smart contract. 

It's arguable that coupling the protocol so tightly to the smart contract wallets is limiting. Expanding these protocols beyond the EVM is already difficult, and coupling the protocol to smart wallets introduces even more barriers there. 

Requiring smart contract wallets also creates a barrier to entry, even for users on EVM chains. Ethereum plan to move completely to smart wallets at some point in the future, but today, most users still transact using traditional wallets. Safe explains on their website that they store 100B in assets, however, the bulk of these assets are held by large institutions who use the wallets for multi-sig. EIP-7702 aims to reduce the barrier of entry for smart contract wallets, but unfortunately, EIP-7702 wallets cannot be used with Safenet and will not help with adoption.

_EIP-7702 wallets have an admin key (owned by the user) which have god-like permissions on the wallet. These permissions would allow the user to extract the locked funds._

