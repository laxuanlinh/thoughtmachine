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
    