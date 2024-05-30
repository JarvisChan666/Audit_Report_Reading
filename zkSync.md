## Gas Optimization
Report link: https://code4rena.com/reports/2024-03-zksync#g-01-function-_revertbatches-can-be-optimized
### Case 1 : **Reuse variables ! ! !**
#### a. Reduce **reading state** variables frequently ! ! !
```solidity
if (_newLastBatch < s.totalBatchesVerified) {
486:             s.totalBatchesVerified = _newLastBatch;
487:         }
// Don't change anything
488:         s.totalBatchesCommitted = _newLastBatch;
```
```diff
function _revertBatches(uint256 _newLastBatch) internal {
    require(s.totalBatchesCommitted > _newLastBatch, "v1"); // The last committed batch is less than new last batch
    require(_newLastBatch >= s.totalBatchesExecuted, "v2"); // Already executed batches cannot be reverted
+    s.totalBatchesCommitted = _newLastBatch;
+    if (s.l2SystemContractsUpgradeBatchNumber > _newLastBatch) {
+        delete s.l2SystemContractsUpgradeBatchNumber;
+    }
    if (_newLastBatch < s.totalBatchesVerified) {
        s.totalBatchesVerified = _newLastBatch;
    }
-    s.totalBatchesCommitted = _newLastBatch;

    // Reset the batch number of the executed system contracts upgrade transaction if the batch
    // where the system contracts upgrade was committed is among the reverted batches.
-    if (s.l2SystemContractsUpgradeBatchNumber > _newLastBatch) {
-        delete s.l2SystemContractsUpgradeBatchNumber;
-    }

-    emit BlocksRevert(s.totalBatchesCommitted, s.totalBatchesVerified, s.totalBatchesExecuted);
-    // s.totalBatchesVerified has been read 3 times ! ! !
}
```
fix:
```solidity
function _revertBatches(uint256 _newLastBatch) internal {
    require(s.totalBatchesCommitted > _newLastBatch, "v1");
    uint256 cachedtotalBatchesExecuted = s.totalBatchesExecuted;
    require(_newLastBatch >= cachedtotalBatchesExecuted, "v2");
    s.totalBatchesCommitted = _newLastBatch;
    
    if (s.l2SystemContractsUpgradeBatchNumber > _newLastBatch) {
        delete s.l2SystemContractsUpgradeBatchNumber;
    }

    if (_newLastBatch < s.totalBatchesVerified) {
        s.totalBatchesVerified = _newLastBatch;//just read once
        emit BlocksRevert(_newLastBatch, _newLastBatch, cachedtotalBatchesExecuted);
    } else {
        emit BlocksRevert(_newLastBatch, s.totalBatchesVerified, cachedtotalBatchesExecuted);
    }
}
```
- make a cache for `s.totalBatchesExecuted` \
------------------------------------------------
#### b. Refactor - reuse, extract variables that are used many times(here is an operation)
```solidity
452:         require(previousBatchNumber + 1 == _expectedNewNumber, "The provided block number is not correct");
453: 
454:         _ensureBatchConsistentWithL2Block(_newTimestamp);
455: 
456:         batchHashes[previousBatchNumber] = _prevBatchHash;
457: 
458:         // Setting new block number and timestamp
459:         BlockInfo memory newBlockInfo = BlockInfo({number: previousBatchNumber + 1, timestamp: _newTimestamp});
```

Extract `"previousBatchNumber + 1"` to `_expectedNewNumber == previousBatchNumber + 1`, don't do the operation too much !

#### Inline functions. Function 1 uses function 2, but function 1 has variables that function 2 has defined, which causes duplication and gas waste.
add and replace function use `DiamondStorage storage ds = getDiamondStorage();` and ` _saveFacetIfNew(_facet);`, but ` _saveFacetIfNew(_facet);` has already defined `ds`, so we can inline the ` _saveFacetIfNew(_facet);` in these two function for gas optimization ! ！！
```solidity
 function _addFunctions(address _facet, bytes4[] memory _selectors, bool _isFacetFreezable) private {
        DiamondStorage storage ds = getDiamondStorage();
        require(_facet.code.length > 0, "G");

        _saveFacetIfNew(_facet);
 }
 function _replaceFunctions(address _facet, bytes4[] memory _selectors, bool _isFacetFreezable) private {
        DiamondStorage storage ds = getDiamondStorage();
        require(_facet.code.length > 0, "K");
......
    }
 function _saveFacetIfNew(address _facet) private {
        DiamondStorage storage ds = getDiamondStorage();

        uint256 selectorsLength = ds.facetToSelectors[_facet].selectors.length;
        // If there are no selectors associated with facet then save facet as new one
        if (selectorsLength == 0) {
            ds.facetToSelectors[_facet].facetPosition = ds.facets.length.toUint16();
            ds.facets.push(_facet);
        }
    }


```

--- 

### Case2: Follow **CEI** Principle(Check, Effect, Interaction), do the check and handle revert first to avoid following business logic wasting gas.
```solidity
l1Bridge = _l1Bridge;
61:         l2TokenProxyBytecodeHash = _l2TokenProxyBytecodeHash;
62: 
63:         if (block.chainid != ERA_CHAIN_ID) {
64:             address l2StandardToken = address(new L2StandardERC20{salt: bytes32(0)}());
65:             l2TokenBeacon = new UpgradeableBeacon{salt: bytes32(0)}(l2StandardToken);
66:             l2TokenBeacon.transferOwnership(_aliasedOwner);
67:         } else {
68:             require(_l1LegecyBridge != address(0), "bf2");
69:             // l2StandardToken and l2TokenBeacon are already deployed on ERA, and stored in the proxy
70:         }
```
fix:
- Check first
- Use "==" instead of "!="