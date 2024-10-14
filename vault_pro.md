# Smart Contract Performance
## Overview
- Simulation is better then Unit testing to test performance of `Smart Contract`
- Consider time complexity, use appropriate data structure for fast access, avoid recursions, break for loop early, use cache for API calls...
- Break code into smaller helper functions for better visibility
## Data requirement
- Hook can retrieve data multiple times and data requirement may cost performance, for example 1 year of postings
- Consider data you need for optimal data requirement
```python
# too expensive
@requires(balances="1 month live")

# much less expensive
@requires(balances="latest live")
def post_posting(vault, effective_data)
    balanaces = vault.get_balance_timeseries().latest()
```
- Only retrieve data that is absolutely necessary
```python
# unnecessarry
@requires(balances="2 months")

# just enough
@requires(balances="33 days")
```

- Code that takes too long might be `TIMEOUT` and won't be able to rerun, it also renders the accounts `unusable`
- Code that uses too much CPU can also throttle and `TIMEOUT`

## Building Smart Contract for peak performance
### Data Fetching
- Optimizing data fetchers will help reduce the amount of data fetched to the `Smart Contracts`.
- This is usually done during dev/improvement phase on the existing `Smart Contracts` by the writers
- We can do this by using `Optimized data fetchers` to fetch data points, `observation` instead of full `timeseries`
- `Optimized data fetcher` is available starting from version 3.10, it allows define a specific window or a single data point in the past
- However it's only able to fetch data about `postings`, `balances` and `observation data`
- We can use `@fetch_account_data` decorator along with `@requires` to define data fetchers in the metadata
```python
data_fetchers = [
    BalancesIntervalFetcher(
        fetcher_id = "live_balances",
        # EFFECTIVE_DATE is the date the hook is set to be executed
        start = DefinedDateTime.EFFECTIVE_DATE,
        # LIVE is when the hook is actually executed
        end = DefinedDateTime.LIVE
    )
]
@fetch_account_data(event_type="END_OF_DAY", balances=["live_balances"])
def scheduled_code(event_type, effective_date):
    end_of_day_datetime = get_end_of_day_date_time(vault, effective_date)
    balances = vault.get_balance_timeseries(fetcher_id = "live_balances").latest()
```
- In the previous example, we fetch the data between the `start` and `end` date which is why we use `BalancesIntervalFetcher`.
- However if we want to take a snapshot in the past then we need to use `BalancesObservationFetcher` instead
```python
data_fetchers = [
    BalancesObservationFetcher(
        fetcher_id = "snapshot_balances_last_month",
        at = RelativeDateTime(
            shift=Shift(months=-1),
            origin=DefinedDateTime.EFFECTIVE_DATE
        )
    )
]
```
- `BalancesObservationFetcher` costs less performance than `BalancesIntervalFetcher` because it doesn't have to fetch data from a timeserie
- Similarly we can define a `PostingIntervalFetcher`
```python
data_fetchers = [
    PostingIntervalFetcher(
        fetcher_id = "posting_one_day_one_month_ago",
        start = RelativeDateTime(
            origin=DefinedDateTime.EFFECTIVE_DATE,
            shift=Shift(months=-1)
        ),
        end = RelativeDateTime(
            origin=DefinedDateTime.INTERVAL_START,
            shift=Shift(day=1)
        )
    )
]
```
- We can use multiple data fetchers like so
```python
@fetch_account_data(
    balances = ["live_balances", "snapshot_balances_last_month"],
    postings = ["posting_one_day_one_month_ago"]
)
def post_posting_code (postings, effective_date):
    live_balances = vault.get_balance_timeseries(fetcher_id = "live_balances").latest()
    balance_last_month = vault.get_balance_observation(fetcher_id = "snapshot_balances_last_month")
    posting_one_day_one_month_ago = vault.get_postings(fetcher_id = "posting_one_day_one_month_ago")
```
- When using `@fetch_account_data` in conjuction with `@requires`, you're limited to 1 `keyword` per decorator
```python
# not allowed
@fetch_account_data(postings = ["posting_one_day_one_month_ago"])
@requires(balances=["1 days"], postings = ["3 days"])

# only 1 type per decorator is allowed
@fetch_account_data(postings = ["posting_one_day_one_month_ago"])
@requires(balances=["1 days"])
```
- Additionally, both decorators can have the same `event_type`, different `keywords` or same `keyword`, different `event_types`
```python
# allowed, same event_type, diff keywords
@fetch_account_data(event_type="EVENT_TYPE_1", postings = ["posting_one_day_one_month_ago"])
@requires(event_type="EVENT_TYPE_1", balances=["1 days"])

# allowed, same keyword, diff event_types
@fetch_account_data(event_type="EVENT_TYPE_1", postings = ["posting_one_day_one_month_ago"])
@requires(event_type="EVENT_TYPE_2", postings=["1 days"])

# not allowed, same keyword and same event_type
@fetch_account_data(event_type="EVENT_TYPE_1", postings = ["posting_one_day_one_month_ago"])
@requires(event_type="EVENT_TYPE_1", postings=["1 days"])
```
- @fetch_account_data can be used along with @requires with parameters=True as well
```python
@fetch_account_data(
    balances = ["live_balances", "snapshot_balances_last_month"],
    postings = ["posting_one_day_one_month_ago"])
@requires(parameters=True)
```
- In Documentation Hub, under each Hook is a section called Allowed Data Fetchers which specifies which fetchers are allowed, for example for `deactivation_hook`, only `postings` and `balances` are allowed

