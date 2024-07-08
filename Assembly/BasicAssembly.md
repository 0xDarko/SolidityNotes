# Assembly

### Accessing Mapping Values

**Example:**

```solidity
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, fundingAddress)
            mstore(add(ptr, 0x20), s_addressToAmountFunded.slot)
            let hash := keccak256(ptr, 0x40)
            amountFunded := sload(hash)    
        }
```

1. We load the free memory pointer into `ptr` so we can access it.
2. We then store `fundingAddress` right away at the start of the free memory pointer. You’ll see why we do this, the ordering of keccak256 hashing is: key → slot
3. We then add `0x20(32 bytes)` to `ptr` which will make `s_addressToAmountFunded.slot` be stored at `0x60` , the next slot in memory. We currently are: 0x40 → `fundingAddress` , 0x60 → `s_addressToAmountFunded` storage slot which holds the length.
4. We then hash the start of the data (`fundingAddress` ) stored at `ptr` and the length of the data to hash which is `0x40(64 bytes)` . Why it’s 64 bytes is because, we are starting at the 32 bytes of ptr, and the other 32 bytes of `s_addressToAmountFunded.slot`
5. The 64-byte block includes the padded `fundingAddress` (32 bytes) and the mapping’s slot number (32 bytes)

**Notes:** The storage location of a value in a mapping is determined by hashing the key together with the storage slot of the mapping. This prevents collisions between different mappings and allows keys to be dynamically hashed into unique storage locations.

---

### Accessing Dynamic Arrays

**Example:**

```solidity
    function getFunders(uint256 index) external view returns(address) {
        //return s_funders[index];
        assembly {
            mstore(0x00, s_funders.slot)
            let base := keccak256(0x00, 0x20)
            let value := add(base, index)
            let funder := sload(value)
            mstore(0x40, funder)
            return(0x40, 0x20)
            }
    }
```

