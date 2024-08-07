Narrow Olive Kookaburra

Medium

# Call of ````revokeAllRole()```` would fail silently

## Summary
````RoleAccessControl.revokeAllRole()```` is wrongly implemented, the call of it would fail silently, and it would also trigger revert of ````RoleAccessControl.revokeRole()```` as a candidated way to remove role.

## Vulnerability Detail
The issue arises on L59, as the  value type of ````accountRoles```` (L20) is ````EnumerableSet````, using ````delete```` can't clear the data correctly.
```solidity
File: contracts\storage\RoleAccessControl.sol
06: library RoleAccessControl {
07:     using EnumerableSet for EnumerableSet.Bytes32Set;
...
19:     struct Props {
20:         mapping(address => EnumerableSet.Bytes32Set) accountRoles;
21:     }
22: 
...
50:     function revokeRole(address account, bytes32 role) internal {
51:         Props storage self = load();
52:         if (self.accountRoles[account].contains(role)) {
53:             self.accountRoles[account].remove(role);
54:         }
55:     }
56: 
57:     function revokeAllRole(address account) internal {
58:         Props storage self = load();
59:         delete self.accountRoles[account];
60:     }
61: }

```
The following PoC shows that: (1) ````ADMIN```` role still exists after ````revokeAllRole()```` (2) And ````revokeRole()```` can't be used as a candidate to remove role once ````revokeAllRole()```` was called
```typescript
import { expect } from 'chai'
import { Fixture, deployFixture } from '@test/deployFixture'
import { RoleAccessControlFacet, MockToken, Diamond } from 'types'
import { HardhatEthersSigner } from '@nomicfoundation/hardhat-ethers/signers'
import { ethers } from 'hardhat'
import { hexlify,  zeroPadBytes } from 'ethers'


describe('RevokeAllRoles() bug test', function () {
    let fixture: Fixture
    let deployer: HardhatEthersSigner
    let diamondAddr: string
    let roleAccessControlFacet: RoleAccessControlFacet
    const ROLE_ADMIN = hexlify(zeroPadBytes(Buffer.from('ADMIN'), 32))

    beforeEach(async () => {
        fixture = await deployFixture()
        const [signer0] = await ethers.getSigners()
        deployer = signer0
        diamondAddr = await fixture.diamond.getAddress()
        const getFacet = <T>(name: string) => ethers.getContractAt(name, diamondAddr) as Promise<T>
        roleAccessControlFacet = await getFacet<RoleAccessControlFacet>('RoleAccessControlFacet')
    })

    it('Test call of RevokeAllRoles() failed silently', async function () {
        let isAdmin = await roleAccessControlFacet.hasRole(deployer, ROLE_ADMIN)
        expect(isAdmin).to.equals(true)

        // 1. ADMIN role still exits after revokeAllRole()
        await roleAccessControlFacet.connect(deployer).revokeAllRole(deployer)
        isAdmin = await roleAccessControlFacet.hasRole(deployer, ROLE_ADMIN)
        expect(isAdmin).to.equals(true)

        // 2. And revokeRole() can't be used to remove role too
        await expect(roleAccessControlFacet.connect(deployer).revokeRole(deployer, ROLE_ADMIN)).to.be.reverted
    })

})
```

And the test log:
```solidity
2024-05-elfi-protocol\elfi-perp-contracts> npx hardhat test .\test\single-cases\BugRevokeAllRoles.test.ts     
  RevokeAllRoles() bug test
deploy MockTokens
token: WBTC 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
token: SOL 0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9
token: USDC 0x5FC8d32690cc91D4c39d9d3abcBD16989F875707
!!!!!!hardhat!!!!!!
...

    ✔ Test call of RevokeAllRoles() failed silently (49ms)


  1 passing (14s)
```

## Impact
accounts with revoked role can still operate on the system, those accounts might be leaked, compromised, owned by former employee ([real case](https://www.ledger.com/blog/security-incident-report)), or third-parties no longer cooperating with. Once it was triggered, may cause the protocol suffering huge damage. For example, a revoked account with ````ADMIN```` role can add some malicious facet to steal all funds held by the protocol. 

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/storage/RoleAccessControl.sol#L59

## Tool used

Manual Review

## Recommendation
Removing roles one by one
