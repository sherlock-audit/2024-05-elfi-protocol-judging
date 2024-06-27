Magnificent Licorice Salmon

Medium

# All onlyRoleAdmin functions are useless as no admin due to empty constructor

## Summary
```RoleAccessControlFacet``` contract cannot be used for granting role as the constructor is empty and does not provide admin role to ```deployer``` or any other address.

## Vulnerability Detail
```RoleAccessControlFacet``` contract have ```onlyRoleAdmin``` modifier which restricts usage by authorised admin only.
```solidity
    modifier onlyRoleAdmin() {
        if (!RoleAccessControl.hasRole(msg.sender, RoleAccessControl.ROLE_ADMIN)) {
            revert RoleAccessControl.InvalidRoleAccess(msg.sender, RoleAccessControl.ROLE_ADMIN);
        }
        _;
    }
```
However, during the deployment no one is grant the role of admin which make all the functions using the ```onlyRoleAdmin``` modifier useless and cannot be used to grant or revoke role.
```solidity
    constructor() {}
```
## Impact
When any function using ```onlyRoleAdmin``` is called of the ```RoleAccessControlFacet``` contract it will revert as there is no admin address that can call the function.
The below foundry POC clearly shows when we try to call the ```grantRole``` function of ```RoleAccessControlFacet``` contract to an address and try to grant it ```Keeper``` role it reverts.
```solidity
    RoleAccessControlFacet public RoleAccess; // pointer to RoleAccessControlFacet contract
    address funder = makeAddr("funder"); // random user creation

    function setUp() public {
        RoleAccess = new RoleAccessControlFacet();
    }


    // forge t --mt test_grantRole -vvv
    function test_grantRole() external {
    
        // expecting revert as caller does not have admin role
        vm.expectRevert(abi.encodeWithSelector(RoleAccessControl.InvalidRoleAccess.selector, address(this), ROLE_ADMIN));
        RoleAccess.grantRole(funder, ROLE_KEEPER); // calling function grant 
    }

```
The result pass clearly indicating revert when ```grantRole``` is called it reverts as expected as the ```deployer(test contract)``` does not have ```admin``` rights to call the function.
```shell
Ran 1 test for test/Contract.t.sol:Contract
[PASS] test_grantRole() (gas: 13815)
Traces:
  [323816] Contract::setUp()
    ├─ [286124] → new RoleAccessControlFacet@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   └─ ← [Return] 1429 bytes of code
    └─ ← [Stop] 

  [13815] Contract::test_grantRole()
    ├─ [0] VM::expectRevert(InvalidRoleAccess(0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496, 0x41444d494e000000000000000000000000000000000000000000000000000000))
    │   └─ ← [Return] 
    ├─ [3064] RoleAccessControlFacet::grantRole(funder: [0xce9Cffa143A44f5C9100fB9e9283baBE761926DC], 0x4b45455045520000000000000000000000000000000000000000000000000000)
    │   └─ ← [Revert] InvalidRoleAccess(0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496, 0x41444d494e000000000000000000000000000000000000000000000000000000)
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.41ms (195.80µs CPU time)

Ran 1 test suite in 1.66s (1.41ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/facets/RoleAccessControlFacet.sol#L15

## Tool used

Foundry

## Recommendation
The recommendation is made to grant admin role to deployer or any other user by calling ```grantRole``` function in the constructor of the ```RoleAccessControlFacet``` contract.
```diff
   constructor() {
+       RoleAccessControl.grantRole(msg.sender, RoleAccessControl.ROLE_ADMIN);
    }
``` 
