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


# Building a Chart of Account in Vault
## Introduction to Chart of Accounts
- A chart of account is an index of all financial accounts in a general ledger of a company 
- It contains all transactions within a period, usually broken down into subcategories
- The range of categories, subcategories and types of attributes that contribute to a category can vary across financial institutions but no more than 6-7 attributes
- The Vault CoA contains 2 types of account Customer Account and Internal Account
## Vault Core System of Record
- Each account is first partitioned by asset type such as cash, commercial money ...
- Then each asset type is divided by denomination
- Each denomination is divided into addresses, each denomination will have different addresses for different purposes
- Every address has 3 phases, pending outbound, pending inbound and committed
## How to build chart of account
- To create a customer account, we need to identify how many customer accounts and which accounts are required, ie do we need to keep both existing closed and open or just open accounts
- We also need to decide whether to manually create accounts or use `Data Loader API` and `Posting Migration API`
- `Internal accounts` can be manually created in Vault via API `/v1/accounts` with `account.type=ACCOUNT_TYPE_INTERNAL`
- A more efficient way to create multiple `internal accounts` is to use `Configuration Layer Utilities`, before that you need to specify the `smart contract` to be associated with the `internal accounts`
- `Internal accounts` can be used to customize accounting models and replicate a business-specific chart of accounts
- `Internal accounts` reflect some operational balances of the bank so their balances can be:
    - Based on some payment integration like nostro accounts
    - Used by Smart contracts to capture profit and loss
    - Defined as either Liability or Asset
- `General Ledger `is the single source of truth
- All postings and changes to the `Posting Ledger `are streamed to Kafka as `Posting Instruction Batches` so that they can update the `General Ledger`, these events are windowed into "days"

## Vault Product ledger capabilities
- Vault maintains product ledgers for all business lines of a bank, following the double bookkeeping practice
- There are 2 product ledgers, `Balances Ledger` and `Postings Ledger`, they interact with each other
- The Balances Ledger contains 2 types of balances, customer account balance and internal account balance
- Customer account balances and accounting behaviors are managed by Smart Contracts
- `Internal accounts (Line of Business)` are used to manage internal accounts and can be mapped to General ledger by account IDs
- These accounts are `defined by banks`, according to the `Chart of Accounts` structure used for each `Line of Business`
- For example, we have a loan account:
    - Vault records a principal balance in the loan account's principal address
    - Vault then creates a corresponding Asset balnace in the internal account
    - Vault calculates accrued interest during the interest lifecycle in the interest address and records corresponing balance in the internal interest address
    - For fees, Vault records a fee in the fee address and records a corresponding fee in the internal fee address
## Maintaining GL money movements outside of Vault
- Vault is the General Ledger and cannot adhere GAA practice so it's recommended to manage General Ledger money movements outside of Vault
- To manage GL using a 3rd party system, we need Event type and Product type
- When charging interest, Smart Contracts make 2 posting instructions
- These posting intructions contain metadata that the GL third party will consume and make postings to GLx
- When the customer wishes to repay, a single external posting instruction is made to the Vault, Smart Contract rebalances the account accordingly and the 3rd party GL system again consumes the events and makes postings to the GL

# Advanced End of day
## What is end of day
- The activity to calculate balances at the end of day, these balances are used to apply interest and fees
- The BAU at the end of day and the interest and fees give the complete financial picture of a set of accounts, this position is then streamed to downstream system for reporting
## Process terminology
- `Day 0`: the day that is ending
- `Day 1`: the next day
- `1&deg; cut-off`: the boudary between `day 0` and `day 1`. This is the primary cut-off, whose balance is used to calculate interest and fees
- `Overnight postings`: interest and fee postings generated by scheduled processes overnight, rely on the balance of cut-off
- `Completion`: when all overnight postings have completed
- `Overnight`: the period between `cut-off` and `completion`
- `BAU postings`: postings generated as the result of customer activities
- `2&deg; cut-off`: the point in time which `End of Day` is scheduled
- `Grace`: the period between cut-off the schedule execution
## Position terminology
- There are 2 ways to handle late postings, either by assigning it to day 0 or day 1
- When assigning to day 0, we have to wait for all overnight postings and schedules to complete because the position is only available at `Completion`
- When assigning to day 1, we have to include all overnight postings of day -1 while day 0 overnight postings are handled by day 1, the position is available at `cut-off`
## BAU and Overnight activity overlap in a 24/7 core
- `00:00`, cut-off, `Interest balance` $100, `Balance at 00:00` $100, `Live balance` $100
- `00:15`, ATM withdrawal $50 ,BAU, `Interest balance` $100, `Balance at 00:00` $100, `Live balance` $50
- `00:30`, day 0 grace, late credit card settlement $20, backdated to day 0 23:59:00, `Interest balance` $80, `Balance at 00:00` $80, `Live balance` $30
- `01:00`, overnight day 0, 2&deg; cut-off, scheduled interest calculation, credit $1 to balnace, backdated to day 0 23:59:59 (configurable in Vault), `Interest balance` $80, `Balance at 00:00` $81, `Live balance` $31
- `01:05`, completion, `Interest balance` $80, `Balance at 00:00` $81, `Live balance` $31

