# Nysa Proposal
The goal of the Nysa project is to translate contracts written in Solidity to Rust. With this approach,
smart contracts will be as efficient as those written using the native framework. Our initial tests 
are yielding very promising results.
TODO!(put some numbers)

## What problem do we solve?
The main advantage of smart contracts written in Solidity is that they are already
written and battle-tested - hence considered secure.
Platforms that offer writing smart contracts in Rust suffer from small amount
of libraries and example code, so they look at Solidity codebase with jealousy.
One approach is to translate EVM bytecode into WASM bytecode. The main issue of this solutions is 
the efficiency and compatibility (?).

## What do we offer?
We strongly believe there is a better solution: **transpiling Solidity code into Rust** code.
It allows us to take advantage of the already existing Rust to WASM compilation pipeline.
In return, we get the same neat, easy, and secure but native code.

## How we do it?
In terms of programming languages, the list of Solidity features can be treated
as a subset of Rust's features.
That means it's possible to convert Solidity into Rust, but the opposite is not possible.
Our architecture is split into 3 main pieces:

- `Solidity Parser` build on top of [lalrpop](https://github.com/lalrpop/lalrpop).
  It parses Solidity Syntax into developer-friendly Solidity AST.
  We have borrowed it as a whole from [the Solang project](https://github.com/hyperledger/solang).
  Further development might require some modification, but it can be considered 95% complete.

- `C3 Lang` is our approach to implement Solidity's (and Python's) inheritance model.
  We designed it as an intermediate target for the transpiler.
  While it works, it needs better tests, redesign, and documentation.

- `Nysa Core` translates the output of the `Solidity Parser` into a `C3 Lang` Rust.

The pipeline looks like this:
```
Solidity Code -> Solidity Parser -> Solidity AST -> C3 Lang AST -> Rust Code
```

## Nysa codebase
We have already managed to implement a significant set of features to transpile Solidity code.
It lives in [the Nysa repository](https://github.com/odradev/nysa).

## Owned Token example
We have a non-trivial example implemented - OwnedToken which is a basic Erc20 token with ownership.


The target contract - `OwnedToken` inherits from both `Context` and `ERC20`.

Solidity code:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Owner {
    address private owner;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Only the contract owner can call this function.");
        _;
    }

    function transferOwnership(address newOwner) public onlyOwner {
        owner = newOwner;
    }

    function getOwner() public view returns (address) {
        return owner;
    }
}

contract ERC20 {
    string public name;
    string public symbol;
    uint8 public decimals;
    uint256 public totalSupply;

    mapping(address => uint256) public balanceOf;

    event Transfer(address indexed from, address indexed to, uint256 value);

    constructor(string memory _name, string memory _symbol, uint8 _decimals, uint256 _initialSupply) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        totalSupply = _initialSupply * 10**uint256(decimals);
        balanceOf[msg.sender] = totalSupply;
    }

    function _transfer(address _from, address _to, uint256 _value) internal {
        require(_to != address(0), "Invalid recipient address.");
        require(balanceOf[_from] >= _value, "Insufficient balance.");

        balanceOf[_from] -= _value;
        balanceOf[_to] += _value;

        emit Transfer(_from, _to, _value);
    }

    function transfer(address _to, uint256 _value) public {
        _transfer(msg.sender, _to, _value);
    }
}

contract OwnedToken is Owner, ERC20 {
    constructor(string memory _name, string memory _symbol, uint8 _decimals, uint256 _initialSupply)
        ERC20(_name, _symbol, _decimals, _initialSupply)
    {}

    function mint(address _to, uint256 _amount) public onlyOwner {
        require(_to != address(0), "Invalid recipient address.");
        totalSupply += _amount;
        balanceOf[_to] += _amount;
        emit Transfer(address(0), _to, _amount);
    }
}
```

At first glance, the contract may seem simple, but it covers many Solidity concepts.
The `Owner` contract has a simple *state*, implements a *constructor*, and possesses the 'onlyOwner' *modifier*, 
which *reverts* the transaction if *`msg.sender`* is not the contract owner. On the other hand, 
the 'ERC20' state contains a *mapping* and defines the 'Transfer' *event*, which is *emitted* upon `transfer`. 
The contract logic consists of various *arithmetic* and *logical operators*. Finally, the main contract, `OwnedToken` 
*inherits* from `Context` and `ERC20` and calls super contracts constructors.

It is transpiled into the following Rust code:
```rust
pub mod errors {}
pub mod events {
    #[derive(odra :: Event, PartialEq, Eq, Debug)]
    pub struct Transfer {
        from: Option<odra::types::Address>,
        to: Option<odra::types::Address>,
        value: nysa_types::U256,
    }
    impl Transfer {
        pub fn new(
            from: Option<odra::types::Address>,
            to: Option<odra::types::Address>,
            value: nysa_types::U256,
        ) -> Self {
            Self { from, to, value }
        }
    }
}
pub mod enums {}
pub mod owned_token {
    // truncated
    // ...

