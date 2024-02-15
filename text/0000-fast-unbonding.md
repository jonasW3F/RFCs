# RFC-0000: Fast Unbonding

|                 |                                                                                             |
| --------------- | ------------------------------------------------------------------------------------------- |
| **Start Date**  | 17.08.2023                                                                                  |
| **Description** | This RFC proposes a safe mechanism to decrease the unbonding time of Relay Chain tokens in situations where less than 1/3 of bonded tokens are unbonding.                                                                                                             |
| **Authors**     |    Jonas Gehrlein & Alistair Stewart                                                                                                         |

## Summary

This RFC introduces a flexible unbonding mechanism for Relay Chain tokens (DOT/KSM), aiming to enhance user convenience without compromising system security. Depending on the proportion of tokens undergoing unbonding, the proposed model adjusts the unbonding duration, ranging from a minimum of 2 days when only a small amount of tokens are in the process, to the conventional 28 days if the unbonding tokens account for 1/3 or more of the total bonded tokens. In scenarios between these two thresholds, the unbonding duration scales proportionately. This ensures that users are never subjected to an unbonding period longer than the current 28 days. As an initial step to gauge its effectiveness and stability, it is recommended to implement and test this model on Kusama (adjusting proportionally to the shorter unbonding period) before considering its integration into Polkadot. By dynamically adapting to the unbonding period, this mechanism strives to make Polkadot more attractive to current and future users, while maintaining the security of the network. We also discuss a potential extension that allows users to pay a fee, if desired, to exit at the minimal possible duration.

## Motivation

Polkadot has one of the longest unbonding periods among all Proof-of-Stake protocols. While stakers are compensated with above-average staking APY, this still harms usability and deters potential stakers that want to contribute to the security of the network. Introducing a variable unbonding time that potentially allows for much faster unbonds, improves the user experience and significantly reduces friction for users in their personal risk management. In addition, this change would increase the competitiveness of native Relay Chain staking, because of enhanced user convenience. 


## Stakeholders

A brief catalogue of the primary stakeholder sets of this RFC, with some description of previous socialization of the proposal.

## Explanation

Before diving into the details of how to implement fast unbonding, it is sensible to give the reader context about why Polkadot has an unbonding period of 28 days in the first place. The reason for it is to prevent long-range attacks (LRA) that becomes theoretically possible if more than 1/3 of validators collude. This attack can only be detected after some time passes. In essence, a LRA describes the inability of users who disconnect at time t0 and reconnects later to realize that validators which were legitimate at t0 but dropped out in the meantime, are not to be trusted anymore. That means, for example, a user syncing the state via a light client could be fooled of trusting a validators that fell outside the active set after t0, and are building a competitive and malicious chain (fork). While this could potentially damage the network largely, it might only be feasible in practice if more than 2/3 of validators collude, and the turnover in the active set is accordingly high. While the risk of this attack should not be understated, we might lessen the security assumptions and allow users to unbond faster.

In short, we want to implement a mechanism that makes sure that the Relay Chain can always slash 1/2 of the stake backing some 1/3 of validators that are caught misbehaving within 28 days. **Other offenses than LRA should become obvious immediately and do not require historical slashing**.

### Mechanism

# TODO: We should include a mechanism that fills up to the maximum stake. and splits it into pieces. Let's say we have 3000 capacity left before hitting the max unstake, then if you want to unbond 5000 you should do it with 3000 and 2000 instead of 5000. The question is whether we want to solve it or just say its bad luck.

When a user decides to unbond their tokens, they don't become instantly available. Instead, they enter an "unbonding queue." The following specification illustrates how the queue works, given a user wants to unbond some portion of their stake denoted as `Unbonding Stake`. We also store a variable, `max_unstake` that tracks how much stake we want to let unbonded potentially earlier than 28 days.

At any time we store `back_of_unbonding_queue_block_number` which expresses when all the existing unbonders have unbonded. Note, that 28 days are 403200 blocks and 2 days are 28800 blocks. 

Let's assume a user wants to unbond some of their stake, `new_unbonding_stake`. Then:

```
unbonding_time_delta = new_unbonding_stake / max_unstake * 403200
```

This number needs to be added to the `back_of_unbonding_queue_block_number` under the conditions that ??. 

```
back_of_unbonding_queue_block_number = max(current_block_number, back_of_unbonding_queue_block_number) + unbonding_time_delta
```

This determines at which block the user has their tokens unbonded, making sure that it is in the limit of a minimum of 2 days and a maximum of 28 days.

```
unbonding_block_number = min(403200, max(back_of_unbonding_queue_block_number - current_block_number, 28800)) + current_block_number
```

