# Solidity Docs (*Notes from readthedocs*)

## Introduction to Smart Contract
### A simple Smart Contract
##### Storage Example
```javascript
pragma solidity ^0.4.0;

contract SimpleStorage {
    uint storedData;

    function set(uint x) public {
        storedData = x;
    }

    function get() public constant returns (uint) {
        return storedData;
    }
}
```
The first line simply tells that the source code is written for Solidity version 0.4.0 or anything newer that does not break functionality (up to, but not including, version 0.5.0).

A contract in the sense of Solidity is a collection of code (its functions) and data (its state) that resides at a specific address on the Ethereum blockchain.

The line `uint storedData;` declares a state variable called `storedData` of type `uint` (unsigned integer of 256 bits).

The functions `set` and `get` can be used to modify or retrieve the value of the variable.

This contract does not do much yet (due to the infrastructure built by Ethereum) apart from allowing anyone to store a single number that is accessible by anyone in the world without a (feasible) way to prevent you from publishing this number. Of course, anyone could just call set again with a different value and overwrite your number, but the number will still be stored in the history of the blockchain.

##### Subcurrency Example
The following contract will implement the simplest form of a cryptocurrency. It is possible to generate coins out of thin air, but only the person that created the contract will be able to do that (it is trivial to implement a different issuance scheme). Furthermore, anyone can send coins to each other without any need for registering with username and password - all you need is an Ethereum keypair.
```javascript
pragma solidity ^0.4.21;

contract Coin {
    // The keyword "public" makes those variables
    // readable from outside.
    address public minter;
    mapping (address => uint) public balances;

    // Events allow light clients to react on
    // changes efficiently.
    event Sent(address from, address to, uint amount);

    // This is the constructor whose code is
    // run only when the contract is created.
    function Coin() public {
        minter = msg.sender;
    }

    function mint(address receiver, uint amount) public {
        if (msg.sender != minter) return;
        balances[receiver] += amount;
    }

    function send(address receiver, uint amount) public {
        if (balances[msg.sender] < amount) return;
        balances[msg.sender] -= amount;
        balances[receiver] += amount;
        emit Sent(msg.sender, receiver, amount);
    }
}
```
The line address `public minter;` declares a state variable of type address that is publicly accessible. The `address` type is a 160-bit value that does not allow any arithmetic operations. It is suitable for storing addresses of contracts or keypairs belonging to external persons. The keyword `public` automatically generates a function that allows you to access the current value of the state variable from outside of the contract.

The next line, `mapping (address => uint) public balances;` also creates a public state variable, but it is a more complex datatype. The type maps addresses to unsigned integers. Mappings can be seen as *hash tables* which are virtually initialized such that every possible key exists and is mapped to a value whose byte-representation is all zeros.

The line `event Sent(address from, address to, uint amount);` declares a so-called “event” which is emitted in the last line of the function `send`. User interfaces (as well as server applications of course) can listen for those events being emitted on the blockchain without much cost. As soon as it is emitted, the listener will also receive the arguments `from`, `to` and `amount`, which makes it easy to track transactions. In order to listen for this event, you would use:
```javascript
Coin.Sent().watch({}, '', function(error, result) {
    if (!error) {
        console.log("Coin transfer: " + result.args.amount +
            " coins were sent from " + result.args.from +
            " to " + result.args.to + ".");
        console.log("Balances now:\n" +
            "Sender: " + Coin.balances.call(result.args.from) +
            "Receiver: " + Coin.balances.call(result.args.to));
    }
})
```
Finally, the functions that will actually end up with the contract and can be called by users and contracts alike are `mint` and `send`. If `mint` is called by anyone except the account that created the contract, nothing will happen. On the other hand, `send` can be used by anyone (who already has some of these coins) to send coins to anyone else.

### Blockchain Basics
##### Transactions
A blockchain is a globally shared, transactional database. This means that everyone can read entries in the database just by participating in the network.

If you want to change something in the database, you have to create a so-called transaction which has to be accepted by all others.

The word transaction implies that the change you want to make (assume you want to change two values at the same time) is either not done at all or completely applied. Furthermore, while your transaction is applied to the database, no other transaction can alter it.

##### Blocks
One major obstacle to overcome is what, in Bitcoin terms, is called a **“double-spend attack”**: What happens if two transactions exist in the network that both want to empty an account, a so-called conflict?

The abstract answer to this is that you do not have to care. An order of the transactions will be selected for you, the transactions will be bundled into what is called a **“block”** and then they will be executed and distributed among all participating nodes. If two transactions contradict each other, the one that ends up being second will be rejected and not become part of the block.

These blocks form a *linear sequence in time* and that is where the word **“blockchain”** derives from. Blocks are added to the chain in rather regular intervals - for Ethereum this is roughly every 17 seconds.

As part of the *“order selection mechanism”* (which is called **“mining”**) it may happen that blocks are reverted from time to time, but only at the “tip” of the chain. The more blocks that are added on top, the less likely it is. So it might be that your transactions are reverted and even removed from the blockchain, but the longer you wait, the less likely it will be.

### The Ethereum Virtual Machine
