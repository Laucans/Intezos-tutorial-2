# Intezos P2

This tutorial will be focus on how to deal with your storage with bigmap.

## Goal 
Improve https://github.com/Laucans/Intezos-tutorial-1 by adding a transfer entry instead of creating a new contract for a new transfer.
It'll impact the way to manage the storage and so others entrypoints

That's better because : 
- Origination fees are paid only once.
- Easier to integrate your contract into dApp with only one contract address
- Only one contract to monitor

## Technologies

Contract : Using Ligo

## You will learn
- Work with `big_map`
- Split your code with `import` and understand difference with `include`
- Work with `timestamp`

# 1. Pre-requisite.
- Be sure you followed or understand concepts of https://github.com/Laucans/Intezos-tutorial-1 
- If you don't have the code, run `cp initial_contracts contracts` into your terminal.
- Be sure to use `dry-run` and others command see in the precedent tutorial. You can get the `contracts_final/intezos.runner.jsligo` file if you want some `parameters` and `storage` instance. 
---
## Update the storage to store multiple transfers

Start by extracting your current storage which represent a transfer into a new type named `transfer` : 

```typescript
type transfer = {
  amount: tez, // The transfered amount (xtz)
  sender: address, // Address of the sender (tzxxx)
  receiver: address, // Address of the recipient (tzxxx)
  secret, // Type secret
  pending: bool // Is the has been already claimed or redeem ?
};
```
And your storage have to mutate contain multiple `transfer`

```typescript
type storage = list<transfer>
```
:warning: Here we are using a `list` which is common into others languages. That's bad in blockchain ecosystem. 
Remember you pay for your execution, at the startup of your contract call, the storage is loaded, load a big `list` can consume a lot of `gas` and so `fees` can be expensive for nothing. Worse, an attacker can growth your `list` and sature memory of the contract making it unusable.

