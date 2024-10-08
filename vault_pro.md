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