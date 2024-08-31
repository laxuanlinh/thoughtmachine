# Vault Core fundamentals

## Legacy banking platforms
- Old system for just some products
- More functionalities, reporting, ...
- Duplication, disjoined data, no real-time reporting
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

## Vault Core Product Engine
- Smart contracts allow any products to be implemented
- Can replicate existing products for migration
- Can use SDK to build or use existing products in product library (generic product, saving account...)
- Built-in testing framework to unit test, mock and simulate.

## Vault Core Ledger
- Rich: Capture multiple partitions of an account, different denomincations, assets, monitor and maximize/minimize interest
- Expressive: Can slice and dice balances into any structure, no limit transparancy, personalized reporting
- Real-time: Stream events, monitor behaviors and maximize/minimize interest
- Can manage any line of funds, currencies, coins or reward points

## Developing Banking Products
- Written in subset of Python, use common functionalities via Contracts API
- Can look into the code the see how it caculcate interest rate ..etc...
- Can be configured to be executed automatically via life cycle hooks when a customer does something
- Can pass parameters

## Smart Contract Hooks
- Contract events: when execute or upgrade a contract on an account for example when opening and closing accounts
    - Post-activate
    - Upgrade
    - Close code.
- Scheduled events: configurable via Smart Contract
    - Execution schedules: returns set of events, schedules that they run on
    - Scheduled code: returns when a scheduled event is executed, for example recurring payments
- Posting events: when an event is posted into the Ledger
    - Pre-posting: before the posting, determines whether the event is accepted into the Ledger
    - Post-posting: runs after an event is posted successfully
    - Pre-parameter: runs everytime there is a request to update the instance level parameters
    - Post-parameter change: run when parameters are updated.
- Parameter events

## Vault Core Integration
- Responsibilities:
    - **Manufactering products** with Smart Contracts, writing python code and deploy to Vault Core
    - **Managing accounts**, every time a customer opens a new account, a Smart Contract is executed to determine the parameters and operations of this account
    - **Postings**: either via Postings API or the Smart Contracts can generate events themselves. The Ledger records these events
    - **Managing customers**: *Vault Core cannot handle PII* so these information has to be stored some where else. The holding of a customer in Vault Core is linked to this information via a *unique ID*.

- Other functionalities within a bank:
    - **KYC and AML**: verify identities and avoid risky customers
    - **Cards and payments**: make and receive payments from other banks
    - **Fraud monitoring**
    - **Credit scoring and credit risks**: who to lend to and how much
    - **Data reporting and analysis**
