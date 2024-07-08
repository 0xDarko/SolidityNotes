# Memory.sol

I’m going through a contract utilizing a utility contract `Memory.sol` , here is the information -

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity 0.8.23;

/// @title Memory utility functions.
/// @notice Modified from skozin's work for Lido. Switch over to the MCOPY
/// instruction once available - https://eips.ethereum.org/EIPS/eip-5656.
library Memory {
```

---

Working through the first function - 

```solidity
    /// @notice Allocates a memory byte array of `len` bytes without zeroing it out.
    /// @param len The length of the byte array to allocate.
    function unsafeAllocateBytes(uint256 len) internal pure returns (bytes memory result) {
        assembly {
            result := mload(0x40)
            mstore(result, len)
            let freeMemPtr := add(add(result, 32), len)
            // Align free mem ptr to 32 bytes as the compiler does now.
            mstore(0x40, and(add(freeMemPtr, 31), not(31)))
        }
    }
```

- `len` is the length of the byte array to allocate.
    - We use uint256 because it is the standard for representing integer values in Solidity, especially lengths and sizes due to its ample range.
- `result := mload(0x40)` - we are assigning result to the free memory pointer which is stored at memory address `0x40` .
    - `result` functions as a variable that will hold the allocated memory’s starting address. Effectively making `result` point to where new data can be safely written in memory.
- `mstore(result, len)` instruction stores the length of the byte array at the beginning of the allocated memory area pointed to by `result` .
    - This is critical because dynamic arrays in Solidity keep track of their size at the first 32 bytes of their allocated memory.
- `let freeMemPtr := add(add(result, 32), len)` involves calculating the new position for the free memory pointer after allocating the byte array.
    - `add(result, 32)` - `result` points to the start of the allocated memory, adding `32` moves past the 32 bytes reserved for storing the array’s length, bringing us to the start of the actual byte array data.
    - `add(add(result, 32), len)` - adds the length of the byte array `len` to the position calculated in the previous step. The result is the memory address immediately after the last byte of the allocated array, which is where the next allocation could theoretically start.
    - `let freeMemPtr := ....` - The entire expression calculates the next free memory address after the allocated array. To ensure this address is aligned to a 32 byte boundary, further calculations are done.
- `and(add(freeMemPtr, 31), not(31))` - This operation aligns the address. By adding `31` and then rounding down to the nearest multiple of `32` (using bitwise AND with the inverse of `31`), we ensure that `freeMemPtr` is on a 32-byte boundary.
    - `mstore(0x40, ...)` - the new aligned `freeMemPtr` is stored back at the memory address `0x40` , updating the free memory pointer to reflect the memory that’s now in use.

**Notes:** 

- The reason why you add 32 bytes to `result` is to skip over the space used to store the length of the byte array. In solidity the first 32 bytes of memory allocated for dynamic arrays are used to store the array’s length. So the actual data of the array starts after this 32-byte prefix.
- Understanding `and(add(freeMemPtr, 31), not(31))` , it is used to round up to the nearest 32 byte boundary. This is how
    - `add(freeMemPtr, 31)` - by adding 31 to `freeMemPtr` we’re assuring that if `freeMemPtr` was already aligned to a 32-byte boundary, we move it to the next boundary. If it wasn’t aligned, we move past the next boundary. This addition guarantees that we have “overshot” to ensure rounding up can occur.
    - `not(31)` - The `not` operations performs a bitwise NOT on `31` . Since `31` is `0001 1111` in binary, its NOT is `1110 0000` . This operation flips all bits of `31` , effectively creating a mask that can be used to zero out the lower 5 bits of any number when combined with a bitwise AND.
    - `(and...)` - The bitwise AND between `add(freeMemPtr, 31)` and `not(31)` zeros out the lower 5 bits of the result of `add(freeMemPtr, 31)` , which rounds down to the nearest multiple of 32. This is because the lower 5 bits of any number determine its modulo 32 remainder.
        - The reason why we zero out the lower 5 bits is because it determines the number’s remainder when divided by 32 (`2^5 = 32`). If a number is exactly on a 32-byte boundary, it’s lower 5 bits are all 0. If it’s 1 byte past a boundary, the last bit is 1, and so on up to 31 bytes past a boundary, where the lower 5 bits are `1111` .
        - By zeroing these bits, we effectively remove any remainder, rounding the number down to the nearest multiple of 32. However, because we first added 31, we ensure that this “rounding down” aligns us with the next boundary if we weren’t already there.
        
        ---
        
        # memcpy
        
        Now the next function `memcpy`
        
        ```solidity
            /// @notice Performs a memory copy of `len` bytes from position `src` to position `dst`.
            /// @param src The source memory position.
            /// @param dst The destination memory position.
            /// @param len The number of bytes to copy.
            function memcpy(uint256 src, uint256 dst, uint256 len) internal pure {
                assembly {
                    // While at least 32 bytes left, copy in 32-byte chunks.
                    for {} gt(len, 31) {} {
                        mstore(dst, mload(src))
                        src := add(src, 32)
                        dst := add(dst, 32)
                        len := sub(len, 32)
                    }
                    if gt(len, 0) {
                        // Read the next 32-byte chunk from `dst` and replace the first N bytes
                        // with those left in the `src`, and write the transformed chunk back.
                        let mask := sub(shl(mul(8, sub(32, len)), 1), 1) // 2 ** (8 * (32 - len)) - 1
                        let srcMasked := and(mload(src), not(mask))
                        let dstMasked := and(mload(dst), mask)
                        mstore(dst, or(dstMasked, srcMasked))
                    }
                }
            }
        ```
        
        - the `for` loop will run as long as there are atleast 32 bytes left to copy, (`gt(len, 31)`  checks if `len > 31`).
        - Inside the `for` loop, `mstore(dst, mload(src))` copies a 32-byte chunk from the source position (`src`) to the destination (`dst`). `mload(src)` reads 32 bytes from `src` , and `mstore(dst, ...)` writes those bytes to `dst` .
        - Then `src` and `dst` pointers are both incremented by 32 to move to the next chunk of data, and `len` is decreased by 32, reflecting the number of bytes remaining to be copied.
        - The `for` loop efficiently copies 32-byte chunks, taking advantage of the EVM’s word size for performance.
        - The `if` conditional checks if there is fewer than 32 bytes left to copy. Handling any remaining bytes.
        - `mask` is calculated to zero out the bytes that shouldn’t be copied over from the `dst` . The calculation of `mask` involves shifting left (`sh1`) a value by `(8 * (32 - len))` , which essentially creates a bitmask to preserve only the part of the `dst` chunk that won’t be overwritten by the remaining source bytes.
        - `srcMasked` reads the next 32-byte chunk from `src` , but masks out (zeros out) the part not part of the remaining `len` bytes.
        - `dstMasked` prepares the destination chunk by masking out the bytes that will be replaced, preserving the rest.
        - FInally `or(dstMasked, srcMasked)` combines the preserved destination bytes with the new bytes from `src` and writes them back to `dst` .
        
        **Example of Masking:**
        
        Assume we are copying 36 bytes from `src` to `dst` . After the `for` loop copies the first 32 bytes, 4 bytes remain to be copied. The above formula would be like `8 * (32 - 4) = 224` . This means the mask will shift `1` left by 224 bits, effectively creating a bitmask that zeroes out the last 4 bytes of a 32-byte word.
        
        - So now mask will be designed to preserve the first 28 bytes of the destination slot `dst` and allow the last 4 bytes to be overwrriten by the source data `src` .
        - `srcMasked` Involves masking the data loaded from `src` so that only the relevant 4 bytes are prepared for copying, leaving the rest as zeros.
        - `dstMasked` Conversely, this involves masking the data at the destination so that the 28 bytes we want to preserve are kept, and the space for the 4 bytes to be copied is zeroed out.
        - Combing them now, the `or(dstMasked, srcMasked` operation then combines the preserved bytes from `dst`  with the 4 bytes from the `src`, ensuring that the final write operation only alters the intended part of `dst` .
        
        ### **Visual Example**
        
        If we were to visualize the binary for the merging step with just 4 bits for simplicity:
        
        - Preserved **`dst`** data: **`1111 0000`** (imagine these leading **`1`**s represent the preserved data of **`dst`**, and **`0000`** is the cleared space for new data).
        - Isolated **`src`** data: **`0000 1111`** (the data to be copied).
        
        After applying **`OR`**:
        
        - Merged result: **`1111 1111`** (the preserved **`dst`** data remains intact, and the **`src`** data is copied into the prepared space).
        
        ---
        
        # copyBytes
        
        ```solidity
            /// @notice Copies `len` bytes from `src`, starting at position `srcStart`, into `dst`, starting at position `dstStart` into `dst`.
            /// @param src The source bytes array.
            /// @param dst The destination bytes array.
            /// @param srcStart The starting position in `src`.
            /// @param dstStart The starting position in `dst`.
            /// @param len The number of bytes to copy.
            function copyBytes(bytes memory src, bytes memory dst, uint256 srcStart, uint256 dstStart, uint256 len)
                internal
                pure
            {
                if (srcStart + len > src.length || dstStart + len > dst.length) revert BYTES_ARRAY_OUT_OF_BOUNDS();
                uint256 srcStartPos;
                uint256 dstStartPos;
                assembly {
                    srcStartPos := add(add(src, 32), srcStart)
                    dstStartPos := add(add(dst, 32), dstStart)
                }
                memcpy(srcStartPos, dstStartPos, len);
            }
        ```
        
        - The `if` statement makes sure that when you add `len` to either the `srcStart` or the `dstStart` you don’t attempt to read or write outside the bounds of the provided arrays, which would cause an error.
        - We add 32 to both the base memory address of `src` and `dst` since dynamic arrays store their length in the first 32 bytes.
        - Then `memcpy` is called, review to above where we discussed the inner workings.