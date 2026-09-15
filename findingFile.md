# Solidity Contract Audit Report

## Project: RenatalSystem
## Contract(s) Audited: `src/RentalEscrow.sol`
## Audit Date:September 15, 2026
## Severity Scale: Critical > High > Medium > Low > Informational

---
## Executive Summary
One vulnerability was identified during the audit of the RenatalEscrow contract:
- **1 High severity** - Authorisation/state-bindinf issue

The High severity issue severely break the contract's intended functionality and should be addressed before deployment.
---

## High Severity Findings

### [H-1] Renter can withdraw another listing's collateral by supplying an arbitary NFT/token ID

**Severity:** HIGH
**Impact:** HIGH
**Likelihood:** HIGH

#### Description
`RentalEscrow.endRental()` does not verify that the nftAddress and _tokenId supplied by the caller correspond to the NFT that the caller actually rented.

The contract only checks that the caller has an expired rental through `RentalEscrow::userInfo[msg.sender]`. It then uses the caller-supplied NFT and token ID to retrieve a listing and transfers that listing's collateralAmount to the caller.

As a result, an attacker with an expired rental can terminate a different user's listing and withdraw its collateral, provided the escrow holds sufficient stablecoins.

#### Impact
An attacker with an expired rental can potentially withdraw the collateral associated with arbitrary listings.

The amount that can be stolen depends on the collateral associated with the targeted listing and the stablecoin liquidity held by RentalEscrow. Repeated exploitation against different listings could drain a significant portion of the escrow's collateral.

#### Proof of Concept
The issue was reproduced by renting NFT #1 with 100 USDC collateral and subsequently calling endRental() for NFT #2 after the rental expired.

The exploit resulted in:

USDC gained: 500000000

where 500000000 represents 500 USDC.

```text
The successful trace shows:

RentalEscrow::endRental(2, Asset)
    ├─ Asset::setUser(2, address(0), 0) → success
    ├─ MockStableCoin::transfer(USER, 500000000) → success
    └─ RentalEnded(...)
```
The following test:


The attacker therefore received NFT #2's 500 USDC collateral while their actual rental was NFT #1.


#### Recommended Mitigation

Bind each renter's state to the specific NFT they rented.

For example:
```javaScript
struct UserInfo {
    address user;
    address nftAddress;
    uint256 tokenId;
    bool isRentingNft;
    uint64 expiry;
}
```

When renting, store the NFT identity:

```javaScript
userInfo[msg.sender] = UserInfo({
    user: msg.sender,
    nftAddress: nftAddress,
    tokenId: tokenId,
    isRentingNft: true,
    expiry: expiry
});
```

Then ensure endRental() can only operate on the NFT recorded for that renter, or preferably derive the NFT and token ID directly from userInfo[msg.sender] instead of accepting them as attacker-controlled parameters.