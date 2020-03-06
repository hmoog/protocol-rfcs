+ Feature name: `white-flag`
+ Start date: 2020-03-06
+ RFC PR: [iotaledger/protocol-rfcs#0005](https://github.com/iotaledger/protocol-rfcs/pull/5)

# Summary

Allow milestones to confirm conflicting bundles. Mutate the ledger state by deterministically ordering the bundles and
ignoring conflicts.

# Motivation

The main motivations:

- Defend against censorship attacks - Conflicts will no longer block bundles from being approved.
- Make reattachments unneccessary - As long as the network is not saturated, theoretically all bundles should be
approved. And no bundle will be left behind.
- Increase TPS - Due to easy node tipselection the network throughput should increase.
- Increase CTPS - Due to the above, increase in TPS and no left-behinds, we expect CTPS to increase as well.

# Detailed design

Let's define a conflicting bundle as a bundle that leads to a negative balance on an address if applied to the current
ledger state.

When a milestone is accepted by a node then the following algorithm to mutate the ledger state is performed:

- Look for milestone whose index is the current latest solid milestone index + 1. Milestones must be applied in order!
- When the milestone is found start traversing down the tangle in DFS going first to the trunk and only then the branch.
Stop at the transactions that has only approved parents.
- Start applying bundles to the diffmap only as you go up the recursion stack (when you climb up back from the leafs).
- Every time you attempt to apply a bundle to the diffmap:
If it is invalid mark it as seen and ignored by the milestone.
If it is valid and not conflicting with the current state then apply it to the ledger state. Mark it as seen and
approved.
If it is valid but conflicting with the current state then place it in an ordered Conflict Set.
- Local overdraft handling: After you are done traversing the subtangle attempt to apply again the conflicting bundles.
Iterate the Conflict Set in order and attempt to apply the bundles to the ledger state. Mark the applied bundles as seen
and approved. Remove applied bundles from the set. Repeat this step until the Conflict Set size doesn't change and mark
the remaining bundles of the set as seen and ignored.

Note Once a bundle is marked as ignored/seen/approved this will be final and it can't be changed by a later milestone
that comes in.

# Drawbacks

- If we ever want to supply a proof that a value transfer is valid and approved, we can't do so by merely supplying the
path from the bundle to the approving milestone as before. A proof will require to have all the transactions that are in
the past cone of the approving milestone to the genesis. However, up until now we never required to give such proofs.
If we want to do easy proofs we can create merkle trees from approved bundle hashes and add them to milestones.
- Everything that is seen is part of the tangle, including double-spend attempts. Meaning we define that possibly
malicious data will be saved as part of the consensus set of the tangle.

# Rationale and alternatives

# Unresolved questions

Since nodes will try to sort out conflicts themselves perhaps it will be wise to add more protection against forks.
In a separate RFC we can maybe define additional data that can be added to milestones to prevent that.