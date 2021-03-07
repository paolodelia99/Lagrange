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

Here functions have the same Python syntax, and they are executable units of code within contracts. Functions may be called internally (using the `@internal` annotation) or externally (using the `@external` annotation) depending on their visibility. 

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

To make all the ERC20 tokens re-used by other applications a [standard](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md) has been defined by Fabian Vogelsteller and Vitalik Buterin. 
Since I want to make a proper token, I've decided to adhere to these standards.

The good thing is that Vyper has a [built-in ERC20 token interface](https://github.com/vyperlang/vyper/blob/master/vyper/interfaces/ERC20.py)! Let's first import it:

```python
# @version >=0.2.11 <0.3.0

from vyper.interfaces import ERC20

implements: ERC20
```

The first thing that comes to my mind when I think about a crypto token is scarcity. So let's fix a maximum cap of 1000000 tokens with an initial supply of 100000 tokens. And now you might wonder, why did you choose to use a cap of 1000000 tokens? Well, ... why not?

Here's how the code is gonna look like: 

```python
# @version >=0.2.11 <0.3.0

from vyper.interfaces import ERC20

implements: ERC20

MAX_SUPPLY: constant(uint256) = 1000000
INIT_SUPPLY: constant(uint256) = 100000

totalSupply: public(uint256)

```

Are we going to mint those tokens or are we going not to event touch them? Maybe, we should write a function that is going to mint those tokens.

```python
@external
def mint(_account: address, _value: uint256) -> bool:
	
	if self.totalSupply + _value <= MAX_SUPPLY:
		self.totalSupply += _value
		self.balances[_account] += _value

	log Transfer(empty(address), _account, _value)
	return True
```

The `mint` function is pretty straightforward to understand. Notice that I've added the state variable `balcances` which is and `HashMap[address, uint256]` that contains all the balances of all the possible addresses, and the `Transfer` event which is an event that is log every time that a transaction has been made.

Now the whole contract looks like this:

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

To make the contract even more standard we need to add a few state variables which are: the `name`, the `symbol`, `allowances`, and the `Approval` event.
`name` and `symbol` are self-explanatory variables instead, I would like to focus on the `allowances` state variable. The `allowances` are a `HasMap[address, HasMap[address, uint256]]` and stores the amount the `spender` is allowed to withdraw from  the `owner`.

To give an example an example, let's say that `Bob` has as address `0x53FB636Da5708A3Ec1D6544F543F8856577F315C` and `Alice` `0x12fE305d63E655317fa7E708aD93C83Bf26EcC47`. Suppose now, to have the following situation: 

 holder address | allowance address | allowance 
 0x53FB...F315C | 0x12fE...EcC47 | 25

what the hash table is telling us is that Bob has allowed Alice to transfer up to 25 tokens from Bob's account. 

To change the allowance we use the function `approve(_spender: address, _value: uint256) -> bool`. `approve` allows `_spender` to withdraw from your account (`msg.sender`, or who else has called the contract) multiple times, up to the `_value` amount. If the function call was a success and `Approval` event is logged.

To initialize some of the state variables when the contract is deployed we have to define, like in Python, the `__init__` function.

Here's how I've implemented the `__init__` function:


```python
@external 
def __init__(founder: address):
	self.totalSupply = INIT_SUPPLY
	self.name = "Paolown Coin"
	self.symbol = "PLW"
	self.founder = founder
	self.balances[self.founder] = self.totalSupply
```

I've also added a `founder` state variable which is the address of the founder/owner of the contract who initially holds all the coins. 

Now we only missing the functions that can allow the token transfer between the users. In the ERC20 standards are: `transfer(_to: address, _value: uint256) -> bool` and `transferFrom(_from: address, _to: address, _value: uint256) -> bool`. 

`transfer` transfers `_value` amount of tokens to address `_to`, and fire the `Transfer` event.
Whereas, `transferFrom` transfers `_value` amount of tokens from address `_from` to address `_to`, but different from `transfer` is used for a withdraw workflow, allowing contracts to transfer tokens on your behalf.

By putting all the pieces together we got the following contract:

```python
# @version >=0.2.11 <0.3.0

from vyper.interfaces import ERC20

implements: ERC20

MAX_SUPPLY: constant(uint256) = 1000000
INIT_SUPPLY: constant(uint256) = 100000

founder: address

totalSupply: public(uint256)
name: public(String[32])
symbol: public(String[5])
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

event Transfer:
	sender: indexed(address)
	receiver: indexed(address)
	amount: uint256

event Approval:
	owner: indexed(address)
	spender: indexed(address)
	value: uint256

@external
def __init__(founder: address):
	self.totalSupply = INIT_SUPPLY
	self.name = "Paolown Coin"
	self.symbol = "PLW"
	self.founder = founder
	self.balances[self.founder] = self.totalSupply

@view
@external
def get_max_supply() -> uint256:
	return MAX_SUPPLY

@internal
def _transferCoins(_src: address, _dst: address, _amount: uint256):
	assert _src != empty(address), "PLW::_transferCoins: cannot transfer from the zero address"
	assert _dst != empty(address), "PLW::_transfersCoins: cannot transfer to the zero address"

	self.balances[_src] -= _amount
	self.balances[_dst] += _amount

@external
def transfer(_to: address, _value: uint256) -> bool:
	assert self.balances[msg.sender] >= _value, "PLW::transfer: Not enough coins"

	self._transferCoins(msg.sender, _to, _value)

	log Transfer(msg.sender, _to, _value)
	return True

@external
def transferFrom(_from: address, _to: address, _value: uint256) -> bool:
	allowance: uint256 = self.allowances[_from][msg.sender]
	assert self.balances[_from] >= _value and allowance >= _value

	self._transferCoins(_from, _to, _value)

	self.allowances[_from][msg.sender] -= _value
	log Transfer(_from, _to, _value)
	return True

@view
@external
def balanceOf(_owner: address) -> uint256:
	return self.balances[_owner]

@view
@external
def allowance(_owner: address, _spender: address) -> uint256:
	return self.allowances[_owner][_spender]

@external 
def approve(_spender: address, _value: uint256) -> bool:
	self.allowances[msg.sender][_spender] = _value
	log Approval(msg.sender, _spender, _value)
	return True

@external
def increaseAllowance(spender: address, _value: uint256) -> bool:
    assert spender != empty(address)

    self.allowances[msg.sender][spender] += _value
    log Approval(msg.sender, spender, self.allowances[msg.sender][spender])
    return True

@external
def decreaseAllowance(spender: address, _value: uint256) -> bool:
    assert spender != empty(address)

    self.allowances[msg.sender][spender] -= _value
    log Approval(msg.sender, spender, self.allowances[msg.sender][spender])
    return True

@external
def mint(_account: address, _value: uint256) -> bool:
	
	if self.totalSupply + _value <= MAX_SUPPLY:
		self.totalSupply += _value
		self.balances[_account] += _value

	log Transfer(empty(address), _account, _value)
	return True
```

For more info you can check out the [repo](https://github.com/paolodelia99/paolown-coin).

# Further Improvements

- for the sake of simplicity, I didn't make my token divisible, but you can make them divisible by adding the state variable `decimals: uint256` which stores how decimal values are you using. Of course, to make it work properly, you have to multiply the whole number of tokens by $10^{decimals}$
- together with the method of `mint` the token you can add a function that can `burn` token reducing the total supply, making them more scarce.

# References

- [ERC20 Token Standard](https://eips.ethereum.org/EIPS/eip-20)
- [Understanding ERC-20 token contracts](https://www.wealdtech.com/articles/understanding-erc20-token-contracts/)
- [Vyper docs](https://vyper.readthedocs.io/en/latest/index.html)