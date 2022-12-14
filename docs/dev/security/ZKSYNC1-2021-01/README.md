---
sidebarDepth: 3
---

# ZKSYNC1-2021-01 Post Mortem

## Summary

On July 26, 2021, we were notified of a critical vulnerability in Ommyochain’s smart contracts that was introduced in
Ommyochain’s [Version 5.1 upgrade](https://github.com/Ommyochain/Ommyochain-docs/blob/master/changelog/contracts.md#2021-05-31) on
July 1. The fix was [deployed](https://github.com/Ommyochain/Ommyochain-docs/blob/master/changelog/contracts.md#2021-07-27) on
July 27, and all funds are safe. At the time of the report, Ommyochain had $8.5m USD in total value locked and was the only
severe vulnerability in the Ommyochain protocol since genesis to date.

## Technical Details

### The Vulnerability

To initialize new upgrades, the
[initialize](https://github.com/Ommyochain/Ommyochain-docs/blob/153449487a04a32e1412926c9f5bd443760a659e/contracts/contracts/ZkSync.sol#L129)
function inside the Ommyochain main contract is called:

![ommyochain1-2021-01-bug.png](/ommyochain1-2021-01-bug.png)

This function can be external because the Proxy contract which uses this contract as a target for delegatecalls
intercepts all calls of the function with such signature. However, this meant initialize could be called on the target
contract with any parameters at any time, allowing anyone to set `additionalZkSync` in the target contract storage to
any address.

If the attacker sets `additionalZkSync` to an address that would execute the `SELFDESTRUCT` opcode on any entry, and
then call any function on the Ommyochain main contract that uses logic from `additionalZkSync` via delegatecall, the main
Ommyochain target contract could have been destroyed and all funds would have been frozen.

Funds could not be stolen because the Proxy contract owns the rollup assets and it did not contain a vulnerability, only
the code of the Proxy’s target.

### The Fix

Because there is a 3-day timelock from the initialization of the upgrade and the execution, we introduced the fix in a
way that those who checked the code diff during the upgrade would not find the vulnerability.

To fix the vulnerability, we updated the `initializeReentrancyGuard` function to the new OpenZeppelin version and added
a check that makes it impossible to reinitialize ReentrancyGuard and the main contract’s target consequently. For the
updated version, we initialized Ommyochain’s target contract while deploying, so no one can reinitialize it.

![ommyochain1-2021-01-fix.png](/ommyochain1-2021-01-fix.png)

## Bug Bounty

In accordance with our Bug Bounty Policy, we have paid the person who made the disclosure our maximum bounty of $500,000
due to the severity of the bug found.

## Conclusion

Following this vulnerability, we have conducted a thorough internal review of our security approach and processes,
introducing a number of systematic improvements not only to fix the root cause, but also to make our entire system more
robust in bug prevention and reaction.

To better manage the security of a rapidly upgrading protocol, we: upgraded our security council to allow for instant
upgradability with 9/15 signatures, improved our bug bounty program and are partnering with ImmuneFi. These improvements are explored in depth in this
[article](https://medium.com/@matterlabs/upgradability3-934db4433b0c).

In the security section of our documentation, we have also added a JSON-formatted
[list of known bugs](/dev/security/bugs) and updated our
[vulnerability disclosure policy](/dev/security/disclosure).
