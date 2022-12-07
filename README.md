---
CIP: ?
Title: Reduce frequency of leader slots to reduce network centralisation
Category: ?
Status: Proposed
Authors:
    - Michael Liesenfelt
    - Terminada
Implementors: []
Discussions:
    - https://github.com/cardano-foundation/cips/pulls/?
Created: 2022-12-07
License: CC-BY-4.0
---

# Reduce frequency of leader slots to reduce network centralisation

## Motivation
Cardano seeks to be a decentralised network that provides equal access to everyone whether they be a billionaire in Silicon Valley or a farmer in remote Western Australia.  Cardano's stake pools are responsible for producing the blocks that form the chain representing the distributed ledger.  However, just as people have different levels of internet connectivity, so do stake pools.

When a stake pool makes a block, it needs to be propagated across the internet to other Cardano nodes.  If the next block producer does not receive the previous block in time then it will make its block at the same point in the chain and cause a fork to occur.  Such forks are resolved in a deterministic manner under Ouroboros by preferring the block with the lower VRF value.  The result is that each competing pool has a 50% chance of their block being adopted.  The pool that loses this contest has their block dropped by the other nodes so that it is never adopted as part of the canonical chain.

The net effect of this is that pools that are further distributed from the major data centres of Europe and USA will have more of their blocks result in chain forks and half will get dropped.  This is a slight unfairness in the current Ouroboros implementation for two reasons:

1. More distributed pools will get less of their blocks adopted to the chain than should be the case based on their controlled stake.  This means they get less control over access to the Cardano ledger than is fair.
2. More distributed pools will earn a lower rate of rewards.

## Quantification of the problem
The odds of another pool being awarded slot leader for the same slot is around 2.5%.  The odds of another pool getting the previous slot is also around 2.5% as is the case for the next slot.

Consider the scenario where the vast majority of pools are located across Europe and USA with fast fibre links between them.  The propagation delays between these nodes is less than 0.5 seconds.  There are only a few nodes located remotely that have propagation delays of greater than 1 second.  The pools in Europe and USA will almost only ever get fork battles when another pool is awarded the exact same slot.  This amounts to 2.5% of the time and since they will lose half, it will cause around 1.3% of their blocks to be dropped from the chain.

But consider the remote pool that suffers from a 1.0 second delay to the rest of the network in Europe and USA.  This pool will get three times the number of fork battles because its blocks will also cause forks with pools awarded the previous slot, or the next slot, as well as those awarded the same slot.  This remote pool will have three times the number of its blocks dropped at around 3.9%.

A worse scenario arises for a pool that suffers 2.0 second propagation delays.  This pool will suffer fork battles if there is a leader for one of the previous 2 slots or the next 2 slots as well as for the same slot.  This will result in 5 times the number of dropped blocks at around 6.5%.

Note that such a 2.0 second delay is still well below the network security limits outlined in the Ouroboros protocol which targets an upper limit for propagation delay of less than 5.0 seconds.  A pool with a 4.0 second delay will have 9 times the number of its blocks dropped (11.7%).

## Proposed solution
The current Ouroboros implementation determines the next block producer by having each stake pool do a leadership VRF calculation every slot (1 second) to see if it is a leader.  This VRF leadership function is calibrated to produce a leader on average every 20 seconds but it is possible for any particular slot to have a leader.

Instead, the protocol could be changed so that the leadership calculation is done only every 20th slot (20 seconds) and the calibration of the leadership VRF function modified so that it produces more valid leaders every time.  This will result in a slot battle each time but these are simple, deterministic, and most importantly fair, to resolve by nodes preferring the block with the lowest VRF score.

Such a change could provide the following advantages:

1. Number of fork battles faced by pools would be identical whether the pool was located in Europe, USA or somewhere very remote.  IE: Fair access, participation and rewards for pools irrespective of where they are located.
2. Less multi-block forks since there is always 20 seconds between blocks for all nodes to agree on the canonical chain.
3. Reduced number of leadership checks for block producer nodes.  This could allow the strict timing requirements on this performance to be relaxed so that "missed slot leader checks" are no longer a problem for stake pool operators.  Such "missed slot leader checks can happen if the Haskell garbage collector is running at the time a leadership calculation is required.
4. Nodes could save some bandwidth because they wouldn't have to transmit the entire block but only the header containing the VRF value.  Once they work out which header has the lowest VRF then they can download the winning block body.
5. Block timing would be predictable.
6. Blocks would have more even filling with transactions.  At present the filling of blocks depends on how long before the last block occurred.  Sometimes when a block is produced only 1 second after a previous block it contains no transactions.
7. There would be back-up block producers since each 20 second battle would have several potential contestants so it won't matter if one fails to produce its block.
8. This might provide some protection against MEV ("Maximum Extractable Value") because individual pools would not know ahead of time if they will be the winner of a pending "slot battle" they will only know that they have a ticket to the slot battle.  Though they could predict their odds of winning based on their calculated VRF value.
9. The same high level design could be used for the Ranking Blocks (RB) in the implementation of Input Endorsers.

## Alternatives
The Ouroboros protocol has been designed based on an upper limit for network propagation delay of 5 seconds.  Instead of doing the leadership check every 20 slots, it could be done every 5.  This would still allow ample time for blocks to be propagated from and to the more remote nodes.  The leadership VRF function could be calibrated to produce a leader on average every 20 seconds.  This would result in less slot battles than the proposed solution and less "back-up" block producers.

## Specification

## Rationale: how does this CIP achieve its goals?

## Path to Active

## Acceptance Criteria

## Implementation Plan

## Copyright
This CIP is licensed under CC-BY 4.0
