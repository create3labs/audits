# GM Name Service Audit Report

- Prepared by: create3labs
- Date: February 26 to March 27, 2024

## Summary & Scope

The Name Service repository was reviewed regularly during development, with the final audit being conducted at commit `a8a940be3ef55ae11d2a442b80afe27818f41db0`.

The following contracts were in scope:

- `contracts/NameService/NameService.sol`
- `contracts/NameService/NameServiceCascade.sol`
- `contracts/TokenURIProvider.sol`
- `contracts/Blast/BlastBaseUpgradeable.sol`

Issues that were identified and fixed during development are listed below.

## Summary of Findings

| ID     | Title                                  | Severity | Fixed |
|--------|----------------------------------------|----------|-------|
| [H-01] | Incomplete cleanup of username mapping | High     | ✓     |
| [H-02] | Potential DoS for gas and yield claim  | High     | ✓     |
| [H-03] | Missing setter for points operator     | High     | ✓     |
| [L-01] | Protocol fee inconsistency             | Low      | ✓     |
| [L-02] | Sell Limit off by one                  | Low      | ✓     |
| [L-03] | Misleading naming                      | Low      | ✓     |
| [L-04] | NPM Audit Warning                      | Low      | ✓     |
| [N-01] | Unused code                            | Note     | ✓     |

## Detailed Findings

### [H-01] Incomplete cleanup of username mapping

The Name Service has an optional feature which allows mapping an address to a token id and thus to a registered name:

```solidity
// used name by user tracked by the tokenId in the gm-app
mapping(address => uint256) public usernames;
```

Calling the corresponding function `setUsername(uint256 tokenId)` is optional, but when used it verifies that the `msg.sender` is also the owner of the given token before updating the mapping with `usernames[msg.sender] = tokenId;`.

The problem is that the `usernames` map is not updated when a token is sold and the owners address will remain in the map indefinitely.

#### Proof of concept

Add to the username tests in `test/NameService.ts`:

```js
it("check [H-01] Incomplete cleanup of username mapping", async function () {
  const { nameService, user1, user2 } = await loadFixture(deployFixture);
  
  const price = await nameService.getBuyPriceInclFeesForSingleToken();
  await (nameService.connect(user1) as NameService).buy("username1", {
    value: price
  });
  await (nameService.connect(user1) as NameService).setUsername(1);
  expect(await nameService.getUsername(user1.address)).to.equal(
    "username1"
  );
  await (nameService.connect(user1) as NameService).sell([1]);

  // this will fail because the usernames map will still point to the
  // previously owned token id instead of 0
  expect(
    await (nameService.connect(user1) as NameService).usernames(
      user1.address
    );
  ).to.equal(0);
});
```

#### Recommendation

Clear the username mapping for the previous owner within the `sell()` function:

```solidity
for (uint256 i = 0; i < idsLength; i++) {
  // we burn the nfts (see node_modules/@openzeppelin/contracts-upgradeable/token/ERC721/extensions/ERC721BurnableUpgradeable.sol)
  address previousOwner = _update(address(0), ids[i], msg.sender);
  
  // [H-01] reset username mapping in case the current owner pointed to this token
  if (usernames[previousOwner] == ids[i]) {
    delete usernames[previousOwner];
  }
  
  // reset the mappings between tokenId and name
  string memory name = tokenIdToName[ids[i]];
  delete nameToTokenId[name];
  delete tokenIdToName[ids[i]];
}
```

### [H-02] Potential DoS for gas and yield claim

The contract `BlastBaseUpgradeable.sol` allows assigning separate addresses to the roles of gas claimer, yield claimer and points operator. For each role there are a couple of functions which can only be called by the address assigned to that role. The problem is, that both the gas claimer and yield claimer role are able to call the function `setBlast(IBlast blast)` which will change the underlying address that is called by most of the other functions. If either the gas claimer or yield claimer is misbehaving it could
- potentially deny the other role from successfully calling its role-specific functions.
- unbeknown to the other role, redirect calls to a contract in its control by deploying a contract with matching ABI to the original Blast contract and setting the address via `setBlast`.

#### Recommendation

Hardcode the address for the underlying Blast contract, because it will never change anyway. Allowing this address to be dynamically changed is unnecessary. The Blast documentation contains an [example usage](https://docs.blast.io/building/guides/gas-fees#setting-gas-mode-to-claimable) like this:

```solidity
contract MyContract {
  IBlast public constant BLAST = IBlast(0x4300000000000000000000000000000000000002);

  constructor() {
    // This sets the Gas Mode for MyContract to claimable
    BLAST.configureClaimableGas();
  }
}
```

### [H-03] Missing setter for points operator

The contract `BlastBaseUpgradeable.sol` has setters to update the address for the gas claimer and yield claimer, but it does not have one for the points operator address.

#### Recommendation

Add a setter function for the points operator address which can only be called by the current points operator:

```solidity
function setPointsOperator(address pointsOperator_) external onlyPointsOperator {
  pointsOperator = pointsOperator_;
}
```

### [L-01] Protocol fee inconsistency

There are inconsistencies regarding the protocol fee defined in `NameService.sol`:

```solidity
// protocol fee; applied to buy and sell transactions
uint256 public fee; // with precision of 1e18; 1e18 = 1
```

- The `fee` is not an absolute value here, it defines the ratio of the actual fee proportional to the token price.
    - **Recommendation** Add an explanation as a comment, or use a better name to reduce risk of misunderstanding.
- When buying or selling Tokens a Buy or Sell event is emitted, respectively. Both contain a `protocolFee` and a `creatorFee` and in both cases the `protocolFee` is set to zero while the `creatorFee` is assigned the fee that was paid.
    - **Recommendation** Emit the events with `protocolFee` set and a `creatorFee` of zero.

### [L-02] Sell Limit off by one

The sell function of `NameService.sol` enforces a limit on the amount of tokens to sell simultaneously, but the error message and the checked condition do not match:

```solidity
require(
  idsLength < 1000,
  "NameService: can't sell more than 1000 tokens at once"
);
```

This will enforce a limit of 999 tokens, not 1000.

#### Recommendation

Replace `idsLength < 1000` with `idsLength <= 1000`.

### [L-03] Misleading naming

The name of the variable `nextTokenId` in `NameService.sol` is misleading, because it does not hold the id of the next token. Instead, it holds the id of the previous token or zero if none have been minted yet.

#### Recommendation

Rename to `prevTokenId` or `currentTokenId`.

### [L-04] NPM Audit Warning

The NPM Audit feature results in a warning:
```
# npm audit report

@openzeppelin/contracts  5.0.0-rc.0 - 5.0.1
OpenZeppelin Contracts base64 encoding may read from potentially dirty memory - https://github.com/advisories/GHSA-9vx6-7xxf-x967
fix available via `npm audit fix`
node_modules/@openzeppelin/contracts
  @openzeppelin/contracts-upgradeable  5.0.0-rc.0 - 5.0.1
  Depends on vulnerable versions of @openzeppelin/contracts
  node_modules/@openzeppelin/contracts-upgradeable
  ```

As base64 encoding is not used the Name Service is not impacted but updating is advised anyway.

#### Recommendation

Update dependencies with `npm audit fix`.

### [N-01] Unused code

There is some unused code in `NameService.sol`, namely both structs `BuyPrice` and `SellPrice` are unused.

#### Recommendation

Remove unused structs.
