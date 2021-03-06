---
eip: 712
title: Ethereum typed structured data hashing and signing
author: Remco Bloemen <remco@wicked.ventures>,
        Leonid Logvinov <logvinov.leon@gmail.com>
discussions-to: remco@wicked.ventures
status: Draft
type: Standards Track
category (*only required for Standard Track): Interface
created: 2017-09-12
requires (*optional): <EIP number(s)>
replaces (*optional): <EIP number(s)>
---
<!--
This is the suggested template for new EIPs.

Note that an EIP number will be assigned by an editor. When opening a pull request to submit your EIP, please use an abbreviated title in the filename, `eip-draft_title_abbrev.md`.

The title should be 44 characters or less.
-->

## Simple Summary
<!-- "If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the EIP. -->

Signing data is a solved problem if all we care about are bytestrings. Unfortunately in the real world we care about complex meaningful messages. Hashing structured data is non-trivial and errors result in loss of the security properties of the system.

As such, the adage "don't roll your own crypto" applies. Instead, a peer-reviewed well-tested standard method needs to be used. This EIP aims to be that standard.

## Abstract
<!-- A short (~200 word) description of the technical issue being addressed. -->

This is a standard for hashing and signing of typed structured data as opposed to just bytestrings. It includes a

*   theoretical framework for correctness of encoding functions,
*   specification of structured data similar to and compatible with Solidity structs,
*   safe hashing algorithm for instances of those structures,
*   safe inclusion of those instances in the set of signable messages,
*   an extensible mechanism for domain separation,
*   new RPC call `eth_signTypedData`, and
*   an optimized implementation of the hashing algorithm in EVM.

It does not include replay protection.

## Motivation
<!-- The motivation is critical for EIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright. -->

A signature scheme consists of hashing algorithm and a signing algorithm. The signing algorithm of choice in Ethereum is `secp256k1`. The hashing algorithm of choice is `keccak256`, this is a function from bytestrings, 𝔹⁸ⁿ, to 256-bit strings, 𝔹²⁵⁶.

A good hashing algorithm should satisfy security properties such as determinism, second pre-image resistance and collision resistance. The `keccak256` function satisfies the above criteria _when applied to bytestrings_. If we want to apply it to other sets we first need to map this set to bytestrings. It is critically important that this encoding function is [deterministic][deterministic] and [injective][injective]. If it is not deterministic then the hash might differ from the moment of signing to the moment of verifying, causing the signature to incorrectly be rejected. If it is not injective then there are two different elements in our input set that hash to the same value, causing a signature to be valid for a different unrelated message.

[deterministic]: https://en.wikipedia.org/wiki/Deterministic_algorithm
[injective]: https://en.wikipedia.org/wiki/Injective_function

### Transactions and bytestrings

An illustrative example of the above breakage can be found in Ethereum. Ethereum has two kinds of messages, transactions `𝕋` and bytestrings `𝔹⁸ⁿ`. These are signed using `eth_sendTransaction` and `eth_sign` respectively. Originally the encoding function `encode : 𝕋 ∪ 𝔹⁸ⁿ → 𝔹⁸ⁿ` was as defined as follows:

*   `encode(t : 𝕋) = RLP_encode(t)`
*   `encode(b : 𝔹⁸ⁿ) = b`

While individually they satisfy the required properties, together they do not. If we take `b = RLP_encode(t)` we have a collision. This is mitigated in Geth [PR 2940][geth-pr] by modifying the second leg of the encoding function:

[geth-pr]: https://github.com/ethereum/go-ethereum/pull/2940

*   `encode(b : 𝔹⁸ⁿ) = "\x19Ethereum Signed Message:\n" ‖ len(b) ‖ b` where `len(b)` is the ascii-decimal encoding of the number of bytes in `b`.

This solves the collision between the legs since `RLP_encode(t : 𝕋)` never starts with `\x19`. There is still the risk of the new encoding function not being deterministic or injective. It is instructive to consider those in detail.

As is, the definition above is not deterministic. For a 4-byte string `b` both encodings with `len(b) = "4"` and `len(b) = "004"` are valid. This can be solved by further requiring that the decimal encoding of the length has no leading zeros and `len("") = "0"`.