1. We first store the slot of `s_funders` to access the dynamic array in memory slot `0x00`
2. We know that like mappings, dynamic arrays first 32 bytes store its length and after that it’s value’s reside in `keccak256` hashing following indices. So what we are doing is hashing the base `s_funders` which we stored at `0x00` and `0x20(32bytes)` of data to be hashed. Which is just the slot number.
3. Next with how `keccak256` works [p(position …n] you add the index to the base “position” which will bring you to the element in the dynamic array.
4. Now we are able to load that value obtained from that positioning via `sload` called upon it storing it in `funder` at `0x40` in memory.
5. Finally we call return on `0x40` position in memory returning `0x20(32bytes)` of data.

### External View Call

**Example:**

```solidity
    function getVersion() public view returns (uint256){
        //return s_priceFeed.version();
        assembly {
            let freeMemPtr := mload(0x40)
            mstore(freeMemPtr, VERSION_SELECTOR)
            let success := staticcall(
                gas(),
                sload(s_priceFeed.slot),
                freeMemPtr,
                0x04,
                freeMemPtr,
                0x20
            )

            if iszero(success) {
                revert(0,0)
            }
            return(freeMemPtr, 0x20)
        }

    }
```

1. First we declare a variable to access the free memory pointer at `0x40` , `freeMemPtr`
2. We then store the function selector for `version()` in memory at the free memory pointer to access that value (4 bytes)
3. We use `staticcall` here because it’s an external call that does not change the state of the blockchain. If it was a call that wasn’t a `view` and modified the blockchain state, you would use `call`
4. Just like how we call, `call` we use `staticcall` declaring the arguments and positioning and return value positioning.
5. if `success` value is zero, revert because if it was a successful call, it would be considered true(1).
6. If it is indeed a successful call, we return the 32 bytes(0x20) of data in the free memory pointer.

---

### argsOffset, argsSize, retOffset, retSize

- **argsOffset:** The memory position where the data (arguments) for the function call begins. It points to the start of the data in memory that will be used as the input for the function call.
- **argsSize:** The size, in bytes, of the data(arguments) you’re passing to the function. This tells the EVM how many bytes of data to read from memory starting at the `argsOffset` .
- **retOffset:** The memory position where the called function will begin to store its return data.
- **retSize:** The number of bytes in memory that the calling function allocates for the returning data. This tells the EVM how much space is reserved for the return data.

---

### Accessing Variables in Packed Slots

**Example:**

```solidity
        // slot 0
    uint128 public s_a;  // 16 bytes
    uint64 public s_b;   // 8 bytes
    uint32 public s_c;   // 4 bytes
    uint32 public s_d;   // 4 bytes
    
    // We want to access s_a which resides on the right side(msb)
    function test_sstore() public {
        assembly {
            let v := sload(0)
            // 111 ... 111 | 000 ... 000
            //             |     128 bits
            let mask_a := not(sub(shl(128, 1),1))
            v := and(v, mask_a)
            v := or(v, 11)

            sstore(0, v)
```

1. First we load slot 0 in a variable called `v`
2. Then we create a mask to zero out `s_a`. It works this way:
`shl(128,1)` shifts 1 left by 128 bits, placing a 1 at the 129th bit position.
`sub(shl(128,1),1)` subtracts 1 which flips all bits to the left of that position (including the 128th bit) to 1 and all bits to the right to 0. Effectively creating a bitmask where the first 128 bits are 0 and the rest are 1.
`not(sub(shl(128,1,1))` inverts this, resulting in a mask where the first 128 bits (msb) are 0, to clear out `s_a` and all others are 1 (to keep `s_b`, `s_c`, and `s_d` )
3. Using `and(v, mask_a)` applies the mask clearing out `s_a` it does this because with the bitwise `and` operator, it sets each bit to 1 if both operands are 1 at that bit position, otherwise to 0. Effectively zeroing out `s_a` because the mask has 0s in the positions corresponding to `s_a`
4. The `or` bitwise sets each bit to 1 if either of the operands has as 1 in that position, otherwise to 0.

---

### Calldata

When grabbing a `bytes calldata` and decoding like this - 

```solidity
    function _decodeTransformERC20Data(bytes calldata _data)
        public
        pure
        returns (address inputToken, address outputToken, uint256 inputTokenAmount, bytes4 selector)
    {
        assembly {
                let p := _data.offset
                selector := calldataload(p)
                inputToken := calldataload(add(p, 32)) // adjusted from 4 to 32
                outputToken := calldataload(add(p, 64)) // adjusted from 36 to 64
                inputTokenAmount := calldataload(add(p, 96)) // adjusted from 68 to 96
        }
    }
```

Whats happening is the following:

1. We assign a variable `p` to point to the **START** of the calldata **DATA** ignoring the length.
2. Since `calldata` is a dynamically-sized byte array, it’s in chunks of 32 bytes. The function selector will be in the first 32 bytes, so we can simply call `calldataload(p)` which is the start of the data, which would be the function selector padded.
3. Once we have the function selector, it’s pretty easy to grab the rest of the values, we simply do `(add(p, 32))` which is adding the 32 byte size of `p` to `32 bytes` so we start at `64 bytes` the next slot of data. You just follow along incrementing by slot size to grab the rest of the data slots in the calldata.

---

### Fixed Sized Arrays

```solidity
// SPDX-License-Identifier: MIT

pragma solidity 0.8.20;

contract AssemblyIf {
    // slot of element = slot where array is declared + index of array element

    // slot 0, slot 1, slot 2
    uint256[3] private arr_0 = [1, 2, 3];
    // slot 3, slot 4, slot 5
    uint256[3] private arr_1 = [4, 5, 6];
    // slot 6, slot 6, slot 7, slot 7, slot 8
    uint128[5] private arr_2 = [7, 8, 9, 10, 11];

    function test_arr_0(uint256 i) public view returns(uint256 v) {
        assembly {
            v := sload(i)
        }
    }

    function test_arr_1(uint256 i) public view returns(uint256 v) {
        assembly {
            v := sload(add(3, i))
        }
    }

    function test_arr_2(uint256 i) public view returns(uint128 v) {
        assembly {
            let b32 := sload(add(6, div(i , 2)))
            // 6 + (0 / 2) = 6
            // 6 + (1 / 2) = 6
            // 6 + (2 / 2 = 7
            // 6 + (3 / 2 = 7
            // 6 + (4 / 2 = 8

            // slot 6, 6, 7, 7, 8
            //      [7,8,9,10,11]
            //       0,1,2, 3, 4
            
            // slot 6 = 1st element | 0th element
            // slot 7 = 3rd element | 2nd element
            // slot 8 = 000 ... 000 | 4th element

            // i is even => get right 128 bits => cast to uint128
            // i is odd => get left 128 bits => shift right 128 bits

            switch mod(i, 2)
            case 1 { v := shr(128, b32) }
            default { v := b32 }
        }
    }
    
        // slot of element = keccak256(slot where array is declared)
    //                     + size of element * index of element
    // keccak256(0), keccak256(0) + 1, keccak256(0) + 2
    uint256[] private arr = [11, 22, 33];
    // keccak256(1), keccak256(1), keccak256(1) + 1 => 0.5 * 2
    uint128[] private arr_2 = [1, 2, 3];

    function test_arr(uint256 slot, uint256 i) 
        public view returns(uint256 val, bytes32 b32, uint256 len)
    {
        bytes32 start = keccak256(abi.encode(slot));
        assembly {
            len := sload(slot)
            val := sload(add(start, i))
            b32 := val
        }
    }

}
```

## Revert

`revert(p, s) → end execution, revert state changes, return data mem[p…(p+s))`

- `p` and `s` are parameters that describe memory segment containing the data to return.
- `p` is the pointer to the starting position in memory where the data should be returned.
- `s` represents the size of the data in bytes being returned.
- It extends up to, but does not include, **`p + s`**. This means the data returned will be from memory locations **`p`** to **`p+s-1`**. So if `p` is 32 and `s` is 10, the data returned will be include bytes in memory from `32 - 41`

```solidity
        (bool success, bytes memory data) = address(this).staticcall(.......
        ......
                revert(add(32, data), mload(data))
              // mload(data) reads the length bytes of data (the size)
							// Datasize = 12,
							// Memory pointer := 32 + 12, return data from 32-43
							// This skips over the length of the data and points to the actual data
```