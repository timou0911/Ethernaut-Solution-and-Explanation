[Level 18. MagicNumber](https://ethernaut.openzeppelin.com/level/0x2132C7bc11De7A90B87375f282d36100a29f97a9)

## In A Nutshell

> Deconstructing contracts from EVM level.

## Concept

1. EVM opcode and understanding of stack.
2. Contract bytecode comprises two parts: creation bytecode and runtime bytecode.

## Level Target

1. Create a contract `Solver` with a size of no more than 10 bytes (10 opcodes).

## Breakdown & Analysis

1. Writing Solidity to break this level is impossible due to the size limit, so we need to write opcode instead.
2. We need to divide our contract into creation bytecode and runtime bytecode, and then address them separately.
3. Creation bytecode should copy the runtime bytecode and then return it; runtime bytecode should return 42 in this case.

## Detailed Steps

`Complete bytecode = creation bytecode + runtime bytecode`

One thing to note is that we must push the opcode's parameters in the stack before using them, and stack is LIFO (Last In First Out).

### Runtime Bytecode

Runtime bytecode is the actual code deployed on chain, so we should limit its size to no more than 10 bytes.

1. To return 42(`0x2a`), the opcode used is `RETURN`(`0xf3`). It takes two parameters: offset(the memory location of `0x2a`) and its size.
2. To store `0x2a` in memory, the opcode used is `MSTORE`(`0x52`) It takes two parameters: offset(the memory location where `0x2a` will be stored) and the value to store(`0x2a`).
3. All parameters must be stored in the stack before use, so we need to call `PUSH1`(`0x60`) for each parameter.

---

1. Push `0x2a` in stack as the value for `MSTORE` → `PUSH1 0x2a` → `602a`.
2. Push `0x70` in stack as the offset for `MSTORE` → `PSH1 0x50` → `6070`.
3. Store `0x2a` in memory location `0x70` → `MSTORE 0x70, 0x2a` → `52`.
4. Push `0x20` in stack as the offset for `RETURN` → `PUSH1 0x20` → `6020`.
5. Push `0x70` in stack as the memory location for `RETURN` → `PUSH1 0x70` → `6070`.
6. Return data in size of `0x20` from memory location `0x70` → `RETURN 0x70, 0x20` → `f3`

⇒ Runtime bytecode = `602a60705260206070f3` in size of 10 bytes.

### Creation Bytecode

Creation bytecode serves as runtime bytecode loader and returns runtime bytecode to EVM.

1. To copy runtime bytecode, the opcode `CODECOPY` is used. It takes three parameters: destOffset(destination in the memory where the result will be copied), offset(byte offset in the code to copy), and code size(10 bytes = `0x0a` in this case).
2. After copying runtime bytecode to memory, we use `RETURN` opcode to return it.

---

1. Push `0x0a` in stack as the size for `CODECOPY` → `PUSH1 0x0a` → `600a`.
2. Push `0x??` in stack as the offset for `CODECOPY` → `PUSH1 0x??` → `60??`. (we will figure the position of runtime bytecode out after knowing the creation bytecode size)
3. Push `0x00` in stack as the destOffset for `CODECOPY` → `PUSH1 0x00` → `6000`.
4. Copy code to memory with destOffset `0x00`, offset `0x??`, and size `0x0a` → `CODECOPY 0x00, 0x??, 0x0a` → `39`
5. Push `0x0a` in stack as the size for `RETURN` → `PUSH1 0x0a` → `600a`.
6. Push `0x00` in stack as the offset for `RETURN` → `PUSH1 0x00` → `6000`.
7. Return data in size of `0x0a` from memory location `0x00` → `RETURN 0x00, 0x0a` → `f3`.

⇒ Creation bytecode = `600a60??600039600a6000f3` in size of 12 bytes, so runtime bytecode starts at position 12 (`0x0c`). `??` = `0c`.

### The Complete Opcode

Complete bytecode = creation bytecode + runtime bytecode = `600a600c600039600a6000f3` + `602a60705260206070f3` = `600a600c600039600a6000f3602a60705260206070f3`. This will be the `data` when we send the transaction.

### Deploy the Contract and Solve It

``` js
tx = await web3.eth.sendTransaction({from: player, data: "600a600c600039600a6000f3602a60705260206070f3"}); // send the contract creation tx
await contract.setSolver(tx.contractAddress); // call the function
```

## Contract Creation Phases

### 1. Compile Time

Solidity compiler(solc) generates two files: `.bin`(bytecode) & `.abi`

### 2. Initialization & Deployment

Creation bytecode initializes the contract and can only be executed by EVM once, so it's not stored on chain. It performs several operations:

1. **Free Memory Pointer Initialization**: set the value of the free memory pointer located at `0x40` as `0x80`.
2. **Constructor Non-payable Check**: if the `constructor` is not labeled as `payable` but `msg.value` is greater than 0, the contract creation transaction will be reverted.
3. **Constructor Execution**: execute the content inside the `constructor`. (since `constructor` only executed once in the initialization phase, it's not stored on chain)
4. **Runtime Bytecode Returning and Storing**: use `CODECOPY` to copy runtime bytecode into memory and return it to store it on chain.

### 3. Runtime

The actual code stored on the blockchain, awaiting trigger.

Reference: [EVM Part II: The Journey of Smart Contracts from Solidity code to Bytecode](https://medium.com/coinmonks/evm-part-ii-the-journey-of-smart-contracts-from-solidity-code-to-bytecode-a21300b23dee)