## Performance impact of Directives
- Avoid instructing multiple `Posting Instruction Batches` because it will lose atomicity and performance
- For example, this is ok:
```python
posting_instructions = []
if fee_related_instructions:
    posting_instructions.extend(fee_related_instructions)
if interest_related_instructions:
    posting_instructions.extend(interest_related_instructions)
return PostPostingHookResult(
    posting_instructions_directives = [
        PostingInstructionDirective(
            posting_instructions = posting_instructions
        )
    ]
)
```
- This is not ok because it's using multiple `instruct_posting_batch`:
```python
if some_instructions:
    vault.instruct_posting_batch(
        posting_instructions = some_instrutions,
        effective_date = effective_date,
        client_batch_id = f'BATCH_0{vault.get_hook_execution_id()}'
    )
if some_other_instructions:
    vault.instruct_posting_batch(
        posting_instructions = some_other_instructions,
        effective_date = effective_date,
        client_batch_id = f'BATCH_1{vault.get_hook_execution_id()}'
    )
```
- This is not ok because it's using multiple directives:
```python
fees_pid = PostingInstructionsDirective(
    posting_instructions = fee_related_instructions
)
interest_pid = PostingInstructionsDirective(
    posting_instructions = interest_related_instructions
)
return PostPostingHookResult(
    posting_instructions_directives = [
        fees_pid,
        interest_pid
    ]
)
```
## Performance impact of Fetchers
- We should always split and optimize the data fetched for schedules because some schedules require a lot of time series data which will impact performance
- We should only fetch just enough data needed for the schedules

## Performance impact of Schedules
- Use `EndOfMonthSchedule` type to define recurring monthly schedules so that we don't have to check for invalid date like 30th Feb, therefore remove overheads and improve performance.
- For example if we want to run a schedule at 12.00PM on 30th every month
```python
def execution_schedules():
    return [
        (
            "TEST_EVENT_1",
            {
                "schedule_method": EndOfMonthSchedule(
                    # runs every 30th of each month
                    day=30,
                    # at 12PM
                    hour=12,
                    # if that month doesn't have 30th then use the first valid day after
                    failover=ScheduleFailover.FIRST_VALID_DAY_AFTER
                )
            }
        )
    ]
```

## Performance impact of Hooks
- We should not defind empty hooks because they still cost performance
- All helper functions should be executed by a self contained helper method named `_handle_name_of_schedule` because the code is cleaner and underscore prefix is considered as private functions
- For example `_handle_accrue_interest()` `_handle_repayment_day()`
- Try to reuse helper functions across contracts

## Performance testing framework
- To test and record execution time of `Smart Contracts` executing the schedules and postings and analyzing to determine the how they will perform in production
- Before performance testing, unit testing, end to end testing and simulation testing are is already conducted
- Most tests invole `individual` `Smart Contracts` running in isolated environments, but Performance testing tests `multiple` `Smart Contracts` at the same time to see what'd happen to the performance at scale

## Design BAU Performance Testing
- Clearly define sensible `TPS` and `roundtrip time` for all scenarios, based on BAU production load
- Ensure `TPS` and `roundtrip time` clearly defines how much time is allocated for Vault
- Leave some headroom for `growth`
- Upstream systems will add time and bottlenecks
- If possible, test end-to-end

## Design Stress Performance Tests
- Define complex journeys to test
- The most complicated edge cases are where we can find the performance issues
- Ensure to test end of day/month/year using your estimated account numbers and posting numbers
- Test Smart Contract complex scenarios:
    - Creating `multiple postings` in 1 schedule
    - Starting `Workflow` in the schedules
    - Complex `Post posting` scenarios
    - Posting to `other Vault Accounts` in the schedules
    - Post posting rebalancing to multiple saving goals and sub accounts
    - Complicated schedules such as Payment Due Date on Credit Card accounts
    - Hybrid scenarios such as incoming postings during schedules
## Design Contract Performance Testing - Framework
- Prepare `a realistic data set`, ideally production clone.
- Test against the data in a dedicated performance testing environment because it will put the system underload
- Test multiple `Smart Contracts` instead of just 1
- Monitor and extract result data and metrics of the tests via `Grafana` or `Kafka`
- Key metrics such as `execution time` and `transactions per second`

## Design Contract Performance Testing - Best practices
### Parameters
- Use shapes only when we have multiple parameters with the same properties
- Don't define new shapes that behave the same as existing shapes
- Use `Money number` type for all interaction with currency, postings and transfer
- Default values should serve as the example of values that we expect 
- When a contract is upgraded and new parameters are introduced, existing default values will be used as actual values

