US1: create account
FC1: create 3-in-1 account
- Customer sends request to create a new 3-in-1 account
- Orchestration service calls `10x service`
- `10x service` calls 10x to create a new party if not exists
- `10x service` calls 10x to create subscriptions to 3 products
- `10x service` stores party ID and 3 subscription keys in own DB because 10x doesn't support fetching all subs of an account
- System responds with either success or error

FC2: show committed interest rate
- Customer sends a request to view committed interest rate
- `Orchestration service` calls `10x service` to check if 3-in-1 already exists, if yes then shows the interest rate from DB
- If not, `Orchestration service` calls `Auto Distribution service` to calculate committed rate
- `Auto Distribution service` calls the `Balance service` to get party balance and group balance to calculate committed rate
- `Orchestration service` returns the committed rate

US2: 
FC1: buy CD/investment
- Customers sends request to buy CD/investment
- `Orchestration service` calculates committed rate and shows to customer
- Customer confirms
- Orchestration service stores the committed rate in the DB
- Orchestration service calls `Auto Distribution service` to distribute balance
- `Auto Distribution service` calls SunTec to get the rate and supply amd calls `Balance service` to get party balance
- `Auto Distribution service` distributes the balance into CASA/CD/investment and calls `10x service` to do postings
- `Orchestration service` returns success or error

FC2: deposit to a zero balance account
- Customer deposits more money
- Orchestration service calls `Auto Distribution service` to distribute balance
- `Auto Distribution service` calls SunTec to get the rate and supply amd calls `Balance service` to get party balance
- `Auto Distribution service` distributes the balance into CASA/CD/investment and calls `10x service` to do postings
- `Orchestration service` returns success or error

US3: withdrawal
FC1: allow user to sell Cd and investment (do we really want to do this?)
FC2: customer doesn't have enough CASA to withdraw
- Customer sends request to withdraw money
- `Orchestration service` calls `Balance service` to calculate total balance including CD and investment
- Regardless of CASA is enough or not, `Orchestration service` calls `Auto distribution service` to calculate the "would-be distribution" after withdrawal to decide how many CD and investment to be sold to satisfy the max CASA and committed rate
- `Orchestration service` then sells CD and investment accordingly
- `Orchestration service` calls `10x service` to put all of the returned money to CASA then allow the customer to withdraw from CASA.
- The outstanding money should be max CASA and optimal CD and investment for the committed rate

US4: calculate interest ???
FC1: allow customer to get daily interest accrual
- Where to store accrued interest daily???

FC2: deposit interest to customer's account
- Where to deposit to? 
- If deposit to CASA, do we distribute money again into CD and investment???
- 