    use super::errors::*;
    use super::events::*;
    use odra::prelude::vec::Vec;

    #[derive(Clone, Default)]
    struct PathStack {
        stack: alloc::rc::Rc<core::cell::RefCell<Vec<Vec<ClassName>>>>,
    }
    // truncated
    // ...

    #[derive(Clone)]
    enum ClassName {
        OwnedToken,
        ERC20,
        Owner,
    }
    # [odra :: module (events = [Transfer])]
    pub struct OwnedToken {
        __stack: PathStack,
        name: odra::Variable<odra::prelude::string::String>,
        symbol: odra::Variable<odra::prelude::string::String>,
        decimals: odra::Variable<nysa_types::U8>,
        total_supply: odra::Variable<nysa_types::U256>,
        balance_of: odra::Mapping<Option<odra::types::Address>, nysa_types::U256>,
        owner: odra::Variable<Option<odra::types::Address>>,
    }
    #[odra::module]
    impl OwnedToken {
        const PATH: &'static [ClassName; 3usize] =
            &[ClassName::Owner, ClassName::ERC20, ClassName::OwnedToken];

        #[odra(init)]
        pub fn init(
            &mut self,
            _name: odra::prelude::string::String,
            _symbol: odra::prelude::string::String,
            _decimals: nysa_types::U8,
            _initial_supply: nysa_types::U256,
        ) {
            self._owner_init();
            self._erc_20_init(_name, _symbol, _decimals, _initial_supply);
        }
        fn _owner_init(&mut self) {
            self.owner.set(Some(odra::contract_env::caller()));
        }
        fn _erc_20_init(
            &mut self,
            _name: odra::prelude::string::String,
            _symbol: odra::prelude::string::String,
            _decimals: nysa_types::U8,
            _initial_supply: nysa_types::U256,
        ) {
            self.name.set(_name);
            self.symbol.set(_symbol);
            self.decimals.set(_decimals);
            self.total_supply.set(
                (_initial_supply
                    * nysa_types::U256::from_limbs_slice(&[10u64])
                        .pow(nysa_types::U256::from(*self.decimals.get_or_default()))),
            );
            self.balance_of.set(
                &Some(odra::contract_env::caller()),
                self.total_supply.get_or_default(),
            );
        }

        fn _transfer(
            &mut self,
            _from: Option<odra::types::Address>,
            _to: Option<odra::types::Address>,
            _value: nysa_types::U256,
        ) {
            self.__stack.push_path_on_stack(Self::PATH);
            let result = self.super__transfer(_from, _to, _value);
            self.__stack.drop_one_from_stack();
            result
        }
        fn super__transfer(
            &mut self,
            _from: Option<odra::types::Address>,
            _to: Option<odra::types::Address>,
            _value: nysa_types::U256,
        ) {
            let __class = self.__stack.pop_from_top_path();
            match __class {
                ClassName::ERC20 => {
                    if !(_to != None) {
                        odra::contract_env::revert(odra::types::ExecutionError::new(
                            1u16,
                            "Invalid recipient address.",
                        ))
                    };
                    if !(self.balance_of.get_or_default(&_from) >= _value) {
                        odra::contract_env::revert(odra::types::ExecutionError::new(
                            2u16,
                            "Insufficient balance.",
                        ))
                    };
                    self.balance_of
                        .set(&_from, self.balance_of.get_or_default(&_from) - _value);
                    self.balance_of
                        .set(&_to, self.balance_of.get_or_default(&_to) + _value);
                    <Transfer as odra::types::event::OdraEvent>::emit(Transfer::new(
                        _from, _to, _value,
                    ));
                }
                #[allow(unreachable_patterns)]
                _ => self.super__transfer(_from, _to, _value),
            }
        }
        pub fn get_owner(&self) -> Option<odra::types::Address> {
            self.__stack.push_path_on_stack(Self::PATH);
            let result = self.super_get_owner();
            self.__stack.drop_one_from_stack();
            result
        }
        fn super_get_owner(&self) -> Option<odra::types::Address> {
            let __class = self.__stack.pop_from_top_path();
            match __class {
                ClassName::Owner => {
                    return self.owner.get().unwrap_or(None);
                }
                #[allow(unreachable_patterns)]
                _ => self.super_get_owner(),
            }
        }
        
