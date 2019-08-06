# Proxy transparency pattern defect

This repository showcases two main components.
- Transparency defect in the [v2.5.2 openzeppelin-sdk upgradeability contracts][v2.5.2 openzeppelin-sdk upgradeability contracts].
- Fix for the aforementioned defect.

Both are demonstrated in similar remix scenarios.
Let `some_address` be an address with a valid checksum.
The transactions presented follow this sequence:
1. A `PartiallyOpaquedContract` contract is deployed by account `A`.
2. `PartiallyOpaquedContract.changeAdmin(some_address)` with 0 wei value attached is called in a transaction by account `A`.
3. `PartiallyOpaquedContract.changeAdmin(some_address)` with 50 wei value attached is called in a transaction by account `A`.
4. An `AdminUpgradeabilityProxy` contract is deployed pointing to the `PartiallyOpaquedContract` deployed in 1. as its logic contract and account `A` as its administrator.
5. `AdminUpgradeabilityProxy.changeAdmin(some_address)` with 0 wei value attached is called in a transaction by account `B`.
6. `AdminUpgradeabilityProxy.changeAdmin(some_address)` with 50 wei value attached is called in a transaction by account `B`.

All transactions are successful except the last one in the defect PoC scenario. Normally, a transparent proxy would allow the user address `B` to access the `changeAdmin` function in the proxied contract.

This is not the case for this set of contracts because omitting the `payable` keyword emits code positioned before the `ifAdmin` modifier code that is in charge of proxying the calls transparently.

## Defect proof of concept

This proof of concept uses a flattened `AdminUpgradeabilityProxy` retrieved from the [v2.5.2 openzeppelin-sdk][v2.5.2 openzeppelin-sdk upgradeability contracts] release. A remix scenario is included to show one case where the proxy effectively "opaques" the proxied contract.

PoC for v2.5.2:
- [Flattened contract](proxy_defect_poc_v2.5.2.sol)
- [Remix scenario](Remix_Scenario_v2.5.2.json)

## Fix proof of concept

This fix was written using v2.5.2 as a base.

What it does is parametrise the `ifAdmin` modifier to distinguish cases where the administrative proxy function should accept network value or not.
A couple of constant named booleans were added to improve the clarity of the code.

This fix wasn't exhaustively tested nor reviewed. Feedback is welcome. Personally, I think a bit more expressivity in Solidity for this particular case might be better but the proposed solution looks good enough. Of course, writing transparent proxies might be a good case for the use of a [DSL](https://en.wikipedia.org/wiki/Domain-specific_language) too.

PoC fix:
- [Flattened fixed contract](proxy_defect_fix_for_v2.5.2.sol)
- [Remix scenario](Remix_Scenario_of_fix_for_v2.5.2.json)

## Consequences of this defect

A proxy implementation that is not transparent is subject to the possibility of [function name clashing](https://forum.openzeppelin.com/t/beware-of-the-proxy-learn-how-to-exploit-function-clashing/1070) with its proxied contract. This means that the proxy and proxied contracts should be checked for function name clashes before the initial deployment and afterwards before each upgrade on the proxy and upgrade candidate contract.

[v2.5.2 openzeppelin-sdk upgradeability contracts]: https://github.com/OpenZeppelin/openzeppelin-sdk/tree/v2.5.2/packages/lib/contracts/upgradeability