### Hooks
- Cache results of calls to `vault` object, don't make multiple calls with the same arguments
- Do not use empty hooks because they still cost some performance
- Only fetch just enough data for the operations, Vault will check `@requires` to decide which and how much data needs to be retrievied

## Functions
- Use underscore `_` with helper functions
- Try to `reuse` functions across `Smart Contracts`
- Use `Type Hints` in helper functions
- Don't pass the entire `vault` object into helper functions, only pass necessary arguments

## Postings
- Consider whether to set `override_all_restritions` to `True`
- Consider what happens if the account you're posting is blocked
- Consider if we should reject a posting
- Only apply restrictions if there are specific reasons
- It's useful to provide Instruction Details in the form of key-value such as `Description` or `Event`
- Try to keep number of `Posting Instruction Batches` as small as possible, ideally 1, if we have to use multiple batches then can add index to the `client_batch_id`
- Do not posting `Posting Instruction Batches` in a `loop`
- Each `client transaction ID` should be unique and contain just enough information to trace including `hook_execution_id`

# Smart Contract Testing
## Unit testing and Simulation testing
- The `Smart Contract SDK` is a Python package which provides unit testing utilities and some examples for unit testing
- Simulation testing is when we register a `Smart Contract` inside an `in-memory approximation` of Vault known as `Contract Simulator`

## Building a testing framework
- Typically we build a `test suite` and use a testing framework like `pytest` to run
- Using Vault classes in the SDK and Mock objects to simulate the inputs and conditions 
### Unit testing
- `Unit testing` can be automated so it's the cheapest to run.
### Simulation testing
- `Simulation testing` involves testing more complex scenarios with multiple events that can represent real business cases, provide a higher degree of confidence
- `Simulation testing` can also be automated but it requires more configuration and more `expensive` and `slower`
- Both these testing can be done without waiting in real time.
### System testing
- `System Testing` provides the `most accurate` testing result, it tests most scenarios, in multiple environments but it also `requires connection` to Vault
- `System Testing` should be used to test all cases not covered by `unit testing` and `simulation testing`
- `System Testing` can also be automated but it requires even more configuration and the most expensive one, but it provides the highest degree of confidence
### Manual Testing
- `Manual Testing` is when we upload a `Smart Contract` to Vault and let it run in real time.
- `Manual Testing` can test specific limited scenarios, removes all isolation and uses a real UI.
- `Manual Testing` cannot be automated and requires the most configuration and setup, it requires significant matter knowledge and allows more free-form testing

## Unit Testing
- Smart Contract code is organized around `Expected Parameters`, `Metadata` and `Hooks`
- Since Unit Testing only tests the Python code, it `doens't cover Expected Parameters` and only `part of Metadata`
- Unit testing is cheap, fast to run and can run in local, multiple tests can run at the same time in batch
- But it's limited within the functions, cannot test againset environment, not all scenarios can be tested
- API cannot be tested, quality of mocking and assertions depends on the writer, Python unit testing standards are not constant
- Should only be used to test specific functions and their edge cases, should cover all lines
- Example:
    - Computing the interest rate:
        - If balance >= 30_000 USD => interest = 15%
        - Else interest = 7%
    - Test cases:
        - Balance = 0 expected rate = 7%
        - Balance = 29_999.999 USD expected rate = 7%
        - Balance = 30_000 USD expected rate = 15%
        - Balance = 30_000.1 USD expected rate = 15%
        - Balance = -30_000 USD expected rate = 7%
- We should always use libraries like pytest/unittest, classes from Contracts SDK and Mock objects.
- Each test should have a meaningful name, aim for 100% test coverage

## Simulation Testing
- Can cover `Metadata`, `Expected Paramters`, `Hooks` and `Vault Helpers`
- Runs `2nd fastest`, covers more scenarios, in the `past or in the future` and runs Smart Contracts against `simulated Vault sandbox`
- Can simulate `existing products` in Vault, no change made to the environment needed
- `Postings and Balances` services are simulated so that posting instruction directives result in updated postings and balances
- `Scheduler` is also simulated so all scheduled code is executed at required times
- `No real time`, all takes `0 second` to run, `no new resources` are actually created
- `Cannot simulate actual API calls` via environments, `cannot run in local` because it has to connect to an endpoint (`/v1/contracts:simunate`)
- Simulation testing `requires access to environment` with actual Vault instances, it cannot test `Restful` or `Streaming API` and currently it's not supporting `event_timezone`
- Simulation should be the `majority` of tests, it can cover complex business scnearios that unit testing cannot
- Example: 
    - Enfoce max balance 1000 USD
    - Test cases, running simulation for 7 days
        - Balance = 0, deposit 100 USD => accepted
        - Balance = 100, deposit 500 USD => accepted
        - Balance = 600, deposit 300 USD => accepted
        - Balance = 1000, deposit 100 USD => `rejected`
