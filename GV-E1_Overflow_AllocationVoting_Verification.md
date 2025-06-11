# Verification of GV-E1: Overflow in `FarmWeightData.nextWeight` (`AllocationVoting.sol`)

## Vulnerability
GV-E1: Overflow in `FarmWeightData.nextWeight` in `AllocationVoting.sol`. The `nextWeight` (and `currentWeight`) field in the `FarmWeightData` struct is a `uint112`. When user votes are aggregated in the `_storeUserVotes` function, the calculated vote weight for a user (`_userWeight.mulWadDown(_votes[i].weight)`) can be a `uint256` significantly larger than `type(uint112).max`. The subsequent cast to `uint112` before adding to `data.nextWeight` truncates the value, leading to an overflow and an incorrect, much smaller weight being stored.

## Certainty of Exploitability
**Certain, given sufficient user reward weight.**

The overflow is mathematically certain if the product `_userWeight.mulWadDown(_votes[i].weight)` exceeds `type(uint112).max`. The `_userWeight` is derived from `LockingController.rewardWeightForUnwindingEpochs()`, which can produce very large `uint256` values if a user has a significant share of a bucket with a large amount of `totalReceiptTokens` and/or a high `multiplier`.

## Analysis of Vote Aggregation Logic and `uint112` Cast

1.  **`FarmWeightData` Struct:**
    ```solidity
    struct FarmWeightData {
        uint32 epoch;
        uint112 currentWeight; // Max value ~5.192e33
        uint112 nextWeight;    // Max value ~5.192e33
    }
    ```

2.  **`_storeUserVotes` Function in `AllocationVoting.sol`:**
    The critical line is:
    `data.nextWeight += uint112(_userWeight.mulWadDown(_votes[i].weight));`
    *   `_userWeight` is a `uint256` representing the user's total voting power for the specified `_unwindingEpochs`.
    *   `_votes[i].weight` is a `uint96` representing the percentage of `_userWeight` allocated to `_votes[i].farm`. It's scaled like a WAD (e.g., `1e18` for 100%).
    *   The product `_userWeight.mulWadDown(_votes[i].weight)` is a `uint256`.
    *   This product is then explicitly cast to `uint112` before being added to `data.nextWeight`. If the product is larger than `type(uint112).max`, its value is truncated, leading to overflow.

3.  **Source of `_userWeight` (from `LockingController.rewardWeightForUnwindingEpochs`):**
    `_userWeight = userShares.mulDivDown(bucketRewardWeight, totalShares);`
    where `bucketRewardWeight = data.totalReceiptTokens.mulWadDown(data.multiplier);`
    *   `data.totalReceiptTokens` (a `uint256`) can be very large, representing the total underlying tokens in a locking bucket (e.g., `10^15` full tokens with 18 decimals would be `10^15 * 1e18 = 10^33` as a `uint256`).
    *   `data.multiplier` (a `uint256`) is typically `1e18` (1x) or `2e18` (2x), but could be higher.
    *   If a user owns most or all shares in a large bucket, `_userWeight` can approximate `bucketRewardWeight`.

## Demonstration of Overflow with Example Values

*   `type(uint112).max` is `5,192,296,858,534,827,628,530,496,329,220,095` (approx `5.192 x 10^33`).

*   **Scenario for `_userWeight`:**
    *   Assume a locking bucket in `LockingController` has `data.totalReceiptTokens = 3 * 10^15 * 1e18 = 3e33` (3 quadrillion full tokens).
    *   Assume `data.multiplier = 2 * 1e18` (2x).
    *   `bucketRewardWeight = (3e33).mulWadDown(2e18) = 3e33 * 2e18 / 1e18 = 6e33`.
    *   If a single user (Alice) owns all shares in this bucket, her `_userWeight` for voting will be `6e33`.

*   **Voting:**
    *   Alice votes and allocates 100% of her weight to `farmA`. So, `_votes[0].weight = 1e18`.
    *   The value before casting is `_userWeight.mulWadDown(1e18) = (6e33).mulWadDown(1e18) = 6e33`.

*   **Overflow:**
    *   `6e33` (the calculated value) is greater than `type(uint112).max` (`~5.192e33`).
    *   `uint112(6e33)` will truncate.
        *   `6e33 = 1 * (type(uint112).max) + (6e33 - type(uint112).max)`
        *   `6e33 - 5.192296858534827628530496329220095e33 = 0.807703141465172371469503670779905e33`
        *   So, `uint112(6e33)` results in approximately `0.8077e33`.
    *   This truncated value (`~0.8077e33`) is then added to `data.nextWeight`.

## Step-by-Step Scenario to Trigger Overflow

*   **a. Initial State:**
    *   `farmA` is a registered farm. `farmWeightData[farmA].nextWeight = 0`.
    *   A locking bucket (e.g., `unwindingEpochs = N`) in `LockingController` has:
        *   `buckets[N].totalReceiptTokens = 6e33`.
        *   `buckets[N].multiplier = 1e18` (1x multiplier for simplicity here, making `bucketRewardWeight = 6e33`).
        *   `IERC20(buckets[N].shareToken).totalSupply()` equals `6e33` (1 share = 1 token value).

*   **b. User with Large Voting Power:**
    *   User Alice holds all `6e33` shares of the `shareToken` for bucket N.
    *   Her `_userWeight = LockingController.rewardWeightForUnwindingEpochs(Alice, N)` is calculated as `(6e33 * 1e18 / 1e18) * (6e33 / 6e33) = 6e33`.

