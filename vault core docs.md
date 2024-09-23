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
- There are 2 types of accounts