## Vault EoD capabilities
### Schedule
- Vault allows to schedule Smart Contracts to execute certain code and passes the time of execution to the code.
- Smart Contracts can also make backdated postings
### Calendar
- Some account operations like interest accrual or application of interest differ during special times like holiday
- Vault provides `Calendar` which is a collection of `Calendar periods`
- The resolution of the calendar periods could be hour, day, month, when each period passes, the counter increase by 1, hour 0, hour 1...
- Once we create a `Calendar` and associate it with a `Calendar Period Descriptor`, we can define special periods where `Smart Contracts` behave differently for example during `bank holidays`
### Processing Group
- A `logical grouping` of `Accounts` and `Plans` that can be used to compartmentize sets of accounts
- Each `Processing Group` operates on an `isolated ledger` so we can run multiple EoD operations against each Processing Groups
- Processing Groups are timezone-aware that can be taken advantage of to schedule and when writing Smart Contracts, we don't have to specify timeozone anymore
- We can `PAUSE` or `UNPAUSE` all schedules of an `entire` Process Group

### Vault jobs
- A `web app` that tracks the `progress` and `health` of `long running` account processes
- Each `Vault Job` consists of multiple `Vault Job Operations` and provides insight into how quickly its operations are being processed, `Complete` or `Error`
- Each `Vault Job Operation` represents an individual process which has a corresponding `scheduled job` in Vault
- `Vault Job Operations` are grouped together by `processing group`, `Smart Contracts version`, `event type` and the `scheduled timestamp` of the underlying scheduled job
- Only `Error Operations` are displayed by Vault Jobs, an error Operation contains IDs and links to affected resources

### Account Schedule Tags
- Are `events` that `provide updates` to the `outcomes` of a set of schedules which are identified by a `Tag`
- Scheduled jobs that have the same tag are called `scheduled operations`
- `Account schedule tags` allow us to see whether a `schedule with a tag` has been completed `across all accounts` which is useful to downstream services so that they can start their EoD reporting

## Bank implementation
- `Definition`: banks need to `define` their EoD process (cut-off, overnight, grace...)
- `Observation`: the events that Vault produces need to be `observed`
- `Reconciliation`: balances across `all accounts` have to be `zero`, use `Stream reconciliation API` to check for `missed` events, reconciliation jobs can be run any time in a day
- `Aggregation`: with reconciliation, now we can `confidently` push all data downstream for `reporting` and `accounting`
- There are 3 types of Ledgers within a bank
  - `General ledger`: provides overarching financial view of the bank, owned Financial Reporting team who also owns the Chart of Accounts, stakeholder is CFO
  - `Subledeger`: provides periodic view at an entity level, number of ledgers can vary from bank to bank, owned by divisional management
  - `Product ledger`: provides real time, 24/7 view related to many products, owned by product owners and business unit management
- General ledger and Subledger solutions could be provided by SAP, Oracle 
- Product ledger solution can be provided by Thought Machine

# Kafka topics
## What does streaming entail
- While all Vault versions are shippped with Kafka, it's not intent for production, clients need to install their own Kafka or they can use other managed services like MSK, Confluent Platform/Cloud as long as the version is compatible
- Kafka is hosted on multiple servers that span across multipel regions and AZs, some of them are storage layer or brokers, some of them run Kafka Connect that can integrate with other sources like DB or other Kafka clusters

