# Changelog

## Revision after independent review (contract stage)

Dispositions of the review's findings. Economics, roles and the approved design (atomic genesis in the
factory constructor, shared 0x08cc hook, bounded re-salt, liquidity gate) are unchanged.

### MEDIUM, fixed: pre-poisoned genesis candidates blocked factory deployment (`PvPadHook`, `PvPadFactory`)

- Cause: the 16 genesis token candidates are a pure function of the future factory address, and the
  bounded salt scan reverted `UnexpectedPoolPrice` inside the constructor once all of them were
  preinitialized at a foreign price; every retry at that address failed the same way.
- Fix: `PvPadHook.realignPool(key, target)`, callable only by the pool's bound registry, moves an
  initialized, empty pool to the target price with a zero-value swap (the PoolManager skips the hook's
  own callbacks when the hook is the swapping caller). It reverts `NothingToRealign` for an
  uninitialized or already-correct pool, `PoolHasLiquidity` if any liquidity exists (never the case
  before graduation thanks to the `beforeAddLiquidity` gate, always the case after), `InvalidCallback`
  for any callback outside a realignment, and `RealignFailed` unless the swap delta is exactly zero
  and the price lands on the target. It makes no call into the registry, so the constructor can use
  it. `PvPadFactory._selectSalt` still prefers an unpoisoned candidate and emits `LaunchSaltRetried`;
  when all 16 are poisoned it returns the first candidate and `_initializeCanonicalPool` realigns it,
  re-checks the price (hard stop kept) and emits `LaunchPoolRealigned(launchId, foreignSqrtPriceX96)`.
  `predictLaunchToken` no longer reverts in that case. `createLaunch` gains the same protection.
- Proofs: the reviewer's `Proof_9d5b3504d43a` (fails before, passes after);
  `LaunchResaltTest.test_everyGenesisCandidatePoisonedStillConstructsAtCanonicalPrice`,
  `test_everyBoundedSaltPoisonedRealignsFirstCandidateAndNeverSeedsForeignPrice` (events, zero value
  moved, untouched sibling candidates, graduation afterwards),
  `test_realignmentFromExtremeForeignPricesInBothDirections`; `PoolRealignTest` (only the bound
  registry, unbound and already-correct pools refused, graduated pool refused, callback
  authentication, realigned pool graduates and charges fees normally).

### MEDIUM, disputed as service configuration: constructor needs live PoolManager code

- Reproduced: without PoolManager code at the configured address the factory constructor reverts, so
  the offline protected floor cannot construct the fourth manifest contract. The same inputs pass on
  a Sepolia fork. Deferring genesis out of the constructor would drop the atomic initialization the
  design relies on, and the review's own genesis proof asserts the genesis pool is at the canonical
  price when the constructor returns. Documented under "Deployment prerequisites" in the README: the
  floor, admission and gas estimates must run against Sepolia state.

### MEDIUM, disputed as service evidence: hook needs a mined CREATE2 salt

- Reproduced: unmined salts cannot construct `PvPadHook` (v4 derives permissions from address bits;
  the brief mandates 0x08cc). Added `script/MineHookSalt.s.sol` (`--sig "mine(address,address)"`),
  an offline evidence generator printing deployer, PoolManager, init-code hash, salt, predicted
  address and flags, plus `ProjectDeploymentTest.test_mineHookSaltScriptMatchesTheDeployedHookAndUnminedSaltsFail`.
  The hook's creation code changed in this revision, so previously mined salts are void.

### LOW, fixed: unguarded `receive()` on the factory and the curve

- `PvPadFactory.receive()` now reverts `NotBondingCurve` unless the sender is a registered curve (the
  graduation sweep still works); `BondingCurve` has no `receive()` at all. Stray ETH is refunded by
  the revert instead of being locked. `CurveInvariantTest`'s donation handler now asserts the refusal
  and the curve's balance equals reserves plus deferred fees exactly;
  `FactoryAdversarialTest.test_strayEthToFactoryOrCurveIsRejectedWhileGraduationSweepStillWorks`.