To avoid it Ligo has `Bigmap` lazily loaded. From [ligolang.org](https://ligolang.org) : 
> A lazily deserialised map that is intended to store large amounts of
data. Here, "lazily" means that storage is read or written per key on
demand. Therefore there are no `map`, `fold`, and `iter` operations as
there are in `Map`.

Because it is a `map`, an unique key will be necessary, couple `sender` `receiver` is a good candidate, but not unique, to ensure unicity we will introduce the `timestamp` of the transfer :
```typescript
type sender = address;

type receiver = address;

type transfer_key = {
  sender,
  receiver,
  transfer_timestamp: timestamp
};

type storage = {
  transfer_ledger: big_map<transfer_key, transfer>,
};
```

## Create a types file

Our `intezos.jsligo` start to be a big fuzzy file containing our contract entrypoints and our datatype, it's time to split it to keep our code clean. 

Create a new file named `storage.types.jsligo` and move your storage types into it. 
I like to keep my parameters type near to my entrypoints but it is personnal preference. 

Now you need to import your new file in your `intezos.jsligo`. 
One way like used before is to use :
```typescript
#include "storage.types.jsligo"
```
:warning: This is a bad way to do it, `#include` will literally copy your file and paste it into your current file. Code collision (variable or function which has been defined twice) is possible. 

Prefer 
```typescript
#import "storage.types.jsligo" "Storage"
```
It will surround your file with a namespace `Storage` to avoid collision. Now to use your type definition, you'll have to prefix it with `Storage`.  
Update your signatures like this to fix your errors : 
```typescript 
const claim = (parameter: claim_parameter, store: Storage.t): [list<operation>, Storage.t]
```

> For you information, each `#` instructions are executed by ligo preprocessor. You can see it like a script which is doing operation onto your files before to pass it through compiler.

## Create the transfer entrypoint

Because our storage has been mutated, we have to redo all our entries. 
We will do the `Transfer` one together, **comments other entries for now**.

### Verification

When you are doing a transfer operation first step is to verify request parameters are compliant

Get the amount sent by the user with [Tezos.get_amount()](https://ligolang.org/docs/reference/current-reference?lang=jsligo) from stdlib.
```typescript
  const transaction_amount = Tezos.get_amount();
```

Then verify that the `amount` is valid, and parameters are filled.
```typescript
  assert_with_error((transaction_amount > (0 as mutez)), "You have to send tokens to initiate a transaction");
  assert_with_error(
    (parameter.secret.question != "" && parameter.secret.encrypted_answer != Bytes.pack("")),
    "Question/answer are mandatory"
  );
```
Check if the contract exist and is a wallet, if not emit an error.  
`type contract<unit>` is how wallet can be identified in Ligo

```typescript
  const _wallet : contract<unit> = Tezos.get_contract_with_error(address, "Not an existing address");
```

### Update storage
We want to generate the new entry which gonna be inserted onto the ledger. Because it is a map we need a key and a value. To instantiate the timestamp use `Tezos.get_now()` which return the current timestamp of the chain.

```typescript
  const transfer_key: Storage.transfer_key =
    { sender: Tezos.get_source(), receiver: parameter.receiver, transfer_timestamp: Tezos.get_now() };

  const transfer: Storage.transfer =
    { amount: transaction_amount, secret: parameter.secret, reason: parameter.reason, pending: true };
```
And then update your `Big_map` by using [Big_map.update](https://ligolang.org/docs/reference/big-map-reference?lang=jsligo) before to return the mutated instance. 

This is how to use the `update` function :   
`Big_map(key_of_the_entry_to_update, Some(New_value) | None(), bigmap_to_update)`

Note: when `None` is used as a value, the value is removed from the `big_map`.

```typescript
  const updated_ledger =
    Big_map.update(
      transfer_key,
      Some(transfer),
      store.transfer_ledger
    );

  return [
    list([]) as list<operation>,
    { transfer_ledger: updated_ledger }
  ]
```


## Create the claim entrypoint

First step is to retrieve the corresponding entry from your ledger, to do it, you will need to build the searched key from parameters.

The type of your parameter need to be updated :
```typescript
export type claim_parameter = {
  sender: Storage.sender,
  transfer_timestamp: timestamp,
  answer: bytes
};
```
And in your `claim` function let's build the key :
```typescript
  // Define the key that you want to retrieve
  const transfer_to_claim_key =
    { sender: parameter.sender, receiver: Tezos.get_source(), transfer_timestamp: parameter.transfer_timestamp };
```
Then you will need to search for the key onto your map. For it I create a function onto a new `utils.jsligo` file which gonna use `Big_map.find_opt(key, Big_map)` to search the transfer into the ledger :
```typescript 
export const get_transfer_from_storage = (key: Storage.transfer_key, store: Storage.t) => {
  return match(Big_map.find_opt(key, store.transfer_ledger))
    {
      when(None()) : failwith("Transfer doesn't exist")
      when(Some (transfer)): transfer
    }
};
```
And I use it in my `claim` function. Like seen before add this code on top of your `intezos.jsligo` file : 
```typescript
#import "./utils.jsligo" "Utils" 
```
Then call it with `Utils.get_transfer_from_storage` :
```typescript
  const transfer_to_claim = Utils.get_transfer_from_storage(transfer_to_claim_key, store);
```


Like for transfer, we will do verification step : 
```typescript
  assert_with_error(
    transfer_to_claim.pending && Utils.encrypt(parameter.answer) == transfer_to_claim.secret.encrypted_answer, "Failed to claim the transfer"
  );
```

Last step is create the tezos `transaction` which gonna be returned to the chain and update your storage, like in `transfer` entry.
```typescript
  const transferOperation: operation =
    Tezos.transaction(unit, transfer_to_claim.amount, Tezos.get_contract_with_error(Tezos.get_source(), "Not an existing address"));
  const updated_ledger =
    Big_map.update(
      transfer_to_claim_key,
      Some({ ...transfer_to_claim, pending: false }),
      store.transfer_ledger
    );
  return [list([transferOperation]), { transfer_ledger: updated_ledger }]
```

## Create the redeem entrypoint

The redeem doesn't introduce new concepts you can try to do it alone or check one solution in `contracts_final` folder.

# Next 
Congrats ! The next step will be to integrate it into your first dApp
