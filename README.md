---
CIP: ?
Title: Reduce frequency of leader slots to reduce network centralisation
Category: ?
Status: Proposed
Authors: Terminada
Implementors: []
Discussions: https://github.com/cardano-foundation/cips/pulls/?
Created: 2022-12-07
License: CC-BY-4.0
---

# Reduce frequency of leader slots to reduce network centralisation
I want to credit Michael Liesenfelt for coming up with the solution for this problem [here](https://github.com/cardano-foundation/CIPs/pull/379#issuecomment-1335340156)

## Motivation
Decentralisation is the core feature that underpins the value of Cardano's distributed ledger and is what makes it anti-fragile.  Indeed, Cardano's security model depends on the Ada token being widely distributed so that it is impossible for a single individual or even an entire country to corrupt or control it's ledger.  Decentralisation is necessary in terms of who owns the tokens as well as in terms of where the participants are geographically located.  These participants are not only the users who submit transactions, but also the stake pools that make the blocks which form Cardano's distributed ledger.

A key design premise in Cardano's proof-of-stake protocol is that the chance for an individual stake pool to make blocks that become part of the canonical chain is directly proportional to the fraction of total Ada it has staked.  This means not only that a stake pool's chance of making blocks is fair, but also that the chance that it's blocks become part of the canonical chain, is also fair.

When a stake pool is selected as the slot leader to make a block, it's block needs to be propagated across the internet to the other nodes.  The current Ouroboros implementation can produce consecutive slot leaders which are closer together in time than acceptable network delay constraints.  This means that it is not uncommon for the next slot leader to have not received the previous block when it becomes time to make it's own block.  When this happens, the next slot leader makes its block upon the same point in the chain as the previous block, causing a fork to occur.  In Ouroboros, such forks are resolved by nodes preferring the block that has the lowest VRF (verified random function) value.  The output of this VRF is deterministic and independent of the block's contents, so it cannot be manipulated by block producers.  The effect of this is that when a fork occurs, each pool has an equal chance of winning the "fork battle" for it's block to be adopted.  The pool that loses the contest will have it's block dropped so that it's block will never form part of the canonical chain.

The problem is that more decentralised pools suffer greater network delays and so they disproportionately experience a greater number of "fork battles" involving their blocks.  This is because the vast majority of the network is located across Europe and USA or with fast fibre links to these areas.  Only a minority of pools by stake weighting, are located in remote areas that suffer greater network delays.  This means that if a pool located near the majority gets awarded a consecutive slot with another pool, this will be unlikely to result in a fork battle because the other pool is most likely closely located.  On the other hand, if a physically remote pool is awarded a consecutive slot with another pool then this will almost always result in a fork battle because it is extremely unlikely that the other pool is close.

To be clear, this means that the more geographically decentralised pools are subjected to a disproportionately greater number of fork battles.  This represents a diversion from stake weighted participation that Ouroboros security is based upon, for the following reasons:

1. More geographically decentralised pools will get less of their blocks adopted to the chain than should be the case relative to their stake.  In other words, they get less control over access to the Cardano ledger than is proportional to their stake.
2. More geographically decentralised pools will earn a lower rate of rewards.

## Why problem noticed recently
Prior to the Vasil hard fork in September 2022, fork battles were resolved by the nodes preferring the block with the lowest "leadership VRF" value.  This produced a result that favoured smaller pools winning fork battles.  For this reason small pools did not notice the problem identified in this CIP because they were more likely to win fork battles prior to the Vasil upgrade.  However, for a small pool, every block is precious and so the operators of these pools are more likely to investigate the reasons causing their blocks to be dropped from the chain.  This is why small pools, that are geographically decentralised, are now particularly noticing this problem.

## Quantification of the problem
The current Ouroboros implementation has 1 second slots with a new block on average produced every 20 seconds.  There is a 5% chance that any particular slot will have an elected leader permitted to make a block.  For a pool awarded a slot, there is a 0.95 chance that no other leader will be awarded this same slot.  Likewise, there is a 0.95 chance that no other pool will be leader for the previous slot and 0.95 chance of no leader for the next slot.

Now consider a pool that is physically located near the centre of the majority in Europe or USA.  It's average propagation delays are going to be less than half a second.  This means that the only "fork battles" it will face will be with those pools elected slot leader for the exact same slot.  It will only very rarely face a fork battle due to transmission delay when one of the few geographically decentralised pools is elected leader consecutively with it (approximately 0% of the time).  So, this centralised pool will face slot battles for only around 5% of its blocks and so will have around 2.5% of its blocks dropped.

But, consider the more geographically decentralised pool that has average propagation delays of just 1 or 2 seconds.  This pool will face fork battles not only with leaders for the same slot, but also with leaders for the previous 1 or 2 slots, and the next 1 or 2 slots.  Therefore, a decentralised pool with 1 second delays will face fork battles over 14.26% (1 - 0.95^3) of its blocks.  Whereas an even more decentralised pool suffering 2 second delays will face fork battles over 22.62% (1 - 0.95^5) of its blocks.  Given that half fork battles will be lost, this will result in these decentralised pools losing 7.13% and 11.31% of blocks respectively.

These are quite large numbers and are a significant force driving pools to be located close to the centre of the network.  Eg: In a major data centre in Europe or USA.

This wrinkle in the current implementation goes against the core philosophy of Cardano to create a diverse, decentralised, physically dispersed, and therefore more resilient, financial operating system for the future.  Rather it creates the opposite incentive to locate all stake pools close to the centre of the network in order to minimise dropped blocks due to forks.

## Current incentives for pool operators
Also incentivise for pools to make empty blocks because these can be sent quicker across the network.  #FIXME

## Proposed solution
The current Ouroboros implementation determines the next block producer by having each stake pool do a leadership VRF calculation every slot (1 second) to see if it is a leader.  This VRF leadership function is calibrated to produce a leader on average every 20 seconds but it is possible for any particular slot to have a leader.

Instead, the protocol could be changed so that the leadership calculation is done only every 20th slot (20 seconds) and the calibration of the leadership VRF function modified so that it produces more valid leaders every time.  This will result in a slot battle each time but these are simple, deterministic, and most importantly fair, to resolve by nodes preferring the block with the lowest VRF score.

Such a change could provide the following advantages:

1. The number of fork battles faced by pools would be identical whether the pool was located in Europe, USA, or somewhere very remote.  IE: Control over and access to the protocol, as well as rewards received, is proportional to stake irrespective of where the pool is physically located.
2. Less multi-block forks since there would always be 20 seconds between blocks for all nodes to agree on the canonical chain.
3. Reduced number of leadership checks for block producer nodes.  This could allow the strict software timing requirements on the performance of the VRF leadership check to be relaxed so that "missed slot leader checks" are no longer a problem for stake pool operators.  Such "missed slot leader checks" can happen if the Haskell garbage collector is running at the time a VRF leadership calculation is required.
4. Nodes could save some bandwidth because they wouldn't need to transmit the entire block but only the header containing the VRF value.  Once they work out which header has the lowest VRF then they can download the winning block body.
5. Block timing would be predictable.  (Though this might have unknown security implications where random block production is an advantage?)
6. Blocks would be more evenly filled with transactions.  At present the filling of blocks depends on how long ago the last block occurred.  Sometimes when a block is produced only 1 second after the previous block it will contain zero transactions.
7. There would be back-up block producers since each 20 second battle would have several potential contestants so it won't matter if one fails to produce its block.
8. This might provide some protection against MEV ("Maximum Extractable Value") because individual pools would not know ahead of time if they will be the winner of the pending "slot battle" they will only know that they have a ticket to the slot battle.  Though they could predict their odds of winning based on their calculated VRF value.
9. The same high level design could be used for the Ranking Blocks (RB) in the implementation of Input Endorsers.

## Alternatives
The Ouroboros protocol has been designed around an imposed upper limit for network propagation delay of 5 seconds to guarantee the protocol's safety.  Instead of doing the leadership check every 20 slots, it could be done every 5.  This would still allow ample time for blocks to be propagated from and to the very remote decentralised nodes.  And, the leadership VRF function could still be calibrated to produce a leader on average every 20 seconds.  This would result in less slot battles than the proposed solution and less "back-up" block producers.  It would also introduce some randomness to individual block timing which might have some value in terms of security.

## Specification

## Rationale: how does this CIP achieve its goals?

## Path to Active

## Acceptance Criteria

## Implementation Plan

## Copyright
This CIP is licensed under CC-BY 4.0