## Kafka architecture
- `Producer`: producing events
- `Broker`: `multiple` brokers, `stateless`, each can handle thousands of requests per seconds
- `Zookeeper`: `keeps states` of all brokers and elect new broker leader, informs producers/consumers about `failure`
- `Consumers`: downstream services
- When the entire Kafka cluster is down, when it's back on, a `Poller` job will poll the `journal table` to check if Kafka missed any events and resend them, this is Disaster recovery, using `Transaction Outbox` pattern.

## Topics
- Kafka can have hundreds of thousands of topics, some common topics:
  - Vault.api.v1.postings.posting_instruction_batch.created: status of a PIB when being created
  - Vault.core.internal.postings.requests.v1: internal posting request
  - Vault.core.internal.postings.responses.v1: internal posting response
  - Vault.core.postings.post_processing_failures: post posting response post_processing_failures
  - ...
- There can be unlimited number of producers and consumers, each consumers can subsribe to multiple topics
- Each topic is partitioned into multiple partitions so that multiple consumers can read from a topic/partition in parallel
- Messages are ordered and served in a FIFO manner, each message has an offset number

## Streaming APIs
- Vault core has 3 `main` APIs that utilize streaming:` Core API`, `Postings API` and `Streaming API`
- Beside, Vault core streaming also includes `Audit API`, `Products API`, `Data Loader API` and `Payments Hub API`

## Use case - Account creation process
- Customer uses bank app to create a new bank account
- Request is routed through bank gateway and conduct KYC/AML checks
- Call Core API /v2/accounts to create a new account
- Vault creates a new account, associates with a Smart Contract version and sychronously responds to service
- Service calls Core API /v2/balances/live and acknowledges bank app that the account has been created
- An event is created and forwarded to `Core Stream API` so that it can be sent to Kafka
- Data warehouse pulls the data to store 

## Use case - Balance transfer
- Customer opens bank app
- Bank app calls `/v1/balances/live` to check balance
- Customer selects payee and amount and hits Send
- Mid tier checks balance by calling `/v1/balances/live`
- Mid tier `KYC` checks
- Calls `Payment processor` to initiate payment
- `Payment processor` sends posting requests to Postings API's request `topics` (`vault.core.postings.requests.v1`)
- `Postings processor` creates the postings and sends response to Postings API's repsonse `topics` (`vault.core.postings.responses.v1`)
- `Payment processor` sends request to `Notification orchestration` to send notificatons to customer that the payment is sucessful

## Creating and managing Kafka topics and partitions
- There are at least 3 brokers and 1 Zookeeper
- A topic can have multiple or 0 partitions
- Partitions can be created using `vault-topic-manager` deployment
- The topic manager has its own API to config a topic and its partitions in a `.proto` file
- The .proto file is then ingested and applied to the Kafka cluster
- All messages with the same partition key is stored in the same partition
- The max number of consumers consuming from a topic = the number of partitions
- By default, each topic has 6 Partitions