### LOW, fixed: unbounded epoch start could park the worker pot

- `WorkerSubsidy.setEpoch` also requires `windowStart - block.timestamp <= 90 days`, so an epoch can
  delay the reserved budget by at most two windows before claims open and recycling follows.
  `FeesWorkersTest.testWorkerEpochValidationDoesNotConsumePot` (far-future starts rejected) and
  `testWorkerEpochStartBoundedByMaxWindowKeepsPotRecoverable`.

### INFO, documented: any contract can bind its own pool to the shared hook

- No code change (tightening `bindPool` to one factory would change the shared-hook design). The README
  now tells integrators to identify launches only through `PvPadFactory` and never through the hook
  address, `PoolBound` events or a foreign registry's `isRegisteredPool`.

### Validation

- `forge build`, `forge test` (135 passed, 0 failed, 0 skipped, 19 suites), `forge fmt --check`,
  `python3 tools/export_abis.py --check` (hook and factory ABIs regenerated). PvPadFactory runtime is
  22,410 bytes, init code 40,343 bytes.

## Contract-stage delivery from pinned source 9007278e

- Restored the approved contracts, tests, specification and ordinary-file dependencies from
  `identity-md-launches/launch-522-workflow-contract-stage-context` at
  `9007278e14dc99dc3882e5909e0f35ee5eb07309`. Contract behavior and frozen economics are preserved.
- Added `ProjectDeployment.t.sol` to exercise actual constructor-only CREATE2 deployment through
  a distinct service factory, role assignment, protocol supply preservation, bad dependency rollback,
  and a second launch that graduates to locked full-range liquidity.
- Added deterministic ABI export tooling with a `--check` mode. Refreshed ABI arrays from the pinned
  compiler and corrected source provenance, deployment handoff, and policy-allocation documentation.
- Excluded the old manifest: generating `launch.json` and independently reviewing its resolved
  constructor roles and hook salt belong to the subsequent assignments.
- Local validation: `forge build`, `forge test` (125 passed, 0 failed, 0 skipped),
  `forge fmt --check`, and `python3 tools/export_abis.py --check` (all eight match).

The historical source-only changelog below describes work already present at the pinned commit.
Its references to a committed manifest refer to that upstream repository, not this delivery.

## Source-only follow-up (continues job 8c0d9340-589f-42c4-a7e7-a34df94a0662)

No contract is deployed, redeployed, replaced or re-minted by this revision. `launch.json` is not
edited; its notes still describe the earlier 0x00cc hook flags and must be regenerated by the manifest
contributor against this source before any deployment. SPEC.md v4 economics are unchanged (1% fee,
50/50 king/creator, 0.0005 ETH launch fee to workers, 4.2 ETH graduation, locked full-range v4 LP).

### HIGH: pre-graduation liquidity grief (`PvPadHook`, `HookMiner`, `PvPadConstants`)

- `PvPadHook` now enables `beforeAddLiquidity`. The callback is PoolManager-only and reverts
  `LiquidityClosed` unless the PoolManager's caller is the factory bound to that pool or the factory
  already reports the pool as graduated (`isRegisteredPool`). Unbound pools that name the hook never
  open. Removal callbacks, donate callbacks and `beforeInitialize` remain disabled.
- Required address flags move from 0x00cc to **0x08cc**; `PvPadHook.REQUIRED_FLAGS`,
  `PvPadConstants.HOOK_FLAGS` and `HookMiner.PVPAD_HOOK_FLAGS` export the value and
  `HookMiner.findPvPadHook(deployer, poolManager)` mines for it. The constructor still accepts only
  the PoolManager and rejects a 0x00cc address with `HookAddressNotValid`.
