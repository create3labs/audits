# GM Protocol Audit Report

- Prepared by: create3labs
- Date: March 4 to 11, 2024

## Summary & Scope

The GM Protocol repository was audited at commit `32052077c578c46bd5e0e46d61a0c9a5e88e5439`

The following source files were in scope:

- `contracts/BondingCurveMarket/interfaces/ICollectionImpl.sol`
- `contracts/BondingCurveMarket/interfaces/ICreatorVaultImpl.sol`
- `contracts/BondingCurveMarket/interfaces/IRouter.sol`
- `contracts/BondingCurveMarket/interfaces/ITokenUri.sol`
- `contracts/BondingCurveMarket/interfaces/Structs.sol`
- `contracts/BondingCurveMarket/CollectionImpl.sol`
- `contracts/BondingCurveMarket/CollectionVerification.sol`
- `contracts/BondingCurveMarket/CreatorVaultImpl.sol`
- `contracts/BondingCurveMarket/DefaultTokenURI.sol`
- `contracts/BondingCurveMarket/Router.sol`

After completion of the fixes, the commit `4e87d2dd33c0e31e3a1ccbc942a89a3003b69991` was reviewed and all the issues mentioned here were solved.

## Summary of Findings

| ID     | Title                             | Severity | Fixed |
|--------|-----------------------------------|----------|-------|
| [L-01] | Missing visibility modifiers      | Low      | ✓     |
| [L-02] | Off-by-one error in length checks | Low      | ✓     |
| [L-03] | Redundant transfer when buying    | Low      | ✓     |
| [L-04] | Redundant event data              | Low      | ✓     |
| [L-05] | NPM Audit Warning                 | Low      | ✓     |
| [G-01] | Late validation check             | Gas      | ✓     |
| [G-02] | Bonding curve price calculation   | Gas      | ✓     |
| [N-01] | Structs wrapped in interface      | Note     | ✓     |
| [N-02] | Indirect imports                  | Note     | ✓     |
| [N-03] | Typos and unused code/imports     | Note     | ✓     |

## Detailed Findings


### [L-01] Missing visibility modifiers

The contract `CollectionImplementation` has two contract variables without a visibility modifier:
```solidity
uint256 currentId;
uint256[] burnedIds;
```
This will default to the `internal` modifier which might not be intended.

#### Recommendation

Add an explicit visibility modifier.

### [L-02] Off-by-one error in length checks

The `initialize` function of contract `CollectionImplementation` validates the length of given name and symbol. The upper limit checks are off by one:

```solidity
uint256 nameLength = bytes(params.collectionSpecs.name_).length;
require(
    nameLength > 0 && nameLength < MAX_NAME_LENGTH,
    "Invalid name length"
);
uint256 symbolLength = bytes(params.collectionSpecs.symbol_).length;
require(
    symbolLength > 0 && symbolLength < MAX_SYMBOL_LENGTH,
    "Invalid symbol length"
);
```

#### Recommendation

Use `nameLength <= MAX_NAME_LENGTH` and `symbolLength <= MAX_SYMBOL_LENGTH`, respectively.


### [L-03] Redundant transfer when buying

The `buy` function of contract `CollectionImplementation` contains redundant transfers of funds:

A transfer to itself is unnecessary:
```solidity
Address.sendValue(payable(address(this)), price);
```

Also, sending back dust to the sender might be redundant if there is no dust.
```solidity
// send the dust back to the sender
Address.sendValue(
    payable(msg.sender),
    msg.value - price - protocolFee - creatorFee
);
```

#### Recommendation

- Remove redundant transfer to itself.
- Consider adding a check if the amount of dust is greater than zero.


### [L-04] Redundant event data

Both the `Buy` and `Sell` events of contract `CollectionImplementation` contain fields for the `collection` address and the `creatorVault` address. Both are redundant as the collection is always the event emitter anyway and the creatorVault never changes, i.e. it can just be read from the public field and does not have to be emitted with every event.

Also, consider adding an index to the buyer/seller address event property. This will simplify finding all events of a certain buyer/seller.

#### Recommendation

- Remove `collection` and `creatorVault` fields from both the `Buy` and `Sell` events.
- Add `indexed` to the `buyer` and `seller` fields, respectively.


### [L-05] NPM Audit Warning

The NPM Audit feature results in a warning:
```
# npm audit report

undici  <=5.28.2
Undici proxy-authorization header not cleared on cross-origin redirect in fetch - https://github.com/advisories/GHSA-3787-6prv-h9w3
fix available via `npm audit fix`
node_modules/undici

1 low severity vulnerability
```

Not a relevant warning in this case but updating is advised anyway.


### [G-01] Late validation check

The `sell` function of contract `CollectionImplementation` validates the minimum sell value is reached and reverts if it is not. The check could happen earlier in the function potentially using less gas if it reverts:

```solidity
require(
    price - protocolFee - creatorFee >= minPrice_,
    "CollectionImpl: price too low"
);
```

#### Recommendation

To save gas perform the validation right after the fee calculation and before burning the NFTs in particular.


### [G-02] Bonding curve price calculation

The implementation of `getPrice` and `getIntegral` of contract `CollectionImplementation` can be optimized to save some gas.

With the parameters being the supply $x$, amount $a$, factor $f$, exponent $p$ and constant $c$ the current implementation translates to:

$f_{integral}(x) = x (\frac{fx^p}{p+1} + c)$<br>
$f_{price}(x) = f_{integral}(x + a) - f_{integral}(x)$<br>

By inlining the integral function and rearranging the operations this can be shortened:

$f_{price}(x) = f_{integral}(x + a) - f_{integral}(x)$<br>
$= (x+a) (\frac{f(x+a)^p}{p+1} + c) - x (\frac{fx^p}{p+1} + c)$<br>
$= ((x+a)^{p+1} - x^{p+1}) \frac{f}{p+1} + ac$<br>

Translated back to solidity this results in:

```solidity
function getPrice(uint256 supply, uint256 amount) public view returns (uint256) {
    uint256 ePlusOne = bondingCurveSpecs.exponent + 1;
    return ((supply + amount) ** ePlusOne - supply ** ePlusOne)
        * bondingCurveSpecs.factor / ePlusOne
        + bondingCurveSpecs.c * amount;
}
```

#### Recommendation

Evaluate if a ~1000 gas (~10%) reduction on `getPrice` is worth it to apply this change.


### [N-01] Structs wrapped in interface

The file `interfaces/Structs.sol` does not technically contain an interface, but three struct definitions that are wrapped in an interface called `Shared`. The interface definition is unnecessary and could be omitted.

#### Recommendation

Consider removing the `Shared` interface.


### [N-02] Indirect imports

The file `CollectionImpl.sol` is missing an import for `CreatorVaultImpl.sol`. The reason it is compiling anyway is that the import is indirectly contained in `Structs.sol`. That file does not directly need it, though.

#### Recommendation

- Remove `import "../CreatorVaultImpl.sol";` in `Structs.sol`
- Add `import "./CreatorVaultImpl.sol";` in `CollectionImpl.sol`


### [N-03] Typos and unused code/imports

- Typo in the struct `ICollectionImpl.InitialzeParams` => `InitializeParams`
- Unused event in `CreatorVaultImplementation` => `event Redeemed(address to, uint256 amount, uint256 share)`
- Unused import in `ITokenUri.sol` => `import "./Structs.sol";`