- Keep simulation tests `as short as possible` because they have performance drawbacks
- Each Contract interaction (schedules, hooks) should be tested with `at least 1` simulation test
- Simulation tests should be run in `CI pipelines` or can run `individual/batch` of tests during development

## System Testing
- Test `all services` and `interfaces` that Smart Contracts interact with with little to no test isolation
- Contract and Workflow testing `interacts directly` with Vault `Restful` and `Kafka`, `except for UI` because Workflow gets data from Kafka rather than Vault Apps
- System Testing should be used to cover all scenarios that Unit Testing and Simulation Testing cannot cover
- It offers the most accurate, covers the most cases and highest degree of confidence
- But it's also very expensive, requires connection to Vault environments
- Example:
    - Make time deposit Smart Contract
    - Cases:
        - Upload a smart contract
        - Create a customer
        - Create a time deposite account inside the smart contract
        - Clean up
- Should create dummy IDs for testing
- After testing, there should be a process to clean up the environment:
    - Zero out or remove all accounts
    - Mark customer as decease or inactve
    - Delete all processes generated
## Manual Testing
- Can test all or specific elements, can be used to test either very complex scenarios or simple ones against real insetance in dev or QA environments
- `Not really any benefits` since most scenarios should be covered by `System Testing`, could be useful to test some edge cases, corner cases or race conditions
- Time consuming, expensive to setup
- Example:
    - Create an account using API
    - Add funds
    - Check the next day if interest has been calculated correctly
    - Create a second account and transfer funds
    - Check UI if the transfer is successful.
- We should only use Manual Testing sanity test other tests or to help QA come up with ideas about System/Simulation testing

## Debugging Smart Contracts
- Use logging, print or IDE to debug 
- Calls to Vault Mock can be seen by outputing `mock_vault.mock_calls`
- If contract behaviors are not in line with expections then we can:
    - Can return contract notifications for hooks containing debug data including parameter values or calculation results
    - Results returned from `simulation test` contain the information about the test
    - For `System tests`, data can be seen in `Operations Dashboard's messages section`
    - Add a `dummy posting` containing debug information in the `instruction details`, the result can be seen in the response object
    - `Exceptions` can be seen in `Service Logs`
- pre_posting_hook is in hot path so it's limited in the directives it can return to show the reason why it's rejected so we can add more details to the returned response to let us know what went wrong
```python
if disallowed_denominations_used:
    return PrePostingHookResult(
        rejection = Rejection(
            message=f"Cannot make transaction in denomination: "
            f"{', '.join(disallowed_denominations_used)}."
            f"Transactions must be in {allowed_denomination} only",
            reason_code=rejectionReason.WRONG_DENOMINATION
        )
    )
```
## Code Coverage
- When running a test in Vault, `2 files` are generated
- The report file contains `high level test coverage report`
- The report.xml file contains the `line number` of covered and uncovered code
- Coverage is a company standard, we should aim for 100%
- We should test edge cases and cover all calculations
- If it's not possible to cover a line, we need to check if it's possible to refactor to make the code or testable
- Sometimes metadata or balance addresses can drag down coverage because unit tests don't call them, this is expected
- The `Inception products library` contains tests to cover all testing areas.
- The `Contracts SDK` contains `contracts_api` package which we can use to import `all Contract API classes` and types needed for testing

# Restful API integration
## Introduction
- Restful API is for synchronous interactions and Kafka streaming API is for real time data comsumption. These are 2 primary interfaces to integrate with Platform Layer
## Benefits
- `Idempotency`: `identical requests` return `identical responses`, for requests that modify the state, a unique `request_id` is assigned
- `Public`: can be used with an `unlimited` number of systems and apps
- `Convinient`: Manages `external file references`, `validate requests`
- `Security`: `Access token`, `field masks`, `HTTPS`

## Vault Core API
- It's Restful only, uses JSON format
- Methods: GET, POST, PUT and DELETE

## Request Components
- URL, URI, parameters...
- Headers: `X-Auth-Token`, generated initially by Vault and can be provisioned by `Core API` or `Ops Dashboard` later
- Optional headers like `X-On-Behalf-Of-Customer-ID`

## Create a new account
- Example of how a new Account is opened
  - Customer asks the bank to open a new account
  - Bank confirms the request is being processed
  - The request is routed though bank gateways
  - Check customer's credentials and validity (KYC, AML, Credit)
  - Call Customer endpoint of `Core API`
  - Send notification that the process of account creation has been commenced
  - Call `/v2/account` endpoint to create a new `Account Record` (status `Pending` or `Opening`)
  - An instance of `Smart Contract` is `associated` with the new account
  - The new record is forwarded to the `Core Streaming API`
  - `Notification` is sent to the customer
