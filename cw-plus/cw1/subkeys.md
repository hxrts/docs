---
order: 2
---

# Subkeys

[CW1 Subkeys](https://github.com/CosmWasm/cosmwasm-plus/tree/master/contracts/cw1-subkeys)
is inspired by [Cosmos SDK feature proposal](https://forum.cosmos.network/t/proposal-adding-subkey-feature-to-cosmos-sdk-and-apply-it-to-the-hub/2358).

This is a basic proxy contract. Initiated contract with some tokens inside,
allowed accounts can execute `CosmosMsg::Bank(BankMsg::Send{})` within the
allowance(`spender, denom`) limits per account basis. Allowance logic is similar
to [CW20](../cw20/spec). After authorization their allowance will be reduced, and
the send message will be relayed. If they don't have sufficient authorization,
or if they try to proxy any other message type, then the attempt will be rejected.
Admin can add an allowance for an account during init msg broadcast or after
init.

The contract consists permissioning logic that allows and disallows specified keys to execute
specific messages. For now these messages are staking messages (covers _Delegate, Undelegate, Redelegate, Withdraw_ for now),
but it is just few lines of code to add new message types. Allowance and permission checks works separately, meaning a subkey can have allowance to spend tokens,
but no permissioned message execution and vice versa.

## Contract demo

First, initialize node repl:

```shell
npx @cosmjs/cli@^0.22 --init https://raw.githubusercontent.com/CosmWasm/cosmwasm-plus/v0.2.1/contracts/cw1-subkeys/helpers.ts
```

::: warning
Helper code is compatible with cw1-subkeys smart contract version **v0.2.1**
:::

Load wallet:

```ts
const client = await useOptions(coralnetOptions).setup(PASSWORD);
const factory = CW1(client);
```

Upload the code and the contract:

```ts
// upload using code below
// if the code is already uploaded use code id to initiate
const codeId = await factory.upload()
// contract is already uploaded on coralnet: codeId -> 12
const { address } = await client.getAccount()
const contract = await factory.instantiate(12, { admins: [address], mutable: true}, "My Gift to a Friend")

// print out contract.contractAddress for later
contract.contractAddress
// address -> 'coral1267wq2zk22kt5juypdczw3k4wxhc4z47mug9fd'
```

We created a contract from a code with only `address` as admin. Update admins
for demonstration.

```ts
// Use a key you control to test out execute with subkey
const friendAddr = "coral1xx79l5q32eqvkk3hc54k92dpq909zh652gw70v"

// generate second address if you don't have one:
// const friendClient = await useOptions(coralnetOptions).setup(PASSWORD, KEY_FILE);
// const friendAddr = await friendClient.getAccount().then(x => x.address);

contract.updateAdmins([address, friendAddr]);
```

After the last line, two admins have control over the sub key master contract.
You can see the new admin added by running `contract.admins()`
Let's delete friends address from admins, you would not want him to
run away with the funds. Remove his address from admins and freeze the contract.
Freezing means admins cannot be modified afterwards.

```ts
contract.updateAdmins([address])
contract.freeze()
```

### Allowance

Let's give some allowance to your friends account, so he can buy a lambo:

```ts
// lets pour some money to the account
client.sendTokens(contractAddress, [{denom: "ushell", amount: "1000000"}])
// verify tokens are transferred
client.getAccount(contractAddress)

contract.increaseAllowance(friendAddr, {denom: "ushell", amount: "90000"})
contract.allowance(friendAddr)
```

Now test if he can execute the message. Open another terminal screen:

```ts
const friendClient = await useOptions(coralnetOptions).setup(PASSWORD, KEY_FILE);
const factory = CW1(friendClient)
const contract = factory.use('coral1267wq2zk22kt5juypdczw3k4wxhc4z47mug9fd')

contract.execute([{bank: {send: {from_address: contractAddress, to_address: address, amount: [{denom: "ushell", amount: "20000"}]}}}])
```

Allowed account can spend the tokens. Lets prank your friend with decreasing
his allowance on admin terminal:

```ts
contract.decreaseAllowance(randomAddress, {denom: "ushell", amount: "69999"}, { at_height: { height: 40000}})
```

After these operations he will only have _1 ushell_ to spend. The prank's
best part is `at_height` field. After height 40000 his allowance will become
inactive meaning he can't spend the tokens anymore.

### Permissions

Initially keys do not have any permission to send permissioned messages and needs to be set key basis:

```ts
let permissions: Permissions = { delegate: true, undelegate: false, redelegate: true, withdraw: true}

contract.setStakingPermissions(randomAddress, permissions)

let dmsg: DelegateMsg = {staking: {delegate: {validator:"coralvaloper1hf50trj7plz2sd8cmcvn7c8ruh3tjhc2uch4gp", amount:{denom:"ureef",amount:"999"}}}}
contract.execute([dmsg])
// will be approved

let unmsg: UndelegateMsg = {staking: {undelegate: {validator:"coralvaloper1hf50trj7plz2sd8cmcvn7c8ruh3tjhc2uch4gp", amount:{denom:"ureef",amount:"999"}}}}
contract.execute([unmsg])
// will be rejected

```

## Contribution

This contracts logic can be improved by adding different message types,
various permissions for message types, daily spendable amount etc. Check
[the cosmos sdk proposal](https://forum.cosmos.network/t/proposal-adding-subkey-feature-to-cosmos-sdk-and-apply-it-to-the-hub/2358) for
more ideas.