## How many partition is enough
- Each consumer with a unique `Group ID` will consume all messages from a topic without interfering with other Groups
- So if we have `an application` that needs to consume `all messages` from a topic, we can create a `consumer group` with multiple consumers so that these instances can consume messages in `parallel`
- When a consumer joins a group, the `leader broker` will assign a `subset of partitions` to each consumer in the group 
- If there are `more consumers` in a group than `partitions`, some consumers will just sit there and `consume no messages`, we can fix this by `increasing` the number of partitions (it's `not possible to decrease` the number of partitions though)
- Having too many consumers will improve Throughput and Availability
- However it also increases latency to replicate messages, cost, resources and boot up time when the whole cluster is down or rolling updates.
- The optimal number of partitions depends on what we do, in some cases 6 partitions is more than enough for a topic for example failure messages that are not common
- We can start at a low number and gradually increase it to meet the demands

## Strict ordering
- Use `event ID` to produce messages to achieve strict ordering.
- A partition will receive all messages with the same `partition key`
- Benefits of strict ordering are messages are `idempotent`, `higher efficency` in processing
- Cons are not easy to reduce number of partitions, repartitioning breaks the order, not able to use DLQ and poor utilization of resources (1 partition may have way more messages than the rest)
- Considering these disadvantages, it's recommended not to use Strict ordering, try to find other mechanisms to order messages

## Kafka Configuration and Setup
- `Vault Certified Environment Matrix` shows the compatibility between Kafka and Vault components every version
- For authentication, Vault supports:
  - `mTLS`: The client `dictates` the CA to sign both server side and client side certificates
  - `SASL-SCRAM`: Scram creds are created by `Thought Machine` in Vault
  - `OAuth`
## Topic creation
- There are 3 ways the client can create a topic:
  - During installation or upgrade by `Kafka topic manager`
  - During integration with external services as part of calling the Postings API by the `Kafka topic manager`
  - Client can create topics manually using `Kafka topic manager`
- When creating topics manually using `Kafka topic manager`, there are 2 ways:
  - If the new topic belongs to an existing service, then just add the new topic to the existing `.proto` file, inside `/vault/infrastructure/topics` of the repo
  - If the new topic is the `first topic` of a `new service` then create a new `.proto` file, we should create a new folder within service's namespace and put the file there. Use the `topics()` definition to generate 2 build rules, 1 is to validate topic properties and 2 is the `.proto_library`. The `.proto_library` needs to be added to the dependencies within `//vault/infrastructure/topicadmin/common_topic_manager:generate_topics_file`
- `Kafka topic manager` can `reconcile` with topics by going through the `.proto` files but `ACL` (Access Control List) has to be enabled
- The topic names should follow the convention that indicates 
  - Product/domain of the service
  - Type of API (internal or public)
  - Type of events
  - Type of topics (used for DLQ...)
- Can use prefixes like vault.*
- Other legacy prefixes are ep.* and scheduler.* are not encouraged

## Topic parameters
- `kafka_name`: name of the topic
- `kafka_retention_period_ms`: the retention period of messages
- `kafka_numer_of_partition`: number of partitions 
- `kafka_safe_to_delete`: flag for safe to delete the topic or not, used for deprecated topics
- `kafka_topic_repartition_strategy`: the strategy to repartition when we need more partitions, like how Kafka redistributes the messages
- `kafka_test_only_topic`: whether the topic is for testing
- `kafka_public_api_topic`: whether this is a public topic that external systems can access
- `kafka_dlq_topic`: whether it's a DLQ topic, by default it has 1 partition and 14 day retention
- There are 3 strategy options:
  - `DISABLE`: default option, never repartition
  - `IF EMPTY`: only partiton if no messages in the topic, used for consumers that require strict ordering because reducing or increasing means delete the whole topic and recreate that topic with a number of partitions
  - `ALWAYS`: always safe to repartition
## Dead Letter queue
- Errors are classified as `Transient` (auto retries) and `Non-Transient` (no retries)
- Access to DLQ should be strictly controlled
- Messages may contain PII in their bodies and error output
- All DLQ topics have suffix `.dlq` or `.failure` in their names
- We prioritize Non-Transient errors first because these are not retries
- Transient errors also need to be taken care of because they may retry indefinitely
- Steps to recover from a error message:
  - Locate the message: using Prometheus to see the error message in the DLQ
  - Access failure reason: we must do this within the retention timeframe
  - Investigate and identify the root cause of failure
  - Instigate remedial actions to correct the error
  - Replay the event
- If the message is in protobuf format, we need to deseialize it first 
  - Determine the .proto file for deserializing 
  - Go to documentation for the specific DLQ 
  - Find the message type under "What message has been sent to the DLQ?"
  - Find the .proto file that contains the message type
  - The line <value> is the message type
  - Locate the message
```zsh
protoc --decode=test_package.TestMessage path_to_proto/test.proto
```
- The `test_package` is the package name, which is defined in the test.proto file
- The `TestMessage` is the message type

# Advanced Smart Contract
## Smart Contracts API requirements
- Contracts API provides a set of common functions 
  - Hook
  - Parameters
  - Directives: commands that are result from the execution of a Smart Contract hook
  - Requirements: data that Smart Contracts have access to during execution
- The type of hook determines what requirement data it has access to
- Data can be retrieved as time series, which allows hooks to retrieve the latest data or of a given time in the past
- `Balances` are stored as time series and can be fetched as part of this time-series
- `effective_datetime` is when the hook is supposed to execute
- Strings like `1 day`, `2 months`, `3 years` are durations starting from the `effective_datetime` to the past
- `latest` means the time-series of the `effective_datetime`
- `live` modifier when used with other range specifier fetches the most current data rather than the data at the `effective_datetime`
- Let say we have a hook to run at 12:00AM but it's delayed til 1AM
  - `effective_datetime` = 12:00AM because this is time time when the hook is supposed to run
  - `1 month` = all values between 12:00AM - 1 month
  - `2 years` = all values between 12:00AM - 2 years
  - `latest` = the value as of 12:00AM
  - `latest live` = the latest value as of 1:00AM
  - `1 day live` = all values 1:00AM - 1 day
- `Balances` can be used to check if the account has any live balance
- `Postings` are fetched similarly to `Balances`, according to their `value_timestamp`
- Flags and parameters are stored as time-series and can be declared and access using `@requires`
- `last_execution_datetime` of a hook can also be accessed with `@requires`

## Effective date
- Most hooks are injected with effective_datetime which is the time when the event is triggered
- In `scheduled_event_hook`, it's the time the schedule is set to run
- In `pre_posting_hook` and `post_posting_hook`, it's the value_timestamp of the posting
- In `pre_parameter_hook` and `post_parameter_hook`, it's the timestamp described in the API call
- In `derived_parameter_hook`, it's from the account details

## Performance requirements
- Only fetch the necesaary data, for example if we only want the data at the end of day, `latest` is sufficient compared to `latest live`
- `Parameters` and `flags` are fetched `entirely` so only fetch them if necessary

## Scheduler
- Scheduler service sits within the kernel of Vault Core and triggers events for Smart Contracts
- The scheduler service creates jobs for schedulers' clients to execute their tasks, these jobs are created according to definitions in the `activation_hook`
- By default, Smart Contracts use `UTC` as timezone but this can be overriden by defining the variable `events_timezone`
- To localize all input dates and times, we can use function `.astimezone(tz=ZoneInfo("timezone name"))`
- `Schedule Groups` are groups of schedules that run in order and can be blocked if the previous schedule has not finished
- One example is when we want a schedule to use the result of its previous schedules like accrued must occur before capitalization
- `Schedule Tags` are events that provide updates on the outcome of a set of schedules identified by a `Tag`
- The collection of all schedules with the same tag is called `Schedule Operation`
- `Schedule Tags` can be used to check if a schedule has finished across all accounts for example when interest payments are all executed across current and saving accounts
- Example: given 1000 customer accounts
  - Each current account has a schedule group that consists of `ACCRUE_INTEREST` and `APPLY_INTEREST` events 
  - `APPLY_INTEREST` can only run when `ACCRUE_INTEREST` has `completed`
  - Once `APPLY_INTEREST` in all `1000` customer accounts has `completed`, a schedule tag with value `INTEREST_PAID` is streamed from Kafka to all downstream services, meaning all Smart Contracts of this version have applied the interest
- When we have `2 events` in a `Schedule Group` and one of them `fails` then the following event `will not trigger` and the `Schedule Group` will not run the `next day`.
- It will also stream an error even through Kafka to trigger a `workflow` to handle calculation this `manually`

## Calendar service
- Calendar service allows user to define lists of days such as bank holidays to be used by Vault components via Core API
- Calendar may emit periodic events to note a change in cycle, which are called `Calendar Periods`
- Components can use this `monotonically` (either never decrease or never increase) increasing period identifier to reference the `window of time` in which they have to process an `instruction`
- We can use `@requires(calendar="effective_date - 1 month")` to fetch `Calendar Events` information
- To use Calendar, 
  - First we need to use the endpoint `/v1/calendar` to create a calendar object `HOLIDAYS`
  - Then we create a Calendar Event object via `/v1/calendar-event` API for the `HOLIDAYS` calendar
  - And finally in Smart Contract, we can check if the posting is in the HOLIDAYS period to give the customer extra reward points
  ```python
    ph_period = vault.get_calendar_events(calendar_ids=["HOLIDAYS"])
    if date.today() in ph_period:
        reward_points *= 2
  ```

## Supervisor contracts
- Supervisor contracts allow multiple accounts to `communicate` and `share` common financial behavior
- The same way Vault accounts are backed by `Smart Contracts`, `Plans` are backed by `Supervisor Contracts`
- `Supervisor contracts` can `modify` or `extend` the behavior of `accounts` that join the `Plan`
- Supervisor contracts are useful in some use cases:
    - `Concentration Risk`: aggregated balance of a customer can be limited by the total balanaces of multiple accounts
    - `Offset Account`: change the interest rate on 1 account based on balance on another account
    - `Reward bundles`: rewards based on the total balances across multiple accounts
    - `Syndicate loan`: sharing profit on a bundle of loans with a third party
- Supervisee contracts (the account's smart contract) logic `holds no` Plan or supervison logic, instead the logic is captured in `Supervisor Contract Logic`
- The supervisor contracts can `intercept` and `override` any hooks of the supervisee contracts
- The supervisor contracts have access to balances and parameters of the supervisee contracts
- Plan parameters are equivalent to instance parameters, are stored in time-series and can be set during plan definition or updates

## Supervisor contract hooks
- There are 2 ways a Supervisor Contract execution can be triggered, either via Plan or Account
- When Plan Scheduled Event is triggered, all Supervisor Contracts of that Plan are executed and all supervisee's hooks are executed as well
    - `activation_hook`: via Plan (cannot execute supervisees)
    - `conversion_hook`: via Plan (cannot execute supervisees)
    - `scheduled_event_hook`: via Plan
- Another way is when an Account receives postings and its hooks are supervised by Plan's Supervisor Contract then both Account's and Plan's hooks are executed
    - `pre_posting_hook`: via Account
    - `post_posting_hook`: via Account
- There are 3 modes of supervision that define the behavior when a hook is supervised
    - `UNSUPERVISED`: only account's hook is executed, Plan's hook is ignored
    - `INVOKED`: the account's hook is executed first, then the directive is passed to Plan's hook to execute. For Supervisor `scheduled_event_hook`, if `supervisee_hook_directives` is required then it's automatically `INVOKED`
    - `OVERRIDE`: only the Plan's hook is executed, account's hook is ignored
- Depends on the version, there are 3 hooks that can be overriden by supervisor contracts
- In docs v4
    - scheduled_event_hook
    - post_posting_hook
    - pre_posting_hook
- Also in docs v3:
    - scheduled_code
    - post_posting_code
    - pre_posting_code
- When a smart contract is supervised, its directives are not immediately instructed but intercepted by the supervisor contract and it decides whether to alter them or not
```python
supervised_smart_contracts = [
    SmartContractDescriptor(
        alias = 'alias1',
        smart_contract_version_id = '{smart_contract_version_id}',
        supervise_post_posting=True
    )
]
@requires(balances='latest live', scope='all')
def post_posting_hook(vault, hook_arguments):
```

## Example - Offset Mortgage
- Given a Mortgage Smart Contract with schedules: transfer_due and yearly_fee
- A Saving Account Smart Contract with schedules: accrue_interest and apply_interest
- We create a Supervisor Contract version from a Supervisor Contract
- From the Supervisor Contract, we create a Plan, which is an instance of Supervisor Contract version
- We create a Plan Update to associate 2 accounts with the plan so the scheduled_code hook can be overwritten
- We can implement logic such as:
    - Reducing the interest rate of Mortgage Account based on the balances of Saving Account
    - Increasing the interest rate of the Saving Account based on how large the Mortgage is

## Versioning and Conversion
- Any change made to the Smart Contract requires a version change
- Changes to parameters do not require version changes
- Account upgrade can be done via Core API or CLU
- We can upgrade all accounts of 1 version to a new version or we can upgrade individual 
- When upgrading and the account has parameters that don't exist in the previous version then they will take `default_value`
- When a new version is deployed, it becomes the `current version`, if no version is deployed then it stays with the existing version
- No new version of balance is created or removed during conversion
- If we fetch a balance that only exists in the new version, Vault will return 0
- If we accept a posting to a balance that doesn't exist, Vautl will create it
- However for schedules, when we upgrade to a new version, all schedules are recreated by default
- We can specify to retain the old schedules but we need to make sure the schedule name and event groups stay the same

## Best practices
- Only use Shape if you have multiple params that have the same properties
- Use Money type for all interaction with currency, postings and transfers
- Client Transaction ID should be unique and easy to trace, a best practice is to include `hook_execution_id`
- Unless there is a specific requirement for restriction, otherwise don't use restriction
- Use instruction details to include metadata as per client requirement such as `Description` and `Event`
- Keep the number of `Posting Instruction Batches` minimum, ideally 1, if we have to have multiple PIB, add an index to the `client_batch_id`
- It's useful to link the triggering `PIB` to the `PIB` that is generated by `post_posting_hook` via the `batch_details`

# Consuming and Storing data
## Data and Data consumption
- There are 3 types of data:
  - Reference data: code table with a code column and a description column
  - Master data: unique data that describes core entities like customers, suppliers, products...
  - Event driven data: data generated by even-driven applications
- Vault relies on 2 components to store data: PostgreSQL and Kafka
## Encryption and Key management
- Entry points communicate via HTTPS
- Internally Vault uses mTLS and Istio
- Data is encrypted at rest at many levels (EBS, S3 bucket, RDS)
- Hashicorp is the central secrets manager

## Data privacy, aduit, compliance and regulation
- Vault has no access to the client's customer or employee data
- Business logic log are recorded in ledger
- Thought Machine is CEP, SOC2 certified and GDPR compliant
- Finance regulations are not applicable to TestMessage

## Application security
- SAML is used to provide user access to Ops Dashboard
- Service Account is provided to grant access to API
- Audit API is used to extract usage logs

## Integration
- Vault integrations offers client to integrate with selected vendors to cover some use cases
- These integrations contain Thought Machine approved code, clear and up-to-date documentation
- However these are not production ready and not maintained by Thought Machine on behalf of a client
- Only `Thought Machine`, `Delivery Partners` and `Partners` (training sandbox and integration sandbox) can build integration
- To validate the integration, Thought Machine requires
  - Use cases
  - Architecture diagram
  - Source code
  - Complete integration guide
  - Relevant assets
- Then Thought Machine can offer `Integration Library` built by the `Inception Team` or `vendors` or client can build their own bespoke applications to sastify their use cases
- Customers should check the existing Integrations first before proposing their own
- Integrations are free-of-charge

## The Integration Library
- Banking ecosystem is divided into 6 segments: Moving Money, Data & analytics, Customer Knowledge, Customer Channels, Infra and Middileware
- The Integration Library contains integrations built by Inception Team or vendors and are well-documented and updated with every major release
- Integration Library can be accessed via `Partner Enablement Portal`

## Data Analytics - Google BigQuery
- Integrate using Streaming API, events are streamed out of dedicated topics, transformed and stored in BigQuery tables
- For Operation reconciliation, we check how many accounts loaded into BigQuery in the chosen timeframe then use Core API to check if everything has been loaded
- For Financial reconciliation, we sum up the total posting amount in BigQuery and compare it to the ledger

## Customer Knowledge - Moneythor
- Moneythor is a orchestration engine that sits between the data source and the customer channels to provide peronalized customer experience in near real-time using Machine Learning and behavior science 
- Events from dedicated topics are transformed and streamed to Moneythor

## Moving Money - Marqeta
- Card issuing platform that connects many schemes and companies such as Mastercard and visualization
- Can update customer's balance right after spending 
- Integrated with Vault directly by mapping the Marqueta JIT requests to Posting Instruction types which allow to hold and release funds

## Customer Interaction - Ezbob
- Small business digital lending that allows fully or partially automated credit decisioin with funds released the same day
- This helps increase customer engagement, faster onboarding and reduce manual processing
- Integrated using Vault API

# Smart Contract from Requirements to Delivery 
## Overview of Requirement gathering process
- There are a few steps in Requirement gathering
- `Product Discovery`: conceptualization, determine product features, use `Product Library Gap` template if similar to a product in the library, Moscow prioritization `must have`, `should have` and `could have` category
- `Product scope and planning`: define scope, prioritize, signoff scope with stakeholders
- `Requirement refinement`: create epics, stories, acceptance criteria, sprint planning
- `Product build and test`: each story is developed and tested, test scripts are written, documented and demo
- `Change requests`: bank raises a change request, refine the request, create new epics and user stories, develop
## Best practice - Gap analysis
- `Gap analysis` is the process that find gaps and similarities between the product and a product in the Product Library
- Most products in the library have a `gap template`
- We can determine which features are already implemented, which require slight adjustmenet
- `Gap analysis` allows us to focus on the features and not go sidetrack outside of Vault's capabilities
- It also provides a simple scope and summary and allows stakeholders to have a complete view of the broken down features
- Steps to do `gap analysis`:
  - Determine if the feature is `within scope`
  - Determind if it's a `functionality gap` or it's `already implemented` in the library
  - Determine the `Moscow` priority of each item
  - Determine the feature details, user stories and acceptance criteria for each feature, work with the bank when each feature and criteria is `individually confirmed`
## Best practice - Scope and Planning
- There are 3 outcomes we want from scoping the project
  - Finalized scope based on priorities
  - Initial plan on which features are developed in each sprint
  - Signoff
## Best practice - Requirement refinement
- A user story should contains 3 statements
  - High level statement: a sentence that describes what the bank wants
  - Business value statement: a sentence that describe the value that the user story provides
  - Acceptance criteria: describes the details `Given`, `When`, `Then`, `Example`
- We need to consider if the feature should be handled by Vault or external system, does Vault offer a better solution for this, how can it be tested and what would trigger this feature from outside.
## Best practice - Build and test 
- The BAs need to ensure that developers understand and utilize Smart Contract correctly
- Features need to be tested again acceptance criteria
- BAs are also responsible for demo and iteratively adjust scope and ACs in response to new information
- BAs raise and manage change requests
- When using Smart Contracts, we need to determine what belongs to a smart contract: `financial logic`, `account lifecycle` and `financial side effects`
- What to keep out of Smart Contracts including:
  - `Decisioning`: decisions that tweak the financial behavior such as what interest rate to apply on an account or product should be determined externally so that Smart Contracts and digest that decision, other complex decisioining that is based on mulitple factors like KYC 
  - `External API calls`
  - `Hierachy and categorization`: parameters should not be used to categorized products or to put it in a hierachy 
- Examples of cases that Smart Contracts can handle:
  - Interest accrued/paid
  - Service fee and overdraft
  - Rebalancing to different account addresses based on business rules
  - Base rate and LIBOR rate changes
  - Tier-based accounts for interest calculations
- Example of cases that Smart Contracts should not handle:
  - Complex fee structures not linked to postings
  - Eligibility rules for account opening
  - Statement generation
  - Complex pricing structures
  - Logic for mass data breach, KYC, complex decisioning
# Real time analytics
## Architecture
- There are 2 options when it comes to architecture: `Lambda` and `Kappa`
## Lambda architecture
- Uses 2 separate data processing systems for different type of workloads
- `Batch data processing`: processes data in `large batch` and put them in a `centralized` data warehouse or file system
- `Stream processing`: processes data in `real time` and put them in `distributed` file systems
- Lambda architecture typically has `3 layers`:
  - Data ingestion layer
  - Simultanious batch (store in warehouse or ODS) and real time processing (Kafka, Storm, Flink)
  - Serving data to users in real-time, users can use SQL or HiveQL

## Kappa architecture
- Treats everything as streams
- Only has 1 layer which:
  - Ingests data and feeds data to both `Real time processor` and `Long term storage` in real time
  - `Real time processor` running on Kafka to stream data to downstream system like BI, fraud detection...
  - A full copy of all data is also captured at `Long term storage` like data lake or warehouse for BI, AI...
  - No separate serving layer, the streaming layer also serve data via SQL
- More suitable for modern cloud environments than Lambda

## Comparison
- Scalability: both `Kappa` and `Lambda` are scalable
- Fault tolerance: `Kappa` is `more fault-tolerant` than `Lambda` thanks to its 1 layer architecture
- Complexity: `Kappa` is `simpler` while `Lambda` has `3 layers` which is more complex
- Data management: `Kappa` is more `streamlined` regarding data pipelines, it can also handle large amount of data just like `Lambda`
- While popular, Lambda batch processing can only process `bound data` which is finite and predictable while nowadays data is more `unbound`

## Streaming API overview
- Vault has 2 types of API that enable interaction with the system and data within it
- When a posting is made to Vault, Streaming API streams the event to downstream systems where it's enriched and fed to machine learning models and to notification engine. The notif engine distribute the event to various channels
- Vault supports Fraud detection models built by third party, rule-based, heuristic or learned models are treated and deployed the same to microservices