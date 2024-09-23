# What is Vault Core
## Vault Core
- Vault Core API: interact with external applications, it does authentication, smart contracts, simulation, role management...
- Postings: keeps track of all movement of funds within Vault Core, credit, debit...
- Streaming: real-time streaming with Kafka, events like accounting, posting events, customer change ...
- Vault Core has 3 main functionalities:
    - Implement bank's portfolio: saving, credit accounts, mortgages ...
    - Maintain customer accounts
    - Generate and record postings in and out of the accounts
- Vault Core Ledger records the postings once they're accepted into Vault Core, this is a common platform that any banks can use
- Smart Contracts are code modules, implemented in Product Engine, allows to reuse code, customize ...
## Vault Core Ledger
- Rich: Capture multiple partitions of an account, different denomincations, assets, monitor and maximize/minimize interest
- Expressive: Can slice and dice balances into any structure, no limit transparancy, personalized reporting
- Real-time: Stream events, monitor behaviors and maximize/minimize interest
- Can manage any line of funds, currencies, coins or reward points
## Account
- There are 2 types of accounts: Customer account and internal account
- Customer accounts always have specific Smart Contract associated with them and a set of parameters. The logic inside smart contract can use the parameters and balances to reject/accept postings and generate postings
- Internal accounts are internal accounts of the bank, it shows how much money the bank owns
- It's used on the Vault ledger to meet accounting requirements
- Internal accounts don't use smart contracts linked Customer accounts with any metadata, it only uses a blank Smart contract to maximize performance.
- Internal accounts are not associated with a customer entity, they're owned by the bank itself
- Account has 5 statuses
    - Pending
    - Cancelled
    - Open
    - Pending closure
    - Close
- When an account is created, it has to be associated with a product with a product_id, however the product_version_id can be updated
- The field `instance_param_vals` or `expected_parameters` contains the parameters of the instance
- `stakeholder_ids` is the list of all stakeholders of an account
- Restrictions can be imposed on each account such as `RESTRICTION_TYPE_PREVENT_ACCOUNT_CREATION` or `RESTRICTION_TYPE_PREVENT_UPDATES`
- Any dynamic data can be stored in field `details` 
## Balances
- Balances are stored against 4 dimensions: Asset type, denomination, address and phase
- `Address`: An account always has at least 1 address, each address has a balance, address is the way to segregate an account to use for different purposes like for saving or interest fee
- `Asset type`: Commercial bank money, cash or reward points
- `Denonimation`: ISO currencies or coins...
- `Phase`: Pending incoming, Pending outgoing, Committed
## Postings
- `Postings` are movements of fund, each posting is either a debit or credit amount
- `Postings` have to be specified in all 4 dimensions
- A set of balanced postings makes a `Posting Instruction`
- There are 9 types of `Posting Instruction`: 
    - Inbound authorization (c)
    - Outbound authorization (c) 
    - Adjustment authorization (c)
    - Settlement (c)
    - Hard inbound settlement (s)
    - Hard outbound settlement (s)
    - Transfer (s)
    - Release (c)
    - Custom
- A set of `Posting Instructions` makes a `Posting Instruction Batch`, if a `Posting` is rejected then the whole batch is rejected
- It's recommended to use the default posting tyeps instead of `Custom` because Custom instructions don't have financial meaning in Vault
## Payment Devices
- Each `Payment Device` has an ID and is created by upper systems and not associated with an account
- Each `Payment Device Link` also has an ID and is associated with an account, used to initiate postings
- A `Payment Device` can be linked to multiple `Accounts`, an `Account` can be linked to multiple `Payment Devices`
- Each Payment Device Link for an account has a Payment Device Token which is a hash value from IBAN, account number...
- Multiple `Payment Device Linked` can be associated to an `Account` and share 1 `Payment Device Token` but only 1 of them is `active`, the rest is `inactive`

## Smart Contract
- It's python code that is run by Vault to operate financial products like saving accounts, lending products or current account

## Smart Contract Parameters
- `Global level`: shared by all `smart contracts`
- `Template level`: shared by all instances of a `smart contract`
- `Instance level`: owned by the product or account
- `Expected Parameters` syntax are used to reference to parameters created via `Parameter API`

## Smart Contract Hooks
- Hooks are Python functions that are called by Vault at different times throughout the lifecycle of smart contract.
- `activation_hook`: when a new account is activated, used to set scheduled events and perform initial money movements
- `pre_posting_hook`: when a posting is about to enter the ledger, used to determine if the posting is accepted or rejected
- `post_posting_hook`: when a posting has been appended to the ledger, used to move additional funds or generate events
- `scheduled_event_hook`: when a scheduled event runs, used to perform periodic actions such as calculating accrued interest
- `pre_parameter_change_hook`: when an *INSTANCE* level parameter or future-dated `ExpectedParameter` is going to be updated, even if its value is the same as the previous. It's not updated if a parameter value is backdated
- `post_parameter_change_hook`: when an *INSTANCE* level parameter or future-dated `ExpectedParameter` has been updated. It's not triggered if backdated
- `derived_parameter_hook`: when a derived parameter is requested, derived parameters are calculated from other parameters
- `conversion_hook`: befor ethe account is converted to a newer Smart Contract version
- `deactivation_hook`: when an account is closed

## Hot Path
- Hot paths refer to chains of events that happens inside Vault as the result of external systems to move money in or out of the bank
- From `pre_posting_hook` => Ledger is considered hot path because it can reject or accept the postings, payment providers always expect the postings to be accepted or rejected as soon as possible so `pre_posting_hook` has to limit the number of data calls and only validate
- Vault Core always ensure that pre hooks and post hooks always happen in order

## Vault Object
- Each hook takes in 2 arguments: `vault` and `hook_parameters`
- The vault object allows to access Vault data available to the hooks

## Float
- It's not recommended to use `float` as parameters to call API
- It's better to use `int` or `Decimal`

## Schedules
- Schedules are defined when the account is first activated
- Schedules can be altered using the logic in the contract when it's active
- The scheduler is a time-based Kafka producer that periodically sends trigger messages to topics to trigger the consumer
- These messages are called `Schedule jobs`, they are created within the context of `Scheduler clients` such as `Smart Contracts` and `Workflows`
- The status of a `Scheduled job` is either `PUBLISHED`, `SUCCEEDED` or `FAILED`
- Schedued jobs have `scheduled_timestamp` is the time it's supposed to run and `published_timestamp` when it actually runs
- When a job is published, its status is recorded to the database
- If a scheduled job fails, it can be republished
## Schedule Tags
- When a scheduled job finishes, it will publish messages to Kafka for consumers, these messages are called `Tags`
- Each tag contains ID, status, report, description, timestamp...
- Collection of all schedules with the same tag is called `Scheduled operation`

## Schedule Group
- `Group Schedules` are schedules that are in a `Schedule Group`
- These `Group Schedules` publish `Scheduled Jobs` in an order, even if it's time to run, they can be still blocked by the previous schedule

