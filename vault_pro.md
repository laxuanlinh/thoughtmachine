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