- The request to create a new account might look like this
```JSON
{
    "request_id": "value",
    "account": {
        "id": "value",
        "product_id": "value",
        "product_version_id": "value",
        "permitted_denominations": [
            "value",
            "value"
        ],
        "status": "ACCOUNT_STATUS_OPEN",
        "opening_timestamp": "value",
        "stakeholder_ids": ["..."]
    }
}
```
- And the response might look like this
```JSON
{
    "id": "123123",
    "name": "value",
    "product_id": "value",
    "product_version_id": "value",
    "permitted_denominations": [
            "value",
            "value"
        ],
    "status": "ACCOUNT_STATUS_OPEN",
    "opening_timestamp": "value",
    "closing_timestamp": "value",
    "stakeholder_ids": ["value", "value"]
}
```
- For an account to be opened, the `Customer ID` must exist so that `KYC/AML/Credits` processes can be run, the account is associated with a Product of Customer's choosing (which is a `Smart Contract version`). 
- The status could be Open or Pending Open depending on the workflow of the bank
- `/v1/customers`: get, create, update, batch get with list of customer IDs, search for customers
- `/v2/accounts`: get, list, create and update customer accounts
- When an account is created successfully, a message is populated to this topic `vault.core_api.v2.accounts.account.events`

## Transfer funds
- Example of transferring funds:
  - Customer initiate request to send 500 USD to another account
  - Bank checks account balance
  - Bank checks credentials and validity
  - Bank debit from source account and credit to beneficiary account, update account balance
  - `Core API` uses customer ID to verify the customer, update balances and stream the result to `Streaming API`
- `/v1/balances/live`: to `view` customer's balances
- `vault.core.postings.requests.v1`: Posting API's topic to `request` to send funds
- `vault.core.postings.responses.v1`: Posting API's topic to `retrieve` the result
- `/v1/posting-instruction-batches`: to create `high priority` Posting Instruction Batches
- When a posting being made, messages are sent to these topics `vault.core_api.v1.balances.account_balance.events` and `vault.core_api.v1.postings.posting_instruction_batch.created`

## Update parameters
- Example of updating account parameters
  - After using the account for a while, the customer is entitled for a credit increase
  - The bank receives customer data from Credit Bureau
  - The bank combines with its own data and decide whether to increase/decrease or keep the customer's spending limit
  - The bank update parameters using Core API and sends notifications to the customer
  - The customer logs into bank app and found that spending limit has increased
- `/v1/parameters`: get, batch get, create and update parameters
- `/v1/parameter-values`: get, batch get, create and update parameter values
- When a parameter is updated or created, messages are sent to `vault.core_api.v2.accounts.account.events`, `vault.core_api.v1.parameters.parameter.events` and `vault.core_api.v1.parameters.parameter_value.events`

## Create restrictions
- Example of creating and applying restriction sets
  - AML system detects abnormal activities on the account and reports to the bank
  - The bank flags the accoutn which initiates a workflow to creates and applies restrictions on the account
  - Create an account restriction set
  - Call account endpoint, update account record and apply the restriction set
  - The bank opens an investigation on the activities
  - The bank decides whether to remove the restrictions or not
- Vault can create restrictions on 3 levels: `Customer`, `Account` and `Payment device` depending on the severity
- If the malicious activity comes from the account or payment device only then the restriction is created for account or payment device
- We can use Ops Dashboard to create a Restriction Set Definication where we can specify the level, which activities we want to restrict
- The flag set by the AML will trigger a workflow to create a restriction set with exceptions.
- When applying restrictions, bank will factor in these exception to reduce the punitive effects
- For example when apply Prevent Credit at Payment Device level, we can also select `prevent_credits_exemption_conditions` to add conditions in which is restriction is not applied
- The parameters of exceptions are included as key-value inside Posting Instruction's `instruction_details` field
- `/v1/restriction-set-definition-versions`: list, batch get or create a new restriction set version
- `/v1/restriction-sets`: list, batch get, create, update a new restriction set
- When a new restriction set is created or updated, messages are sent to topic `vault.core_api.v1.restrictions.restriction_set.events`

## Close an account
- Example of account closure
  - Customer intitiates request to close an account
  - The bank checks if account's status is `Open`
  - The bank checks if balance > 0
  - The bank checks parameter whether any restrictions prevent account closure
  - The account cannot be a joint account as it requires authorization from other stakeholders
  - A check if there are no outstanding account/product version updates
  - The customer receives a notification that the account has been closed
  - A request to close account is queued
  - This request executes `deactivation_hook` and the account's status is updated to Closed, all schedules are disabled
- To close account, we still use `/v2/account` and `/v1/customer` to close
- `/v1/balances` to check or update balances
- After the account is closed, a message is sent to topic `vault.core_api.v2.accounts.account.events`

# Configuration Layer Utilities
- CLU is a command line interface for configuring resources
- Resources including smart contract, product, product version, permissions, roles, internal accounts, supervisor contract, workflow policies
- It sits between users and Vault API to manage external file references, it's idempotent, if we import a file twice it won't affect anything

