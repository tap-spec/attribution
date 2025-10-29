# Performance Optimizations

This document describes the performance optimizations implemented in the attribution codebase.

## Overview

Several performance improvements have been made to reduce computational complexity and improve efficiency for common operations.

## Optimizations

### 1. Privacy Budget Store - Map-based Lookups

**Problem**: The privacy budget store was implemented as an array with linear search (`Array.find()`), resulting in O(n) lookups for each privacy budget operation.

**Solution**: Replaced the array-based storage with a `Map<string, PrivacyBudgetStoreEntry>` using composite keys in the format `"${site}:${epoch}"`.

**Impact**: 
- Lookup complexity: O(n) → O(1)
- Insertion complexity: O(n) → O(1)
- Particularly beneficial when dealing with many epochs or sites

**Files modified**:
- `src/backend.ts`: Changed `#privacyBudgetStore` from array to Map
- `src/backend.ts`: Updated `#deductPrivacyBudget()` method
- `src/backend.ts`: Updated `#zeroBudgetForSites()` method
- `src/backend.ts`: Updated `clearState()` method

### 2. Precise Sum Calculation - Kahan Summation

**Problem**: Using `reduce()` for summing arrays can suffer from floating-point precision errors, and the code had TODO comments indicating this needed improvement.

**Solution**: Implemented the Kahan summation algorithm, which provides better numerical accuracy when summing floating-point numbers.

**Impact**:
- Improved numerical precision for credit allocation
- More accurate histogram l1 norm calculations
- Slightly better performance for large arrays compared to naive reduction

**Files modified**:
- `src/backend.ts`: Added `preciseSum()` function
- `src/backend.ts`: Updated `fairlyAllocateCredit()` to use `preciseSum()`
- `src/backend.ts`: Updated histogram l1 norm calculation
- `src/allocate.test.ts`: Updated test code to use `preciseSum()`

### 3. Optimized Impression Matching Logic

**Problem**: The `#commonMatchingLogic()` method was computing values like `conversionCaller` and `impressionCaller` unconditionally, even when the corresponding filters were empty.

**Solution**: 
- Moved `conversionCaller` computation outside the loop (constant value)
- Added filter existence checks to avoid unnecessary computations
- Only compute `impressionCaller` when the filter is non-empty

**Impact**:
- Reduced unnecessary string operations in the hot path
- Better branch prediction for common cases with empty filters
- More efficient filtering logic

**Files modified**:
- `src/backend.ts`: Optimized `#commonMatchingLogic()` method

## Performance Characteristics

### Before Optimizations

- Privacy budget lookup: O(n) per operation
- Sum calculations: Naive reduction with potential precision loss
- Impression matching: Unconditional value computation

### After Optimizations

- Privacy budget lookup: O(1) per operation
- Sum calculations: Kahan algorithm with better precision
- Impression matching: Lazy evaluation of conditional values

## Testing

All existing tests pass with these optimizations. The changes maintain backward compatibility while improving performance characteristics.

## Future Optimization Opportunities

1. **Impression Indexing**: Consider indexing impressions by site or epoch for faster lookups in multi-epoch scenarios
2. **Lazy Epoch Calculation**: Cache epoch calculations when the same site/timestamp pairs are checked multiple times
3. **Batch Operations**: Implement batch processing for multiple conversions to amortize fixed costs
4. **Memory Pooling**: Reuse allocated histogram arrays instead of creating new ones each time
