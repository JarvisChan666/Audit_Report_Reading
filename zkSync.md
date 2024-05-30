## Gas Optimization
https://code4rena.com/reports/2024-03-zksync#g-01-function-_revertbatches-can-be-optimized
1. Case 1 : **Reuse variables ! ! !**\
a. Reduce **reading state** variable frequently ! ! !
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
```solidity
- make a cache to `s.totalBatchesExecuted` \
b. Refactor - reuse, extract variable that using many times(here is an operation)

452:         require(previousBatchNumber + 1 == _expectedNewNumber, "The provided block number is not correct");
453: 
454:         _ensureBatchConsistentWithL2Block(_newTimestamp);
455: 
456:         batchHashes[previousBatchNumber] = _prevBatchHash;
457: 
458:         // Setting new block number and timestamp
459:         BlockInfo memory newBlockInfo = BlockInfo({number: previousBatchNumber + 1, timestamp: _newTimestamp});
```
Extract `"previousBatchNumber + 1"` to `_expectedNewNumber == previousBatchNumber + 1`, don't do operation too much !