## Setup
- `Access token`: only minimum access to the resources is provided to the service account
- `Supported resources`: access control, Core API, products, workflows
- `Create resource files`: create 1 or more resource files
- `Create manifest file`: defines a collection of resources uploaded and managed by CLU

## Usage
- `Validation`: CLU provides the ability to validate resources before importing
- `Exit Codes`: upon running the CLU, the results are codes that can be `success`, `failure` or `partial success`
- `Activation`: when importing, there is an option to activate resource on import, this will set the resource as latest version
- `Output format`: plain text or JSON

## Benefits
- Efficient mechamism to import resources into Vault instance
- Can import and update
- Provides clear feedback on failed imports/updates
- Secured HTTPS
- Consistent and generic resource file format which has a payload field that can map 1-1 to existing examples in docs
- Allows dependencies between resources fields in different resources after import
- Automates extraction and creation of resources

## Prerequisites
- Vault is installed and running
- `CLU version` is compatible
- `Service Account` is created
- CLU can `access` Vault API from the computer it's running on
- HTTPS

## CLU workflows
- Check if the resources are on the `supported resources` list
- Add `Resource ID` to the `manifest.yaml` file
- Add new resources or copy resources from configuration tool to the CLU
- Run validate to make sure that all resources are valid
- Run the CLU to create resources synchronously in Vault

## Methods and Options
- The correct order for commands is:
    - Command type: `import`, `validate`, `convert`
    - Environment variables
    - Configuration file
- Convert checks if there is an ongoing Plan Migration, `attempts to create` supported resources, `ignore` unsupported ones.
- `All 3` command types validate resources
- Options: --config, --auth-token, --activate-on-import ...
- CLU supports config files written in JSON or YAML
- `manifest.yaml` is in the root directory, comprised of supported Resource IDs that are to be imported to Vault
```YAML
pack_version: 1.0.0
pack_name: Configuration pack name
resource_ids:
- current_account
- supervisor_contract_resource_id
- supervisor_contract_version_resource_id
```
- `resource.yaml` / `resource.yaml` can be in the same directory as `manifest.yaml` or can be in subdirectories, they define the configuration for resources. They can have any names but have to have suffix `resource.yaml` for `single` resource definition or `.resources.yaml` for `multiple` definitions
- It's not advisable to store access token in he config files
- If the flag activate-on-import is set then Vault will execution activation after successfully imports `manifest.yaml`
- To set resource version as active, can simply set `activate` flag to true

## Configuration Pack
- Self-contained logical unit that contains manifest and resources and external files
- External files are any markup files, Python code for Smart Contract ...

## Configuration Bundle
- `Collection of CLU packs` that have been grouped together so that they can be executed simultanously
- We can bundle CLU packs in 1 directory and run import, validate, convert by providing the `path to the directory`
- Manifest files can have different names but must have suffix `.manifest.yaml`
- All Resource IDs have to be `unique` across all resource files
- Resources within a bundle can only `reference` to other resources `within their own pack`

## Dependencies
- Dependencies can be specified in the `payload` field that reference to external resources
- Specify dependencies using syntax `&{dependency_resource_id:dependency_resource_field}`
- If a resource depends on a failed resource, it will fail to be imported as well
- Some resources don't have `vault_id` until they are created, for example `Smart Contract Version`, unlike `Supervisor Contract version`
- For `Supervisor Contract version`, the `vault_id` is a reference to `current_account` which is the version of the `Smart Contract` it supervises
```Python
api='3.0.0'
verson='1.0.3'
supervised_smart_contracts=[
    SmartContractDescriptor(
        alias='example alias',
        smart_contract_version_id='&{current_account}'
    )
]
```
- Definition of the dependency resources must exist within the same resource pack

## How to import CLU resources
- In the root directory, create a `manifest.yaml` file and specify `pack_version`, `pack_name` and `resource_ids`
- Create a resources subdirectory and within it create a code_files subdirectory where we will create Product Version file with Contract code
- Within the resources directory, create a contract resource and contract resource version file
- Import using CLU
```
root:
    - manifest.yaml
    - resources:
        - code_files:
            - product1.py
            - product2.py
        - contract resource
        - contract resource version
```
- An example of a `resource.yaml` file
```YAML
---
type: SMART_CONTRACT_VERSION
id: current_account
payload:
    product_version:
        display_name: Current Account
        code: @{current_account.py}
        product_id: current_account
        params:
            - name: denomination
              value: 'GBP'
            - name: addition_denominations
              value: '["USD", "SGD"]'
```
- The deployment process doesn't remove any resources, it only creates new ones or updates
- After creating all files, we can run:
```zsh
./clu validate manifest.yaml
```

- There are 2 ways to automate deployment:
    - Use Core API to write integration in merge pipeline so that it can make relevant calls to upload new financial product
    - Use CLU integrated with the merge pipeline to upload 

## CLU support for Product Library installation
- Use the CLU to load products from Inception library
- Includet he latest version of CLU and Inception library in the directory
- The manifest file contains all references to all resources required for a specific package
- The resource files contain a set of resources like workflow, internal smart contract, flags ...