- The inward-moving boundary scan in `PvPadFactory._availableRange` is kept as defense in depth; it is
  no longer reachable through the PoolManager, so every launch locks the full range.
- Proof (fails on the previous source, passes now):
  `GraduationLiquidityTest.test_tickOverflowGriefOnUpperBoundaryIsRejectedBeforeGraduation`.
  Before: the dust deposit at `[887160, 887220]` with the per-tick cap succeeded and the locked upper
  tick became 887100 (`upper boundary must stay full range: 887100 != 887220`; a plain
  `vm.expectRevert` on the deposit failed with `next call did not revert as expected`). After: the
  deposit reverts `WrappedError(hook, beforeAddLiquidity, LiquidityClosed, HookCallFailed)`, costs the
  attacker nothing, and `graduate(0)` locks `[-887220, 887220]`. Further proofs: lower boundary, eight
  boundary ticks plus positions around the price, attempt while the curve is already full, liquidity
  opening after graduation, unbound pool stays closed, `HookInvariantTest` second launch.

### LOW: dust sell at the graduation threshold (`BondingCurve`)

- Behaviour: at exactly 4.2 ETH of accounted reserves both trade directions revert `NotReady` and both
  quotes return zero, so no sell can move the curve off the threshold before the permissionless
  `graduate(id)` executes. This gate was already in the accepted source; this revision re-proves it.
- Proof: a scratch counterfactual curve with the two threshold checks removed let a 5_892_857_143-unit
  (99 wei) sell succeed and un-fill the curve (`dust sell must not un-fill the curve`), after which the
  sweep reverted `NotReady`. Against the shipped curve
  `GraduationLiquidityTest.testFuzz_thresholdSellsOfAnySizeRevertAndGraduationProceeds` (256 runs,
  both overloads, 1 unit to the holder's whole balance) and
  `PvPadIntegrationTest.test_thresholdDustSellCannotInvalidatePermissionlessGraduation` pass with
  reserves, balances and allowances unchanged and graduation succeeding afterwards.

### LOW: poisoned predicted pool bricks `createLaunch` (`PvPadFactory`)

- `createLaunch` (and genesis construction) scans up to `MAX_SALT_ATTEMPTS = 16` salt derivations.
  Attempt 0 is the original salt; attempt n appends `n` to the encoded salt input. A predicted pool
  that is uninitialized or at the canonical price is used; a poisoned one is skipped and
  `LaunchSaltRetried(launchId, attempt, token)` is emitted. When every candidate is poisoned the call
  reverts `UnexpectedPoolPrice` and charges nothing. The hard price check before `initialize` is kept,
  so a launch is never seeded at a foreign price. `predictLaunchToken` exposes the selection to UIs.
- Proofs: `LaunchResaltTest` (skip to next salt, canonical preinitialization keeps attempt 0,
  successive front-running, fuzzed poison depth, all 16 poisoned fails closed, genesis re-salt inside
  the constructor, prediction matches actual creates);
  `FactoryConstructionTest.test_poisonedPredictedPoolIsSkippedAutomaticallyAndNeverSeeded` replaces the
  previous expectation that a poisoned predicted pool reverts the create.

### Tests and docs

- Fixtures use `HookMiner.findPvPadHook`; `HookSecurityTest` covers the new flags, the rejected legacy
  address and the `beforeAddLiquidity` gate; `HookInvariantTest` now expects saturation attempts to
  revert and adds a real foreign position after graduation.
- `docs/abi/PvPadHook.json` and `docs/abi/PvPadFactory.json` regenerated with `tools/export_abis.py`.
- README documents the 0x08cc flags, the liquidity gate, the bounded re-salt and the stale manifest
  note; `docs/ABI.md` and `test/README.md` updated.
- Validation: `forge build`, `forge test` (123 passed, 0 failed, 0 skipped), `forge fmt --check`.
  Slither and Mythril were not run. An independent adversarial review before release is still required.
