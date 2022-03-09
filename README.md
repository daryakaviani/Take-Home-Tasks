# Take-Home-Tasks

StakeDaoPerpVault Solidity smart contracts

## Prerequisites

- [NodeJS](https://nodejs.org/en/)
  - v12.22.4 <=

## Installation

To install all necessary dependencies, from project root run:

```shell
npm ci
```

add a `.secret` file containing your testing mnemonic at the project root folder.

## Compiling contracts

To compile the contracts, from project root run:

```shell
npm run compile
```

## Testing contracts

To test the contracts, from project root run the following:

### Running unit tests

```shell
npm run test
```

# OPTIMIZATIONS

# Optimizations

# Loop Length Caching

## Optimization Overview

Accessing the length of an array in a for loop header means the Solidity compiler reads the array length every iteration. In a `memory` array, we expend 3 additional gas per iteration via the `mload` operation. If we cache the array length in the stack, we only call the `mload` operation once, with further iterations using the cheaper `dupN` function instead.

## Example: `totalStakedaoAsset`

### **Purpose**

Returns the total sdecrv controlled by the vault by iterating through the action contracts and accumulating their current values.

### **Relevance to Deposit & Withdraw**

`depositETH` calls `totalStakedaoAsset` twice directly and `withdrawETH` calls `_getWithdrawAmountByShares`, which calls `totalStakedaoAsset` as well. As the length of `actions` increases, so does the amount of gas expended by iterating over them.

### **Before**

```solidity
for (uint256 i = 0; i < actions.length; i++) {
	debt = debt.add(IAction(actions[i]).currentValue());
}
```

### **After**

```solidity
uint256 length = actions.length;
for (uint256 i = 0; i < length; i++) {
	debt = debt.add(IAction(actions[i]).currentValue());
}
```

I have also modified `setActions`, `closePositions`, and `rollOver` to have cached loops, but these are less impactful since they are not directly related to withdraw and deposit.

---

# Immutable State Variables

## Optimization Overview

The `immutable` keyword stores variables in code as opposed to storage. Normally, loading variables from storage uses `sload`, which is expensive. With the keyword, the compiler does not save a storage slot for the immutable value and occurrences throughout the codebase are preset at deployment time.

## Example: `IERC20 ecrv`

### **Purpose**

Represents the ecrv LP token received from depositing into the curve pool.

### **Relevance to Deposit & Withdraw**

`depositETH` calls `ecrv.safeIncreaseAllowance(sdecrvAddress, ecrvToDeposit)`, while both `depositETH` and `withdrawETH` call `ecrv.balanceOf(address(this))`. The addition of the `immutable` keyword cuts gas costs by reducing storage lookups.

### **Before**

```solidity
IERC20 ecrv;
```

### **After**

```solidity
IERC20 immutable ecrv;
```

This change was also made in `ShortOTokenActionWithSwap.sol`.

---

# Unchecked Loops

## Optimization Overview

While `i++` and `i += 1` require the solidity compiler to check for overflow. However, this is unnecessary because the length of the array we are incrementing over must necessarily be at most `2**256 - 2`. Implementing unchecked incrementation can save 30-40 gas per iteration.

## `uncheckedIncrement`

### **Purpose**

Increment an integer without checking arithmetic.

### **Relevance to Deposit & Withdraw**

`depositETH` calls `totalStakedaoAsset` twice directly and `withdrawETH` calls `_getWithdrawAmountByShares`, which calls `totalStakedaoAsset` as well. `totalStakedaoAsset` necessitates iterating over the `actions` array. `setActions`, `closePositions`, and `rollOver` have also been updated to use this modification, although less relevant to depositing and withdrawal.

### Before

```solidity
function totalStakedaoAsset() public view returns (uint256) {
    ...
    for (uint256 i = 0; i < length; i++) {
        debt = debt.add(IAction(actions[i]).currentValue());
    }
		...
}
```

### **After**

```solidity
function totalStakedaoAsset() public view returns (uint256) {
    ...
    for (uint256 i = 0; i < length; i = uncheckedIncrement(i)) {
        debt = debt.add(IAction(actions[i]).currentValue());
    }
		...
}
/**
 * @dev increment i without checking arithmetic
 */
function uncheckedIncrement(uint i) private pure returns (uint) {
    unchecked {
        return i + 1;
    }
}
```

---

# Custom Errors

## Optimization Overview

Introduced in version `0.8.4`, custom errors are more gas efficient than require and revert strings. Their primary benefit is in the gas cost of deployment (cut about 2KB from `OpynPerpVault.sol` contract size), but also reduces gas cost during runtime when the revert condition is met. 

## Custom Errors

`OpynPerpVault.sol` and `OpynPerpVault.ts` have been updated with the following custom errors instead of the original error strings:

```solidity
// 01: actions for the vault have not been initialized
error NoActionInitialized();
// O2: cannot execute transaction, vault is in emergency state
error EmergencyState();
// O3: cannot call setActions, actions have already been initialized
error ActionsInitialized();
// O4: action being set is using an invalid address
error InvalidAddress();
// O5: action being set is a duplicated action
error DuplicateAction();
// O6: deposited ETH (msg.value) must be greater than 0
error EmptyDeposit();
// O7: cannot accept ETH deposit, total sdecrv controlled by the vault would exceed vault cap
error ExceedVaultCap();
// O8: unable to withdraw ETH, sdecrv to withdraw would exceed or be equal to the current vault sdecrv balance
error ReachVaultBalance();
// O9: unable to withdraw ETH, ETH fee transfer to fee recipient (feeRecipient) failed
error FeeTransferFailed();
// O10: unable to withdraw ETH, ETH withdrawal to user (msg.sender) failed
error FailedUserWithdrawal();
// O11: cannot close vault positions, vault is not in locked state (VaultState.Locked)
error VaultUnlocked();
// O12: unable to rollover vault, length of allocation percentages (_allocationPercentages) passed is not equal to the initialized actions length
error InconsistentAllocPercent();
// O13: unable to rollover vault, vault is not in unlocked state (VaultState.Unlocked)
error VaultLocked();
// O14: unable to rollover vault, the calculated percentage sum (sumPercentage) is greater than the base (BASE)
error SumExceedsBase();
// O15: unable to rollover vault, the calculated percentage sum (sumPercentage) is not equal to the base (BASE)
error SumNotBase();
// O16: withdraw reserve percentage must be less than 50% (5000)
error WithdrawReserve();
// O17: cannot call emergencyPause, vault is already in emergency state
error InEmergency();
// O18: cannot call resumeFromPause, vault is not in emergency state
error NotInEmergency();
// O19: cannot receive ETH from any address other than the curve pool address (curvePool)
error NoncurvePool();
```

### **Relevance to Deposit & Withdraw**

Cheaper errors makes it so that upon deposit and withdrawal failure, gas costs are lower.

## Example: `withdrawETH`

### Before

```solidity
function withdrawETH(uint256 _share, uint256 minEth) external nonReentrant {
    ...
    // send fee to recipient 
    (bool success1, ) = feeRecipient.call{ value: fee }('');
    require(success1, 'O9');

    // send ETH to user
    (bool success2, ) = msg.sender.call{ value: ethOwedToUser }('');
    require(success2, 'O10');
		...
}
```

### **After**

```solidity
function withdrawETH(uint256 _share, uint256 minEth) external nonReentrant {
    ...
    // send fee to recipient
    (bool success1, ) = feeRecipient.call{value: fee}("");
    if (!success1) {
        revert FeeTransferFailed();
    }

    // send ETH to user
    (bool success2, ) = msg.sender.call{value: ethOwedToUser}("");
    if (!success2) {
        revert FailedUserWithdrawal();
    }
		...
}
```

---

# Require Checks

## Optimization Overview

When checking a `uint` in a `require` statement, switching from checking `> 0` to `!= 0` saves 6 gas because at compile-time, if statements jump to the end of a block, meaning the implicit `if (x > y) {..}` within the `require` statement gets compiled to `y x GT ISZERO _join JUMPI .. _join JUMPDEST`. In contrast, the `!= 0` approach avoids the `ISZERO` instruction entirely.

## `actionsInitialized`

### **Purpose**

Checks if the actions have been initialized.

### **Relevance to Deposit & Withdraw**

`depositETH` and `withdrawETH` (and less significantly `closePositions` and `actionsInitialized`) all call `actionsInitialized`.

### **Before**

```solidity
function actionsInitialized() private view {
	require(actions.length > 0, "O1");
}
```

### **After**

```solidity
function actionsInitialized() private view {
	require(actions.length != 0, "O1");
}
```

## `depositETH`

### **Purpose**

Deposits ETH into the contract and mint vault shares.

### **Before**

```solidity
require(msg.value > 0, "O6");
```

### **After**

```solidity
require(msg.value != 0, "O6");
```

---

# Version Upgrade

## Optimization Overview

Starting from version `0.8.0`, Safemath is automatically integrated, which can be more efficient than the library approach. In this case, the switch reduced the `OpynPerpVault.sol` contract size by ~1 KB. Starting from `0.8.2`, low-level inliner reduces runtime gas, inlining short functions that do not contain control-flow branches or opcodes with side-effects. Finally, custom errors were only introduced in `0.8.4`.