*   **c. Voting Action:**
    *   Current epoch is `E_current`. `farmWeightData[farmA].epoch` is `< E_current - 1`, so `currentWeight` and `nextWeight` will be reset to 0 for `farmA` before adding new vote.
    *   Alice calls `AllocationVoting.vote(Alice, assetAddress, N, [], [{farm: farmA, weight: 1000000000000000000}])` (empty liquid votes, 100% to `farmA` in illiquid).

*   **d. Overflow Occurs in `_storeUserVotes`:**
    1.  `_userWeight` is `6e33`. `_votes[0].farm` is `farmA`, `_votes[0].weight` is `1e18`.
    2.  `FarmWeightData memory data = farmWeightData[farmA];`
        *   Since `data.epoch` is old, `data` is reset: `{epoch: E_current, currentWeight: 0, nextWeight: 0}`.
    3.  The line `data.nextWeight += uint112(_userWeight.mulWadDown(_votes[0].weight));` executes:
        *   `value_to_add_uint256 = (6e33).mulWadDown(1e18) = 6e33`.
        *   `value_to_add_uint112 = uint112(6e33) = 6e33 % (type(uint112).max + 1) = approx. 0.8077e33`.
        *   `data.nextWeight = 0 + value_to_add_uint112 = approx. 0.8077e33`.
    4.  `farmWeightData[farmA]` is updated with this new `data`.

*   **Outcome:** `farmWeightData[farmA].nextWeight` is now `~0.8077e33` instead of the correct `6e33`. The vote has been massively understated.

## Detailed Impact Assessment

1.  **Corrupted Vote Aggregation:** The primary impact is the severe misrepresentation of voting power allocated to a farm. Farms receiving votes that trigger this overflow will have their `nextWeight` (and subsequently `currentWeight`) set to a value significantly lower than the actual sum of votes intended by users.
2.  **Incorrect Governance Outcomes:**
    *   Functions like `getVote(farm)` and `getVoteWeights()` will return these understated weights.
    *   Any protocol mechanism that relies on these functions for decisions (e.g., rebalancers distributing assets, yield allocation systems) will operate on incorrect data.
    *   Farms that should receive high allocations based on user preference will receive much less, while other farms might benefit disproportionately. This undermines the entire purpose of the allocation voting system.
3.  **Loss of Voter Influence:** Users with very large `_userWeight` will find their voting power effectively capped and then wrapped around by the `uint112` limit for any single farm they vote for with a high percentage. Their true influence is not reflected.
4.  **Potential for Strategic Manipulation:** While complex, sophisticated actors could potentially analyze the thresholds and try to strategically split votes or influence total staked amounts to trigger overflows for specific farms, thereby manipulating relative weights to their advantage.

This vulnerability can lead to a significant deviation from the intended governance-driven allocation of resources, effectively breaking the voting mechanism for high-weight scenarios.

## Existing Code-Level Mitigation Analysis

*   **`uint112` Type:** The choice of `uint112` for `currentWeight` and `nextWeight` in `FarmWeightData` is the direct cause. This type is too small to hold the potential aggregated weight derived from `uint256 _userWeight` values.
*   **No Input Validation or Capping:** There are no checks on the magnitude of `_userWeight.mulWadDown(_votes[i].weight)` before the `uint112` cast to prevent or handle the overflow.
*   **No SafeMath for Accumulation (Implicit):** While `+=` is used, the overflow happens due to the cast *before* the addition, so SafeMath on the addition itself wouldn't prevent the truncation from the cast.

There are no specific mitigations in place for this overflow.

## Impact
**Critical.**
The overflow leads to silent and severe corruption of vote data, which directly impacts governance decisions on asset and yield allocation. This can cause significant financial misallocations contrary to voter intent, undermining the core functionality and fairness of the protocol's governance system.

## Recommendations

1.  **Change Data Type for Weights:**
    *   The most straightforward and robust solution is to change the data type of `currentWeight` and `nextWeight` in the `FarmWeightData` struct from `uint112` to `uint256`.
    ```solidity
    struct FarmWeightData {
        uint32 epoch;
        uint256 currentWeight; // Changed from uint112
        uint256 nextWeight;    // Changed from uint112
    }
    ```
    This would allow the sum to accommodate the full range of possible `_userWeight` values.

2.  **Input Validation/Capping (Less Ideal):**
    *   Alternatively, if changing the struct is undesirable for gas or other reasons (though `uint256` is standard), the input value could be capped *before* the cast and addition.
    ```solidity
    // Inside _storeUserVotes
    uint256 voteValue = _userWeight.mulWadDown(_votes[i].weight);
    if (voteValue > type(uint112).max) {
        voteValue = type(uint112).max; // Cap the individual vote contribution
    }
    // Ensure data.nextWeight itself doesn't overflow when adding
    if (type(uint112).max - data.nextWeight < uint112(voteValue)) { // Check before adding
        data.nextWeight = type(uint112).max; // Saturate at max
    } else {
        data.nextWeight += uint112(voteValue);
    }
    ```
    This is more complex and leads to loss of precision/intent if votes are capped, but prevents overflow. Changing to `uint256` (Recommendation 1) is generally cleaner and preferred if feasible.

Recommendation 1 (changing `FarmWeightData` members to `uint256`) is strongly advised as it directly addresses the root cause by providing sufficient storage capacity.