        pub fn mint(&mut self, _to: Option<odra::types::Address>, _amount: nysa_types::U256) {
            self.__stack.push_path_on_stack(Self::PATH);
            let result = self.super_mint(_to, _amount);
            self.__stack.drop_one_from_stack();
            result
        }
        fn super_mint(&mut self, _to: Option<odra::types::Address>, _amount: nysa_types::U256) {
            let __class = self.__stack.pop_from_top_path();
            match __class {
                ClassName::OwnedToken => {
                    self.modifier_before_only_owner();
                    if !(_to != None) {
                        odra::contract_env::revert(odra::types::ExecutionError::new(
                            1u16,
                            "Invalid recipient address.",
                        ))
                    };
                    self.total_supply
                        .set(self.total_supply.get_or_default() + _amount);
                    self.balance_of
                        .set(&_to, self.balance_of.get_or_default(&_to) + _amount);
                    <Transfer as odra::types::event::OdraEvent>::emit(Transfer::new(
                        None, _to, _amount,
                    ));
                    self.modifier_after_only_owner();
                }
                #[allow(unreachable_patterns)]
                _ => self.super_mint(_to, _amount),
            }
        }
        fn modifier_before_only_owner(&mut self) {
            if !(Some(odra::contract_env::caller()) == self.owner.get().unwrap_or(None)) {
                odra::contract_env::revert(odra::types::ExecutionError::new(
                    3u16,
                    "Only the contract owner can call this function.",
                ))
            };
        }
        fn modifier_after_only_owner(&mut self) {}
        pub fn transfer(&mut self, _to: Option<odra::types::Address>, _value: nysa_types::U256) {
            self.__stack.push_path_on_stack(Self::PATH);
            let result = self.super_transfer(_to, _value);
            self.__stack.drop_one_from_stack();
            result
        }
        fn super_transfer(&mut self, _to: Option<odra::types::Address>, _value: nysa_types::U256) {
            let __class = self.__stack.pop_from_top_path();
            match __class {
                ClassName::ERC20 => {
                    self._transfer(Some(odra::contract_env::caller()), _to, _value);
                }
                #[allow(unreachable_patterns)]
                _ => self.super_transfer(_to, _value),
            }
        }
        pub fn transfer_ownership(&mut self, new_owner: Option<odra::types::Address>) {
            self.__stack.push_path_on_stack(Self::PATH);
            let result = self.super_transfer_ownership(new_owner);
            self.__stack.drop_one_from_stack();
            result
        }
        fn super_transfer_ownership(&mut self, new_owner: Option<odra::types::Address>) {
            let __class = self.__stack.pop_from_top_path();
            match __class {
                ClassName::Owner => {
                    self.modifier_before_only_owner();
                    self.owner.set(new_owner);
                    self.modifier_after_only_owner();
                }
                #[allow(unreachable_patterns)]
                _ => self.super_transfer_ownership(new_owner),
            }
        }
    }
}
```

Full code with tests lives [here](https://github.com/odradev/nysa/tree/feature/odra/examples/owned-token/nysa/src).

### Odra vs Nysa
Compare

# Timeline
Proposed list of milestones.

## Milestone #1 MVP (UniswapV2)
This milestone aims to transpile Uniswap v2 [core contracts](https://github.com/Uniswap/v2-core/blob/master/contracts).
The code can be transpiled to Rust (compatible with the Odra Framework) and compiled into a wasm file.

Translating Uniswap v2 contracts from Solidity to Rust requires handling all the fundamental language features, starting from contract declarations, interfaces, state definition, function invocations, event emission, error handling, ensuring type compatibility, all the way to calling external contracts.

Additionally, besides purely syntactic issues, addressing the inheritance model in Solidity, contextual parsing of code, and creating contract factory are also necessary.

Covered Solidity features:
- Functions
    * modifiers,
    * constructors,
    * function calls,
    * returns (includes returing multiple values and named values).
- Contract state variables 
    * single values,
    * mappings,
    * nested mappings,
    * assiging default value,
    * default getters for the public variables.
- Types
    * integers (singned/unsigned),
    * bool,
    * string,
    * address,
    * arrays,
    * bytes,
    * custom enums and structs
- Checked/unchecked arithmetic.
- Local variables.
- Error handling.
- Control Structures
    * if/else
    * while loops
    * tenary operator (? :).
- Operators.
    * logical operators,
    * math operators,
    * bitwise operators.
- Basic special variables and functions
    * msg.sender
    * msg.value
    * block.timestamp
    * abi.encode
    * abi.decode
    - Inheritance.
- Events.
- External contract calls.
- Libraries
- [Salted contract creations / create2](https://docs.soliditylang.org/en/v0.8.21/control-structures.html#salted-contract-creations-create2)

## Milestone #2 Full syntax support

Although the functionality set required by Uniswap v2 smart contracts is relatively rich, it does not exhaust all that is available in Solidity. 
The second milestone aims to address the missing language features in terms of syntax as well as idiomatic expressions.

The secondary goal is to fully cover with tests the order of evaluation of expressions.

Solidity contracts written using the OpenZeppelin framework are compatible with Nysa.

## Milestone #3 Docs
The documentation of all the supported features and discrepancies to EVM are documented.
It includes tutorials and an introduction for Solidity developers.

## Milestone #4 Solidity tests
Tests written in solidity can be transpiled into Rust tests.

E2E JavaScript tests written for OpenZeppelin can be run against the Casper Virtual Machine provided with the Odra Framework.
The tests are passing.