## Add CLU to CI/CD pipelines
- Create a `Configuration package 1` to deploy new Smart Contracts to `DEV` environment, test and verify
- Run `simulation testing` via Core API's simulation testing endpoints against the `DEV` environmemt
- Deploy to the `Configuration package 1` to `STG` environment
- `End-to-end testing` in `STG` environment using other services such as Customer Application which then execute API calls to Core API, Postings API or Workflow API
- Changes may be required so we create `Configuration package 2` and deploy to `STG` again
- Acceptance testing passes, we extract the `Configuration Layer` of `STG` environment and deploy it to `PRE-PROD`
- We can use different versions `.manifest.yaml` file for different environments

## How to write manifest.yaml
- Manifest only has 3 fields:
    - pack_version
    - pack_name
    - resource_ids
```yaml
pack_version: 1.0.0
pack_name: Configuration pack name
resource_ids:
    - current_account
    - saving_account
    - supervised_saving_account
```

## How to write resource/resources.yaml files
- Resource files have the following fields
    - type(*)
    - id(*)
    - vault_id: some resources don't have this field because we don't know their id until after creation
    - vault_version: unused
    - on_conflict: what happens if this resource already exists `IGNORE`, `UPDATE` or `SKIP`
    - payload(*): product version and migration strategy, product version has product ID and ref to the Python file, later converted to JSON to call relevant APIs
- Resources files have the field `resources` in addition to the above fields in order to list multiple resources
- Example of a resource file:
```YAML
---
type: SMART_CONTRACT_VERSION
id: saving_account_contract
payload: |
    product_version:
        display_name: Saving Account Contract
        product_id: saving_account_contract
        code: '@{saving_account.py}'
    migration_strategy: SMART_CONTRACT_VERSION_STRATEGY_NEW_PRODUCT
```
- Example of a resources file:
```YAML
---
resources:
  - type: SMART_CONTRACT_VERSION
    id: saving_account_contract
    payload: |
        product_version:
            display_name: Saving Account Contract
            product_id: saving_account_contract
            code: '@{saving_account.py}'
  - type: SUPERVISED_CONTRACT_VERSION
    id: supervised_account_version
    vault_id: supervised_account_version_1
    payload: |
        supervisor_contract_version:
            id: supervised_account_version_1
            supervisor_contract_id: @{saving_account_contract}
            display_name: Supervised Account Contract
            description: Supervised Contract for Account Contract
            code: '@{code_files/supervisor.py}'
```

# Vault Overview, Tooling and Infrastructure
## Architecture
- Vault is deployed to AWS but can be deployed to other cloud providers
- All technologies are not vendor-locked and can be swapped out with others, for example EKS can be changed to managed Kubernetes clusters
- On the outside, it has a `Vault VPC` that spans across multiple `Availability Zones`
- Vault Core services are deployed to Kubernetes (EKS)
- For database, Vault is using `RDS` with Postgres DB with a `master` instance and a `hot-standby` instance, which `replicate synchronously`
- For managing secrets, Vault is using `Hashicorp` but it can be swapped with `AWS Secrets Manager` as well
- For monitoring, it's using `Prometheous` for metrics collection and `Grafana` for visualization
- For logging, it's using `fluentd` to collect logs, `ElasticSearch` for log searching and `Kibana` for visualization
- For networking, `Kubernetes Ingress` is used to `routing/load balancing` for traffic from outside while `Istiod` is used for `service mesh` for communication between pods

## Kubernetes
- For tracing, Vault is using `Zipkin`, probably will move to `Jeager`
- In terms of networking, Vault is a bit different to other legacy systems is that it's using Mutual TLS where both the server and client have to present TLS certificate so that the other can verify
- Vault supports HTTP and gRPG load balancing

## Installation
- There is a YAML file that handles the configuration where the client can configure across all services in Vault, for example whitelisting rules, service level information...

# Downstream events from Vault
## Overview
- Vautl has 2 APIs to interact with external services, Restful API and Kafka
- `Restful API` supports Core, Postings, Payment Hub, Audit, Access Control, Data Loader API while `Kafka` doesn't support `Access Control API` 
- Vault uses streaming API across all components thus they can stream all state change events to downstream services to do things like ML, analysis, fraud detection...
- There are 2 data types streamed: `fact` (Posting creation, balance update) and `resource mutation` (account update, contract version update...)

## Architecture
- All state change events being streamed to downstream services is useful to inform them to take action for example when a new account is created, a message is sent to other services to send emails, create bank statements and issue cards
- Vault uses Transaction Outbox pattern to ensure `at least once delivery`
- Every events streamed to Kafka is actually recorded in a database in 2 tables `resource_table` and `journal_table`
- For example when a new account is created:
    - Account Service creates a commit intent to the `resource_table` in the DB
    - Account Service sends a message to Kafka and marks the event as `Delivered` in the `journal_table`
    - If Kafka suffers any outage, when it's back online, a Poller will check if there are any missing events in the `journal_table`, if yes then it will pull the missing events to Kafka and at the same time mark the events as `Delivered`