Ultimately, the user's token are unbonded at `unbonding_block_number`.

#### Rebonding

Users that chose to unbond might want to cancel their request and rebond. There is no security loss in doing this, but with the scheme above, it could imply that a large unbond increases the unbonding time for everyone else later in the queue. When the large stake is rebonded, however, the participants later in the queue move forward and can unbond more quickly than originally estimated.

To avoid this, we should store the `unbonding_time_delta` with the unbonding account. If it rebonds when it is still unbonding, then this value should be subtracted from `back_of_unbonding_queue_block_number`. So unbonding and rebonding leaves this number unaffected. Note that we must store unbonding_time_delta, because in later eras `max_unstake` might have changed and we cannot recompute it.

----

### Extension

As an extension to a simple unbonding queue, we might add a market component that lets users unbond at the minimum possible waiting time  )(== *quickest unbond*, e.g., 2 days), by paying a fee. To achieve this, we propose to split the total unbonding capacity into two chunks, with the first capacity for the simple queue and the remaining capacity for the quickest unbond. By doing so, we allow users to choose whether they want the quickest unbond and paying a dynamic fee or join the simple queue. Setting a capacity restriction for both queues allows to guarantee a minimum unbonding time in the simple queue, while allowing users with the respective willingness to pay to get out even earlier. The fees are dynamically adjusted and are proportional to the unbonding stake (and thereby expressed in a percentage of the requested unbonding stake). In contrast to a single queue, this prevents the issue that users paying a fee jump in front of other users not paying a fee, pushing their unbonding time back (which would be terrible for UX). The revenue generated should be burned.

### Discussions
There are a few small design considerations that are open for discussion:
1) Do we also want to allow for rebonds in the possible market extension?
2) If some entity decides to unbond quickest but their stake exceeds the capacity, we'd want to have a portion unbond quickest and the rest join the queue.
3) For simplicity, the whole stake unbonds at a given block number. This means it might be more efficient to do many small unlocks.

### UI
Having two types of queues makes the decision for users more complex. UIs should provide information to a user unbonding such as "Do you like to join the queue and unbond in 10 days (latest) or pay X% of your unbonding token to exit in 2 days?

Due to the nature of the unbonding queue, dividing the total unbond into smaller unbonds would decrease the expected average unbonding time for a nominator. The trade-off will be that it requires more extrinsics and at some point it is not efficient to split further. We leave it UIs to figure out user interfaces that educates and allows nominators to decide how to split their unbonds.

---


Detail-heavy explanation of the RFC, suitable for explanation to an implementer of the changeset. This should address corner cases in detail and provide justification behind decisions, and provide rationale for how the design meets the solution requirements.


## Drawbacks


- **Lower security:** Without a doubt, the theoretical security decreases. But, as we argue, the attack is still costly enough and the attack is sufficiently theoretical. Apart from that, it would greatly improve user experience of stakers making this trade-off worthwhile.
- **Griefing attacks:** A large holder could pretend to unbond a large amount of their tokens to prevent other users to exit the network. This would, however be costly due to the fact that the holder loses out on staking rewards. The larger the attack, the higher the costs.

## Testing, Security, and Privacy

Describe the the impact of the proposal on these three high-importance areas - how implementations can be tested for adherence, effects that the proposal has on security and privacy per-se, as well as any possible implementation pitfalls which should be clearly avoided.

## Performance, Ergonomics, and Compatibility

Describe the impact of the proposal on the exposed functionality of Polkadot.

### Performance

Is this an optimization or a necessary pessimization? What steps have been taken to minimize additional overhead?

### Ergonomics

If the proposal alters exposed interfaces to developers or end-users, which types of usage patterns have been optimized for?

### Compatibility

Does this proposal break compatibility with existing interfaces, older versions of implementations? Summarize necessary migrations or upgrade strategies, if any.

## Prior Art and References
- Ethereum proposed a [similar solution](https://blog.stake.fish/ethereum-staking-all-you-need-to-know-about-the-validator-queue/)
- Alistair did some initial [write-up](https://hackmd.io/SpzFSNeXQM6YScW1iODC_A)
- There are [other solutions](https://arxiv.org/pdf/2208.05408.pdf) that further mitigate the risk of LRAs.

## Unresolved Questions

Provide specific questions to discuss and address before the RFC is voted on by the Fellowship. This should include, for example, alternatives to aspects of the proposed design where the appropriate trade-off to make is unclear.

## Future Directions and Related Material

Describe future work which could be enabled by this RFC, if it were accepted, as well as related RFCs. This is a place to brain-dump and explore possibilities, which themselves may become their own RFCs.
