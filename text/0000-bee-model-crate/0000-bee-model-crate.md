+ Feature name: `bee-transaction-crate`
+ Start date: 2019-09-06
+ RFC PR: [iotaledger/bee-rfcs#3](https://github.com/iotaledger/bee-rfcs/pull/3)
+ Bee issue: [iotaledger/bee#43](https://github.com/iotaledger/bee/issues/43)

# Summary

IOTA is a distributed ledger that was designed for payment settlement and data
transfer between machines and devices in the Internet of Things (IoT)
ecosystem. This RFC introduces a `Transaction` type to represent transactions
sent through the IOTA network, and proposes a way to construct it in an
idiomatic way.

# Motivation

+ Payment settlements or data transfers can be done with the help of these transactions. 
+ ...

# Detailed design

## General

A transaction has a total size of 8019 trits and consists of 15 fields of
constant length. Their names, purpose, and size (in units of trit) are
summarized in the table below:

| Name                              | Description                                                         | Size (in trits) |
| --                                | --                                                                  | --              |
| `signature_or_message_fragment`   | contains the signature of the transfer or user-defined message data | 6561            |
| `address`                         | receiver (output) if value > 0, or sender (input) if value < 0      | 243             |
| `value`                           | the transferred amount in IOTA                                      | 81              |
| `obsolete_tag`                    | another arbitrary user-defined tag                                  | 81              |
| `timestamp`                       | the time when the transaction was issued                            | 27              |
| `current_index`                   | the position of the transaction in its bundle                       | 27              |
| `last_index`                      | the index of the last transaction in the bundle                     | 27              |
| `bundle`                          | the hash of the bundle essence                                      | 243             |
| `trunk`                           | the hash of the first transaction referenced/approved               | 243             |
| `branch`                          | the hash of the second transaction referenced/approved              | 243             |
| `tag`                             | arbitrary user-defined value                                        | 81              |
| `attachment_timestamp`            | the timestamp for when Proof-of-Work is completed                   | 27              |
| `attachment_timestamp_lowerbound` | *not specified*                                                     | 27              |
| `attachment_timestamp_upperbound` | *not specified*                                                     | 27              |
| `nonce`                           | is the Proof-of-Work nonce of the transaction                       | 81              |

Transactions should be immutable after construction and are thus only accessible through getter methods.

This RFC introduces the `Transaction`, `UnprovenTransaction`, and
`UnprovenTransactionBuilder` types. A `Transaction` is a struct with all fields
as defined in the above table. An `UnprovenTransaction` shares all fields with
`Transactions` with the exception of `nonce`. An `UnprovenTransaction` is built
via the builder pattern using `UnprovenTransactionBuilder`.

The flow of types is as follows:

```
UnprovenTransactionBuilder -> UnprovenTransactionBuilder::build -> UnprovenTransaction

UnprovenTransaction -> UnprovenTransaction::compute_nonce -> Transaction

UnprovenTransaction -> UnprovenTransaction::set_nonce -> Transaction
```

The reasons for constructing the `Transaction` through an intermediary type (instead of directly
going `TransactionBuilder -> Transaction`) is for the following 4 reasons:

1. the proof of work algorithm acts on the data in `UnprovenTransaction` without the `nonce`;
2. different proof of work algorithms might be used;
3. the `nonce` can be set from storage if it was previously calculated and stored;
4. the fields of a transaction should be verified and checked indepedently of the nonce to feed it
into the proof of work algorithm (first point).

## `Transaction` and `UnprovenTransaction`

A `Transaction` is defined as an `UnprovenTransaction` transaction plus a `nonce` field.
We decide to do this separation, because:

The fields of `UnprovenTransaction` struct defined below are in the same order as
they appear in the wire format:

```rust
struct UnprovenTransaction {
    signature_fragments: String,
    address: String,
    value: i64,
    obsolete_tag: String,
    timestamp: u64,
    current_index: usize,
    last_index: usize,
    bundle_hash: String,
    trunk: String,
    branch: String,
    tag: String,
    attachment_timestamp: u64,
    attachment_timestamp_lower_bound: u64,
    attachment_timestamp_upper_bound: u64,
}
```

```rust
struct Transaction {
    transaction: UnprovenTransaction,
    nonce: String,
}
```

### `UnprovenTransactionBuilder`

An `UnprovenTransaction` is constructed via the builder pattern using an `UnprovenTransactionBuilder`.

```rust
#[derive(Default)]
pub struct UnprovenTransactionBuilder {
    signature_fragments: Option<String>,
    address: Option<String>,
    value: Option<i64>,
    obsolete_tag: Option<String>,
    current_index: Option<usize>,
    last_index: Option<usize>,
    bundle_hash: Option<String>,
    trunk: Option<String>,
    branch: Option<String>,
    tag: Option<String>,
    attachment_timestamp: Option<u64>,
    attachment_timestamp_lower_bound: Option<u64>,
    attachment_timestamp_upper_bound: Option<u64>,
}
```

The fields are set on `UnprovenTransactionBuilder` through setter methods.
`UnprovenTransactionBuilder::build` builds the `UnprovenTransaction` and
verifies that all required fields are set and uphold their respective
invariants.

```rust
impl UnprovenTransactionBuilder {
    // Similar for the other fields; can add convenience setters for converting from other types.
    pub signature_fragments(&mut self, signature_fragments: String) -> &mut Self {
        self.signature_fragments.replace(signature_fragments);
        self
    }

    pub build(self) -> Result<UnprovenTransaction, UnprovenTransactionBuilderError> {
        unimplemented!()
    }
}
```

### Constructing a `Transaction` from an `UnprovenTransaction`

A `Transaction` can be constructed from an `UnprovenTransaction` by either
computing the `nonce` via Proof-of-Work from the fields of
`UnprovenTransaction`, or it can be set directly by recalling the `nonce` from
memory if it was already computed and stored before. We use the type `Nonce`
here without further specifying it and leave it as an implementation detail.
Similary, we don't specify the proof of work algorithm any further and leave
it as a function from a stream of bytes to a `Nonce`, `f: &[u8] -> Nonce`.

```rust
impl UnprovenTransaction {
    pub fn calculate_nonce<F>(&self, pow_algorithm: F) -> Nonce
    where
        F: Fn(&[u8]) -> Nonce
    {
        let bytes: Vec<u8> = self::serialize_to_byte_stream();
        pow_algorithm(&bytes)
    }

    pub prove_transaction<F>(self, pow_algorithm: F) -> Transaction
    where
        f: Fn(&[u8]) -> Nonce
    {
        let nonce = self.calculate_nonce<F>(pow_algorithm);
        self.set_nonce(nonce)
    }

    pub set_nonce(self, nonce: Nonce) -> Transaction {
        Transaction {
            unproven_transaction: self,
            nonce: nonce,
        }
    }
}
```

As reflected by the code above, the Transaction struct (that is to be returned)
gets initialized directly in PoW. This has some advantages, like the
*timestamp* and the *nonce* can directly be passed to the struct.
A disadvantage is, the building of transactions takes place in the PoW model.

# Drawbacks

Without any bee-model crate, nodes can not exchange transactions. Therefore
this crate seems necessary.

# Rationale and alternatives

- The distinction between Transaction and Transaction Builder as well as Bundle
  and Bundle Builder makes the code cleaner. Properties are clearly assigned to
  specific data objects and not mixed up.
- The proposed crate interface is relatively intuitive. Completely different
  alternatives did not naturally come to mind.
- The validation logic could be done in setter functions instead of the
  separate validator class.
- This kind of interface is relatively minimal and easily extended upon in
  a future iteration.

# Unresolved questions

- How to handle deserialization of the incoming, encoded transaction?
- Where should the actual Transaction struct be initialized/returned? Current
  ideas are to put it in the build() of the TransactionBuilder or as it
  currently is in the compute() of Pow.
- Where should Transaction fields get validated? One-time validation once
  build() is called by the separate *TransactionBuilderValidator* as it
  currently is, or use **setters** in the TransactionBuilder and validate in
  each setter separately?
- Should we use setters for the TransactionBuilder?
- Introducing a fluent API to the TransactionBuilder?