- The Poller knows the missing events because Disaster recovery job marks each outage with a start and end timestamp, so the Poller can just find events between these start and end timestamp

## Events relating to moving money
- Vault is only 1 of the multitude of systems for recording fund movements
- All of these movements are summerized and aggregated into a General Ledger

### Card schemes and processes
- Vault can process requets from any card processing schemes but it's not a card scheme handling system

### Fund movements and 
- `Close loop account` (CLA) is implemented by Vault using `Restriction set definition version` at `payment device` level
- `Responsiveness`: near real-time data access allows banks to quickly adapt to business change
- `Managing Risks`: near real-time granular data support advanced anaytics capabilities to quickly flag suspicious transactions
- `Ease of integration`: Vault uses Payment device with links to multiple accounts that can receive and initiate postings

### Payment Hubs and Gateways
- `Vault Payment `is a standalone payment processing platform that can issue and process on Mastercard rail.
- `Vault Payment` can be integrated with `Vault Core` or legacy systems

### Liquidity management 
- Vault supports dynamic management of intraday liquidity risk so that bank can manage liquidity position and can meet payment obligations
- Liquidity allocation is performed dynamically, enabling automation/scheduling of liquidity position so that banks can better manage buffer and handle unexpected spikes
- Postings and balances can be used to determine customer's liquidity position across products and internal accounts, capturing the most granular data in near real-time
- Vaul is the single source of truth for liquidity

### Liquidity management architecture
- `Payment engine` receives payment from card schemes and make API calls to `Postings API`
- Vault uses `Kafka` to stream out postings and balances events, Kafka keeps these events in queue and waiting for downstream systems to consume
- `Stream processing engine` consumes the events from postings and balance topics and save it to the `database` or `repository` for `offline analytics`
- `Stream processing engine` streams events to `real-time monitor platform` that tracks liquidity in real-time
- Real-time schedules run a `forecasting model` using real-time and historical data 
- The real-time liquidity platform `informs` a `liquidity service` to manage liquidity position in Vault using `Postings API`

## Events relating to Data and General Ledger
### Support data integration and General Ledger
- All events related to postings are streamed as posting instruction batches, these events are consumed to update the `General Ledger`
- These events are usually windowed into a specific day, using the cut-off time
- For regulatory reports, streamed events are not enough for banks to build their own view of this window so Vault also support API calls to the Core API to do `Integrity Check`, `Posting Matching`, `Balance Matching` and `Discrepancy Resolution`

### Real-time general ledger and reconciliation integration
- Events allow General Ledger to do reconciliation in real-time at any time instead of waiting until EoD
### General Ledger and reconciliation integration architecture
- Vault processes postings and balances and generates postings and balances events
- Vault streams out posting and balance events to Kafka, along with other events like parameter updates or account updates (`Operation events`)
- An `Operational Data Store` consumes these events while `General Ledger postings` are also streamed to `General Ledger` to update
- The bank queries `Core API` to get `accurate balances` as of the cut-off time.
- The bank carries out `reconciliation` by `comparing data` from Core API and General Ledger, any `discrepancies` are fixed by calling the `Postings API`
- `Operation events` will indicate the completion of a `Schedule Tag`, which can trigger the `final round` of reconciliation before EoD position is produced
- `Reports` are generated once the bank is confident that `all data` has been received from Vault

## Events relating to Customer channels
- Data streamed from Vault can be used to run analysis to build insights about customer habits, preferences which support future decision making whether manual or automated

### Customer insights and personalized product offerings
- Clients System of engagement calls Core API to create/mutate resources
- Vault streams mutation events to Kafka
- Kafka streams events to Stream processign engine (Google dataflow, Apache Flink...) for aggregation, deduplication, transformation
- Data is saved in a data store (Google BigQuery) for analytics or model training
- Stream processing engine interacts with a trained model by providing real time events and receives insights
- Insights are pushed to the customer
- When an opportunity for a new product is detected, Pricing and credit risk engine is called to produce a suitable offering
- Pricing and credit risk engine calls Vault to simulate the product given the terms, the received results are checked against 

### Customer notifications
- Vault streams out events about postings, balances, accounts ...
- Events are consumed, enriched and transformed by a machine learning model and a notification request is sent to a notification engine
- The Notification engine distributes the notifications to various providers

## Events relating to Customer knowledge
### Improve customer royalty
- Use streamed data, banks can use trained models to provide predictive alert when a spending limit is reached, categorize transactions into geo-location, type of merchange or spend classification, if/else to transfer salary into saving accounts or rewards customers for saving $20 a month.
- Payment engine inititates postings via Postings API
- Vault streams posting and balance events to Kafka
- Events are consumed by Stream processing engine
- Data is saved to data repository for offline training
- Stream processing engine feeds data into System of engagement where data is enriched and transformed in to customer-facing transactions
- Leverage System of engagement to develop reward measures, gamification offers...