The above definition is not obviously collision free. Does a bytestring starting with `"\x19Ethereum Signed Message:\n42a…"` mean a 42-byte string starting with `a` or a 4-byte string starting with `2a`?. This was pointed out in [Geth issue #14794][geth-issue-14794] and motivated Trezor to [not implement the standard][trezor] as-is. Fortunately this does not lead to actual collisions as the total length of the encoded bytestring provides sufficient information to disambiguate the cases.

[geth-issue-14794]: https://github.com/ethereum/go-ethereum/issues/14794
[trezor]: https://github.com/trezor/trezor-mcu/issues/163

Both determinism and injectiveness would be trivially true if `len(b)` was left out entirely. The point is, it is difficult to map arbitrary sets to bytestrings without introducing security issues in the encoding function. Yet the current design of `eth_sign` still takes a bytestring as input and expects implementors to come up with an encoding.

### Arbitrary messages

The `eth_sign` call assumes messages to be bytestrings. In practice we are not hashing bytestrings but the collection of all semantically different messages of all different DApps `𝕄`. Unfortunately, this set is impossible to formalize so we approximate it with the set of typed named structures `𝕊` and a domain separator `𝔹²⁵⁶` to obtain the set `𝔹²⁵⁶ × 𝕊`. This standard formalizes the set `𝕊` and provides a deterministic injective encoding function for `𝔹²⁵⁶ × 𝕊`.

### Note on replay attacks

This standard is only about signing messages and verifying signatures. In many practical applications, signed messages are used to authorize an action, for example an exchange of tokens. It is _very important_ that implementers make sure the application behaves correctly when it sees the same signed message twice. For example, the repeated message should be rejected or the authorized action should be idempotent. How this is implemented is specific to the application and out of scope for this standard.


## Specification
<!-- The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (cpp-ethereum, go-ethereum, parity, ethereumj, ethereumjs, ...). -->

The set of signable messages is extended from transactions and bytestrings `𝕋 ∪ 𝔹⁸ⁿ` to also include structured data `𝕊`. The new set of signable messages is thus `𝕋 ∪ 𝔹⁸ⁿ ∪ 𝕊`. They are encoded to bytestrings suitable for hashing and signing as follows:

*   `encode(transaction : 𝕋) = RLP_encode(transaction)`
*   `encode(message : 𝔹⁸ⁿ) = "\x19Ethereum Signed Message:\n" ‖ len(message) ‖ message` where `len(message)` is the _non-zero-padded_ ascii-decimal encoding of the number of bytes in `message`.
*   `encode(domainSeparator : 𝔹²⁵⁶, message : 𝕊) = "\x19\x01" ‖ domainSeparator ‖ hashStruct(message)` where `domainSeparator` and `hashStruct(message)` are defined below.

This encoding is deterministic because the individual components are. The encoding is injective because the three cases always differ in first byte. (`RLP_encode(transaction)` does not start with `\x19`.)

The encoding is compliant with [EIP-191][eip191]. The 'version byte' is fixed to `0x01`, the 'version specific data' is the 32-byte domain separator `domainSeparator` and the 'data to sign' is the 32-byte `hashStruct(message)`.

[eip191]: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-191.md

### Definition of typed structured data `𝕊`

To define the set of all structured data, we start with defining acceptable types. Like ABIv2 these are closely related to Solidity types. It is illustrative to adopt Solidity notation to explain the definitions. The standard is specific to the Ethereum Virtual Machine, but aims to be agnostic to higher level languages. Example:

```cpp
struct Message {
    address from;
    address to;
    string contents;
}
```

**Definition**: A _struct type_ has valid identifier as name and contains zero or more member variables. Member variables have a member type and a name.

**Definition**: A _member type_ can be either an atomic type, a dynamic type or a reference type.

**Definition**: The _atomic types_ are `bytes1` to `bytes32`, `uint8` to `uint256`, `int8` to `int256`, `bool` and `address`. These correspond to their definition in Solidity. Note that there are no aliases `uint` and `int`. Note that contract addresses are always plain `address`. Fixed point numbers are not supported by the standard. Future versions of this standard may add new atomic types.

**Definition**: The _dynamic types_ are `bytes` and `string`. These are like the atomic types for the purposed of type declaration, but their treatment in encoding is different.

**Definition**: The _reference types_ are arrays and structs. Arrays are either fixed size or dynamic and denoted by `Type[n]` or `Type[]` respectively. Structs are references to other structs by their name. The standard supports recursive struct types.

**Definition**: The set of structured typed data `𝕊` contains all the instances of all the struct types.

### Definition of `hashStruct`

The `hashStruct` function is defined as

*   `hashStruct(s : 𝕊) = keccak256(typeHash ‖ encodeData(s))` where `typeHash = keccak256(encodeType(typeOf(s)))`

**Note**: The `typeHash` is a constant for a given struct type and does not need to be runtime computed.

### Definition of `encodeType`

The type of a struct is encoded as `name ‖ "(" ‖ member₁ ‖ "," ‖ member₂ ‖ "," ‖ … ‖ memberₙ ")"` where each member is written as `type ‖ " " ‖ name`. For example, the above `Message` struct is encoded as `Message(address from,address to,string contents)`.

If the struct type references other struct types (and these in turn reference even more struct types), then the set of referenced struct types is collected, sorted by name and appended to the encoding. An example encoding is `Transaction(Person from,Person to,Asset tx)Asset(address token,uint256 amount)Person(address wallet,string name)`.

### Definition of `encodeData`

The encoding of a struct instance is `value₁ ‖ value₂ ‖ … ‖ valueₙ`, i.e. a simple concatenation of the encoding of the member values. Each encoded member value is exactly 32-byte long.

The atomic values are encoded as follows: Boolean `false` and `true` are encoded as `uint256` values `0` and `1` respectively. Addresses are encoded as `uint160`. Integer values are sign-extended to 256-bit and encoded in big endian order. `bytes1` to `bytes31` are arrays with a beginning (index `0`) and an end (index `length - 1`), they are zero-padded at the end to `bytes32` and encoded in beginning to end order.

The dynamic values `bytes` and `string` are encoded as a `keccak256` hash of their contents.

The array values are encoded as the `keccak256` hash of the concatenated `encodeData` of their contents (i.e. the encoding of `SomeType[5]` identical to that of a struct containing five members of type `SomeType`).

The struct values are encoded recursively as `hashStruct(value)`. This is undefined for cyclical data.

### Definition of `domainSeparator`

The domain separator is a 256-bit number that is constant for a given domain, use-case, DApp or implementation by any other name. It serves to disambiguate otherwise identical messages between different domains. So two DApps, both accepting a `Transfer(address to)` message would not accidentally accept each others signed messages.

Besides accidental compatibility, there is also a malicious scenario where an attacker creates a DApp that tricks users into signing a message that will be valid in a different domain than the user intended. By having a trusted user-agent involved in the derivation of the domain separator, this can be partially mitigated. For example, the user-agent can include the current domain name in the domain separator.

Since different domains have different needs, an extensible scheme is used where the DApp specifies a `DomainSeparator` struct type and an instance `domainSeparatorInstance` which it passes to the user-agent. The user-agent can then apply different verification measures depending on the fields that are there.

```
domainSeparator = hashStruct(domainSeparatorInstance)
```

where the type of `domainSeparatorInstance` is a stuct named `DomainSeparator` with one or more of the below fields. The user-agent can reject (i.e. refuse to sign) depending on the `domainSeparatorInstance` object.

* A field `bytes32 salt` will always be accepted.
* A field `string origin` can be rejected if the supplied value does not match the current [`origin`][mdn-origin] as specified in the HTML standard.
* A field `address contract` can be rejected if the user-agent determines the current request is not appropriate for the given contract. This allows the user-agent to implement custom anti-phising for well-known contracts.
* Unrecognized fields can be rejected.
* Future extensions to the standard can add new fields with new constraints.

[mdn-origin]: https://developer.mozilla.org/en-US/docs/Web/API/HTMLHyperlinkElementUtils/origin

Note that a console based user-agent is allowed to accept domain separators using the `domain` field. It has no way to verify the value and needs to trust the user.

DApp implementors should not add non-standard fields, they can always use the `salt` field to implement domain specific extensions.




### Specification of the `eth_signTypedData` JSON RPC

**TODO**: Update.

This EIP proposes a new JSON RPC method to the `eth` namespace: `eth_signTypedData`.

Parameters:

0.  `TypedData` - Typed data to be signed
1.  `Address` - 20 Bytes - Address of the account that will sign the messages

Returns:

0.  `DATA` - signature - 65-byte data in hexadecimal string

Typed data is the array of data entries with their specified type and human-readable name. Below is the [json-schema](http://json-schema.org/) definition for `TypedData` param.

```json-schema
{
  items: {
    properties: {
      name: {type: 'string'},
      type: {type: 'string'}, // Solidity type as described here: https://github.com/ethereum/solidity/blob/93b1cc97022aa01e7daa9816bcc23108bbe008b5/libsolidity/ast/Types.cpp#L182
      value: {
        oneOf: [
          {type: 'string'},
          {type: 'number'},
          {type: 'boolean'},
        ],
      },
    },
    type: 'object',
  },
  type: 'array',
}
```

There also should be a corresponding `personal_signTypedData` method which accepts the password for an account as the last argument.

### Specification of `web3.{eth,personal}.signTypedData`

**TODO**: Write.

```javascript
// Client-side code example
const typedData = [
  {
    'type': 'string',
    'name': 'message',
    'value': 'Hi, Alice!',
  },
  {
    'type': 'uint',
    'name': 'value',
    'value': 42,
  },
];
const signature = await web3.eth.signTypedData(typedData, signerAddress);
// or
const signature = await web3.personal.signTypedData(typedData, signerAddress, '************');
```

```javascript
// Signer code JS example
import * as _ from 'lodash';
import * as ethAbi from 'ethereumjs-abi';

const schema = _.map(typedData, entry => `${entry.type} ${entry.name}`).join(',');
// Will generate `string message,uint value` for the above example
const schemaHash = ethAbi.soliditySHA3(['string'], [schema]);
const data = _.map(typedData, 'value');
const types = _.map(typedData, 'type');
const hash = ethAbi.soliditySHA3(
    ['bytes32', ...types],
    [schemaHash, ...data]
);
```

## Rationale
<!-- The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion. -->

The `encode` function is extended with a new case for the new types. The first byte of the encoding distinguishes the cases. For the same reason it is not safe to start immediately with the domain separator or a `typeHash`. While hard, it may be possible to construct a `typeHash` that also happens to be a prefix of a valid RLP encoded transaction.

The domain separator prevents collision of otherwise identical structures. It is possible that two DApps come up with an identical structure like `Transfer(address from,address to,uint256 amount)` that should not be compatible. By introducing a domain separator the DApp developers are guaranteed that there can be no signature collision.

The domain separator also allows for multiple distinct signatures use-cases on the same struct instance within a given DApp. In the previous example, perhaps signatures from both `from` and `to` are required. By providing two distinct domain separators these signatures can be distinguished from each other.

**Alternative 1**: Use the target contract address as domain separator. This solves the first problem, contracts coming up with identical types, but does not address second use-case. The standard does suggest implementors to use the target contract address where this is appropriate.

The function `hashStruct` starts with a `typeHash` to separate types. By giving different types a different prefix the `encodeData` function only has to be injective for within a given type. It is okay for `encodeData(a)` to equal `encodeData(b)` as long as `typeOf(a)` is not `typeOf(b)`.

### Rationale for `typeHash`

The `typeHash` is designed to turn into a compile time constant in Solidity. For example:

```javascript
bytes32 constant MESSAGE_TYPEHASH = keccak256(
  "Message(address from,address to,string contents)");
```

For the type hash several alternatives where considered and rejected for the reasons:

**Alternative 2**: Use ABIv2 function signatures. `bytes4` is not enough to be collision resistant. Unlike function signatures, there is negligible runtime cost incurred by using longer hashes.

**Alternative 3**: ABIv2 function signatures modified to be 256-bit. While this captures type info, it does not capture any of the semantics other than the function. This is already causing a practical collision between ERC20's and ERC721's `transfer(address,uint256)`, where in the former the `uint256` revers to an amount and the latter to a unique id. In general ABIv2 favors compatibility where a hashing standard should prefer incompatibility.

**Alternative 4**: 256-bit ABIv2 signatures extended with parameter names and struct names. The `Message` example from a above would be encoded as `Message(Person(string name,address wallet) from,Person(string name,address wallet) to,string message)`. This is longer than the proposed solution. And indeed, the length of the string can grow exponentially in the length of the input (consider `struct A{B a;B b;}; struct B {C a;C b;}; …`). It also does not allow a recursive struct type (consider `struct List {uint256 value; List next;}`).

**Alternative 5**: Include natspec documentation. This would include even more semantic information in the schemaHash and further reduces chances of collision. It makes extending and amending documentation a breaking changes, which contradicts common assumptions. It also makes the schemaHash mechanism very verbose.

### Rationale for `encodeData`

The `encodeData` is designed to allow easy implementation of `hashStruct` in Solidity:

```javascript
function hashStruct(Message memory message) pure returns (bytes32 hash) {
    return keccak256(
        MESSAGE_TYPEHASH,
        bytes32(message.from),
        bytes32(message.to),
        keccak256(message.contents)
    );
}
```

it also allows for an efficient in-place implementation in EVM

```javascript
function hashStruct(Message memory message) pure returns (bytes32 hash) {

    // Compute sub-hashes
    bytes32 typeHash = MESSAGE_TYPEHASH;
    bytes32 contentsHash = keccak256(message.contents);

    assembly {
        // Back up select memory
        let temp1 := mload(sub(order, 32))
        let temp2 := mload(add(order, 128))

        // Write typeHash and sub-hashes
        mstore(sub(message, 32), typeHash)
        mstore(add(order, 64), contentsHash)

        // Compute hash
        hash := keccak256(sub(order, 32), 128)

        // Restore memory
        mstore(sub(order, 32), temp1)
        mstore(add(order, 64), temp2)
    }
}
```

The in-place implementation makes strong but reasonable assumptions on the memory layout of structs in memory. Specifically it assume structs are not allocated below address 32, that members are stored in order, that all values are padded to 32-byte boundaries, and that dynamic and reference types are stored as a 32-byte pointers.

**Alternative 6**: Tight packing. This is the default behaviour in Soldity when calling `keccak256` with multiple arguments. It minimizes the number of bytes to be hashed but requires complicated packing instructions in EVM to do so. It does not allow in-place computation.

**Alternative 7**: ABIv2 encoding. Especially with the upcoming `abi.encode` it should be easy to use `abi.encode` as the `encodeData` function. The ABIv2 standard by itself fails the determinism security criteria. There are several valid ABIv2 encodings of the same data. ABIv2 does not allow in-place computation.

**Alternative 8**: Leave `typeHash` out of `hashStruct` and instead combine it with the domain separator. This is more efficient, but then the semantics of the Solidity `keccak256` hash function are not injective.

**Alternative 9**: Support cyclical data structures. The current standard is optimized for tree-like data structures and undefined for cyclical data structures. To support cyclical data a stack containing the path to the current node needs to be maintained and a stack offset substituted when a cycle is detected. This is prohibitively more complex to specify and implement. It also breaks composability where the hashes of the member values are used to construct the hash of the struct (the hash of the member values would dependent on the path). It is possible to extend the standard in a compatible way to define hashes of cyclical data.

Similarly, a straightforward implementation is sub-optimal for directed acyclic graphs. A simple recursion through the members can visit the same node twice. Memoization can optimize this.


## Rationale for `domainSeparator`

A field `string eip719dsl` can added and be rejected if the value does not match the hash of the [EIP-719][eip719] DSL interface string.

[eip719]: https://github.com/ethereum/EIPs/issues/719


## Backwards Compatibility
<!-- All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright. -->

The RPC calls, web3 methods and `SomeStruct.typeHash` parameter are currently undefined. Defining them should not affect the behaviour of existing DApps.

The Solidity expression `keccak256(someInstance)` for an instance `someInstance` of a struct type `SomeStruct` is valid syntax. It currently evaluates to the `keccak256` hash of the memory address of the instance. This behaviour should be considered dangerous. In some scenarios it will appear to work correctly but in others it will fail determinism and/or injectiveness. DApps that depend on the current behaviour should be considered dangerously broken.

## Test Cases
<!-- Test cases for an implementation are mandatory for EIPs that are affecting consensus changes. Other EIPs can choose to include links to test cases if applicable. -->

```
struct Person {
    string name;
    address wallet;
}

struct Message {
    Person from;
    Person to;
    string message;
}
```

Then `schemaHash(Message)` is as follows:

```
bytes32 constant PERSON_SCHEMA_HASH = keccak256(
    "Person(string name,address wallet)"
);

bytes32 schemaHash = keccak256(
    "Message(Person from,Person to,string message)"
    "Person(string name,address wallet)"
  );
```

(Note that the string is split up in substrings for readability. The result is equivalent if all strings are concatenated).

```
struct Order {
    address from;
    address to;
    uint128 amount;
    uint64 timestamp;
    string message;
}

function dataHash(Order order) returns (bytes32) {
    return keccak256(
        bytes32(order.from),
        bytes32(order.to),
        bytes32(order.amount),
        bytes32(order.timestamp),
        keccak256(order.message)
    );
}
```

```
struct Person {
    string name;
    address wallet;
}

struct Message {
    Person from;
    Person to;
    string message;
}

function dataHash(Person person) returns (bytes32) {
    return keccak256(
        keccak256(person.name),
        bytes32(person.wallet),
    );
}

function dataHash(Message message) returns (bytes32) {
    return keccak256(
        dataHash(message.from),
        dataHash(message.to),
        keccak256(message.message)
    );
}
```

## Implementation
<!-- The implementations must be completed before any EIP is given status "Final", but it need not be completed before the EIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details. -->

To be done before this EIP can be considered accepted:

*   [ ] Finalize specification
  *   [ ] Domain separators
  *   [ ] Prevent replay attacks
*   [ ] Add test vectors
*   [ ] Review specification

To be done before this EIP can be considered "Final":

*   [ ] Implement `eth_signTypedData` in major RPC providers.
*   [ ] Implement `web3.sign` in Web3 providers.
*   [ ] Implement `keccak256` struct hashing in Solidity.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
