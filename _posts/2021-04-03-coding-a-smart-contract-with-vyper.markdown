---
layout: post
title:  "Coding an ERC20 token smart contract with Vyper"
comments: true
date:   2021-03-04 22:13:42 +0200
tags: [coding, vyper, ethereum, blockchain]
author: "Paolo"
image: "vyper-logo-transparent.png"
cut: "-50"
---

As I'm getting to know more and more crypto space and, especially the Ethereum ecosystem I'm kinda wondering how those smart contracts work. This idea of a programmable blockchain sounds so cool to me! So I've decided to get to know to build on of those smart contracts starting from the silliest thing that one person could do: building his own coin.

The main difference from the Ethereum blockchain from the Bitcoin one is that the Ethereum blockchain is a "programmable blockchain". But how is this blockchain programmable? The Ethereum accounts can be of two types: 

- an **externally owned accounts** controlled by private keys
- a **contract account** controlled by their contract code.

But how the hell an account can be controlled by his code? Well, I forgot to say that an Etherum account contains four fields:

- the **nonce**, a counter used to make sure each transaction can only be processed once
- The account's current **ether balance**
- The account's **contract code**, if present
- The account's **storage** (which is empty by default)


So in a contract account, every time the contract account receives a message its code activates, allowing it to read and write to internal storage and send other messages or create contracts in turn.

You can see a smart contract as an "*autonomous agent*" that live inside the Ethereum blockchain, always executing a specific piece of code when "poked" by a message or transaction, and having direct control over their own ether balance and their own key/value store to keep track of persistent variables.

## First look at Vyper

Coming from a python background I was looking for a smart contract language similar to it. After a little bit of research, I found **Vyper**, the perfect fit!

Vyper syntax is almost equal to the Python one, however is a statically types language and it has some constructs that are specific to the Ethereum smart contract.

Let's take look at an example:

```python
# @version >=0.2.11 <0.3.0

event Payment:
    amount: int128
    sender: indexed(address)

total_paid: int128

@external
@payable
def pay():
    self.total_paid += msg.value
    log Payment(msg.value, msg.sender)
```

The first line of the file represents the allowed versions of the contract, the versions string use npm style syntax.

`total_paid` is a state variable that is permanently stored in the contract storage. They are declared outside of the body of any functions, and initially contain the default value for their type. State variables are accessible in function via the `self`-object.

Here functions have the same exact Python syntax, and they are executable units of code within contracts. Functions may be called internally (using the `@internal` annotation) or externally (using the `@external` annotation) depending on their visibility. 

For defining your own data types in Vyper there aren't classes but just like in the C programming language you can use Structs. 
Here's how we can define a `Transaction` with a `struct`:  

```python
struct Transaction:
    spender: indexed(address)
    receiver: indexed(address)
    amount: uint256
```

The `event Payment` above is like of `struct` but is used to provide an interface for the EVM's logging facilities. Events may be logged with specially indexed data structures that allow clients, including light clients, to efficiently search for them.

Other useful constructs of the Vyper language are Interfaces. An interface is a set of function definitions used to enable communication between smart contracts. A contract interface defines all of that contract’s externally available functions. By importing the interface, your contract now knows how to call these functions in other contracts.

Interfaces can be added to contracts either through inline definition or by importing them from a separate file.

Here's an example of how to define an inline interface:

```python
interface FooBar:
    def calculate() -> uint256: view
    def test1(): nonpayable
```

For more info about the Vyper language, you can check out the [docs](https://vyper.readthedocs.io/en/latest/index.html).

## Writing the token contract

In order to make all the ERC20 token re-used by other application a [standard](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md) has been defined by Fabian Vogelsteller and Vitalik Buterin. 
Since I want to make a proper token, of course, I've decided to adhere to these standards.

The good thing is that Vyper has a [build-in ERC20 token interface](https://github.com/vyperlang/vyper/blob/master/vyper/interfaces/ERC20.py)! Let's first import it:

```python
# @version >=0.2.11 <0.3.0

from vyper.interfaces import ERC20

implements: ERC20
```

The first thing that comes to my mind when I think about a crypto token is scarcity. So let's fix a maximun cap of 1000000 token with and initial supply of 100000 tokens. And now you might wonder, why did you choose to use a cap of 1000000 token? Well, ... why not?

Here's how the code is gonna look like: 

```python
# @version >=0.2.11 <0.3.0

from vyper.interfaces import ERC20

implements: ERC20

MAX_SUPPLY: constant(uint256) = 1000000
INIT_SUPPLY: constant(uint256) = 100000

totalSupply: public(uint256)

```

Are we going to mint those tokens or are we going not to event touch them? Maybe, we shold write a function that is going to mint those tokens.

```python
@external
def mint(_account: address, _value: uint256) -> bool:
	
	if self.totalSupply + _value <= MAX_SUPPLY:
		self.totalSupply += _value
		self.balances[_account] += _value

	log Transfer(empty(address), _account, _value)
	return True
```

The `mint` function is pretty straightforward to understand. Notice that I've added the state variable `balcances` which is and `HashMap[address, uint256]` that contains all the balances of all the possible addresses, and the `Transfer` event which is an event that is log every time that a transation has been made.

Now whole contract look like:

```python
# @version >=0.2.11 <0.3.0

from vyper.interfaces import ERC20

implements: ERC20

MAX_SUPPLY: constant(uint256) = 1000000
INIT_SUPPLY: constant(uint256) = 100000

totalSupply: public(uint256)

balances: HashMap[address, uint256]

event Transfer:
	sender: indexed(address)
	receiver: indexed(address)
	amount: uint256

@external
def mint(_account: address, _value: uint256) -> bool:
	
	if self.totalSupply + _value <= MAX_SUPPLY:
		self.totalSupply += _value
		self.balances[_account] += _value

	log Transfer(empty(address), _account, _value)
	return True    
```

To make the contract even more standard we need to add a few state variable which are: the `name`, the `symbol` and the `Approval` event.
