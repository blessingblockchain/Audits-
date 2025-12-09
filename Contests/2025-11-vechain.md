# Plume Network
Plume Network || EVM-compatible-blockchain || 17 Jul 2025 to 14 August 2025 

My Finding Summay
|ID|Title|Severity|
|:-:|:---|:------:|
|[C-01](#c-01-A-delegator-who-signals-exit-and-waits-for-the-validator-to-finish-its-period-can-no-longer-withdraw-in-the-`unstake`-function-causing-permanent-loss-of-funds.)|A delegator who signals exit and waits for the validator to finish its period can no longer withdraw in the `unstake` function causing permanent loss of funds.|CRITICAL|



## [C-01] A delegator who signals exit and waits for the validator to finish its period can no longer withdraw in the `unstake` function causing permanent loss of funds.

## Description
A delegator who signals exit and waits for the validator to finish its period can no longer withdraw: when the validator eventually transitions to EXITED, calling `unstake()` underflows during the internal checkpoint update and reverts in panic `0x11`. This permanently freezes the user’s staked VET and any pending rewards, so every NFT that exits after its validator leaves the set becomes stuck.


## Vulnerability Details
Stargate records effective stake per future period using checkpoints. Entering a delegation writes an increase at `completedPeriods + 2`, and `requestDelegationExit()` already removes that weight immediately so the next period excludes the exiting token.

```solidity
        $.protocolStakerContract.signalDelegationExit(delegationId);
        (, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(
            delegation.validator
        );
        _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
```


Later, when the owner calls `unstake()` after the validator has exited, the code runs the same decrement again simply because `currentValidatorStatus == VALIDATOR_STATUS_EXITED`, even though the stake was already removed:


```solidity
        if (
            currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
            delegation.status == DelegationStatus.PENDING
        ) {
            (, , , uint32 oldCompletedPeriods) =
                $.protocolStakerContract.getValidationPeriodDetails(delegation.validator);

            _updatePeriodEffectiveStake(
                $,
                delegation.validator,
                _tokenId,
                oldCompletedPeriods + 2,
                false
            );
        }
```

`_updatePeriodEffectiveStake` subtracts the token’s effective stake from whatever the validator’s checkpoint holds.

- After the earlier removal, the value is already zero, so the subtraction underflows and triggers panic code `0x11`, reverting the entire `unstake()` call.

My poc test reproduces the failure end-to-end.

## Impact Details
1. Permanent freezing of funds

Users who exit while the validator is still active cannot ever reclaim their staked VET once that validator later exits; every `unstake()` attempt reverts due to the double decrease. This effectively locks their principal and any unclaimed rewards forever in the Stargate contract.

Because exit-and-redelegate is a normal lifecycle step, any validator churn can strand all delegators who requested exits before the validator left, leading to widespread fund loss and protocol deadlock.

## Proof of Concept
Add this to the stake.test.ts file and run the test;

```solidity
  it("should revert when unstaking after validator exit because of double effective stake decrease", async () => {
        const levelSpec = await stargateNFTMockContract.getLevel(LEVEL_ID);
        await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        const tokenId = await stargateNFTMockContract.getCurrentTokenId();

        // set an initial completed period so the delegation start period is deterministic
        await (
            await protocolStakerMockContract.helper__setValidationCompletedPeriods(
                validator.address,
                10
            )
        ).wait();

        await (await stargateContract.connect(user).delegate(tokenId, validator.address)).wait();

        // fast forward one period so the delegation becomes active before requesting exit
        await (
            await protocolStakerMockContract.helper__setValidationCompletedPeriods(
                validator.address,
                11
            )
        ).wait();

        await (await stargateContract.connect(user).requestDelegationExit(tokenId)).wait();

        // advance periods and mark validator as exited to trigger the additional decrease inside unstake()
        await (
            await protocolStakerMockContract.helper__setValidationCompletedPeriods(
                validator.address,
                13
            )
        ).wait();
        await (await protocolStakerMockContract.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_EXITED
        )).wait();

        await expect(stargateContract.connect(user).unstake(tokenId)).to.be.revertedWithPanic(
            0x11
        );
    });
```
