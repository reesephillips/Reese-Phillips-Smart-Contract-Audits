# Core Contracts - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Stakers who stake at the end of the period receive the same rewards as those who stake at the beginning of the period](#H-01)
    - ### [H-02. Boost delegation will not work because `userBoosts` is updated incorrectly when delegating boosts](#H-02)
    - ### [H-03. User can delegate boost to themselves using different addresses an unlimited amount of times](#H-03)
    - ### [H-04. FeeCollector does not utilize the `deposit` function in the Treasury contract which could lead to stuck fees](#H-04)
- ## Medium Risk Findings
    - ### [M-01. Gauge reward periods are not enforced correctly due to incorrect timing logic](#M-01)
    - ### [M-02. Wrong price for house could be sent if responses are not recieved in order from Oracle](#M-02)
    - ### [M-03. Inconsistent tracking of total debt and liquidity leads to incorrect utilization rate calculation](#M-03)
    - ### [M-04. `lastUpdateTimestamp` is returned from oracle but never checked which could lead to stale NFT prices](#M-04)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: Regnum Aurum Acquisition Corp

### Dates: Feb 3rd, 2025 - Feb 24th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-02-raac)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 4
- Medium: 4
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Stakers who stake at the end of the period receive the same rewards as those who stake at the beginning of the period            



## Summary

Users have the ability to stake some staking tokens in the Gauge contracts to be eligible for rewards. The problem is that late stakers are rewarded the same as early stakers leaving no incentive for locking tokens for an extended period of time. Users will just stake at the end of the period to claim the maximum amount of rewards.

## Vulnerability Details

Users can stake tokens through the [stake](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/governance/gauges/BaseGauge.sol#L261) function in `BaseGauge` to be eligible for rewards. The problem is that late stakers and early stakers can earn the same amount of rewards. If we look in the [earned](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/governance/gauges/BaseGauge.sol#L583) function, it only uses the current reward per token amount. It does not keep a snapshot of when the staker staked within the period. They can come in and stake when reward per token is at its maximum amount at the end of the period and instantly be eligible for all rewards.

```Solidity
/**
     * @notice Calculates earned rewards for account
     * @param account Address to calculate earnings for
     * @return Amount of rewards earned
     */
    function earned(address account) public view returns (uint256) {
        return (getUserWeight(account) * 
            (getRewardPerToken() - userStates[account].rewardPerTokenPaid) / 1e18
        ) + userStates[account].rewards;
    }
```

## Impact

No incentive for staking longer durations

## POC

The following demonstrates one staker staking at the beginning of the period and another right at the end. They both end up earning the same amount of rewards.

Add the following test to `BaseGauge.test.js` and run `npx hardhat test test/unit/core/governance/gauges/BaseGauge.test.js`

```Solidity
it("should demonstrate equal rewards for early and late stakers", async () => {
            // User1 stakes at the start of the period
            await baseGauge.connect(user1).stake(ethers.parseEther("100"));
            
            // Move to almost end of period (6.9 days)
            await time.increase(6.9 * DAY);
            
            // User2 stakes same amount near end of period
            await baseGauge.connect(user2).stake(ethers.parseEther("100"));
            
            // Move past minimum claim interval
            await time.increase(1.2 * DAY);
            
            // Both users claim rewards
            await baseGauge.connect(user1).getReward();
            await baseGauge.connect(user2).getReward();
            
            // Get final reward balances
            const user1Rewards = await rewardToken.balanceOf(user1.address);
            const user2Rewards = await rewardToken.balanceOf(user2.address);
            
            // Calculate reward difference (should be significant but isn't)
            const rewardDiff = user1Rewards - user2Rewards;
            console.log("User1 rewards:", ethers.formatEther(user1Rewards));
            console.log("User2 rewards:", ethers.formatEther(user2Rewards));
            console.log("Reward difference:", ethers.formatEther(rewardDiff));
            
            // Despite User1 staking for 6.9 days longer, rewards are nearly identical
            expect(rewardDiff).to.be.lt(ethers.parseEther("0.1")); // Difference is negligible
        });
    });
```

Receive the following output

```Solidity
 Reward Distribution Vulnerability
User1 rewards: 900.000000000000107142
User2 rewards: 900.000000000000107142
Reward difference: 0.0
```

## Tools Used
Manual Review
## Recommendations
Create snapshots for stakers and reward based on staking duration

## <a id='H-02'></a>H-02. Boost delegation will not work because `userBoosts` is updated incorrectly when delegating boosts            



## Summary

When a user wishes to delegate their boost to another user, their corresponding `userBoosts` mapping is updated. The issue is the delegation functions update the pool address in the mapping to the user they are delegating to so the corresponding pool they are receiving a boost for will be lost.

## Vulnerability Details

A user chooses to delegate their boost to another user and calls [delegateBoost](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/governance/boost/BoostController.sol#L212). The delegation storage variable is initialized with the `msg.sender` and `to` parameter. The problem is this `to` parameter that is supposed to represent who the user is delegating to actually is in the spot in the mapping where the pool address should be.


```Solidity
function delegateBoost(
        address to,
        uint256 amount,
        uint256 duration
    ) external override nonReentrant {
        if (paused()) revert EmergencyPaused();
        if (to == address(0)) revert InvalidPool();
        if (amount == 0) revert InvalidBoostAmount();
        if (duration < MIN_DELEGATION_DURATION || duration > MAX_DELEGATION_DURATION) 
            revert InvalidDelegationDuration();
        
        uint256 userBalance = IERC20(address(veToken)).balanceOf(msg.sender);
        if (userBalance < amount) revert InsufficientVeBalance();
        
@>        UserBoost storage delegation = userBoosts[msg.sender][to];
        if (delegation.amount > 0) revert BoostAlreadyDelegated();
        
        delegation.amount = amount;
        delegation.expiry = block.timestamp + duration;
        delegation.delegatedTo = to;
        delegation.lastUpdateTime = block.timestamp;
        
        emit BoostDelegated(msg.sender, to, amount, duration);
    }
```

If we look at how the `userBoosts` variable is declared, we can see the intended behavior.


```Solidity
// @notice Maps user addresses to their boost information for each pool
    mapping(address => mapping(address => UserBoost)) private userBoosts; // user => pool => boost
```

## Impact
Unable to delegate boost to a user for a given pool
## Tools Used
Manual Review
## Recommendations
There doesnt need to be a new UserBoost mapping created. There is already a `delegatedTo` variable as part of the `UserBoost` object.
## <a id='H-03'></a>H-03. User can delegate boost to themselves using different addresses an unlimited amount of times            



## Summary

In `BoostController` users have the option to delegate their boost to another user. The problem is they can delegate to an unlimited amount of different addresses. A user could create several different addresses and delegate boost to themselves.

## Vulnerability Details

Users have the ability to delegate their boost amount through the [delegateBoost](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/governance/boost/BoostController.sol#L212) function in the `BoostController`. A problem arise due to the fact that the user can delegate an unlimited amount. The only check is that they have not already delegated to the address. An attacker could create several addresses and continuously delegate their boost amount to each of these addresses. They would then receive significantly more rewards than intended.

```Solidity
/**
     * @notice Delegates boost from caller to another address
     * @param to Address to delegate boost to
     * @param amount Amount of boost to delegate
     * @param duration Duration of the delegation in seconds
     * @dev Requires sufficient veToken balance and no existing delegation
     */
    //@audit-data user can delegate to an unlimited amount of addresses
    function delegateBoost(
        address to,
        uint256 amount,
        uint256 duration
    ) external override nonReentrant {
        if (paused()) revert EmergencyPaused();
        if (to == address(0)) revert InvalidPool();
        if (amount == 0) revert InvalidBoostAmount();
        if (duration < MIN_DELEGATION_DURATION || duration > MAX_DELEGATION_DURATION) 
            revert InvalidDelegationDuration();
        
        uint256 userBalance = IERC20(address(veToken)).balanceOf(msg.sender);
        if (userBalance < amount) revert InsufficientVeBalance();
        
        UserBoost storage delegation = userBoosts[msg.sender][to];
        if (delegation.amount > 0) revert BoostAlreadyDelegated();
        
        delegation.amount = amount;
        delegation.expiry = block.timestamp + duration;
        delegation.delegatedTo = to;
        delegation.lastUpdateTime = block.timestamp;
        
        emit BoostDelegated(msg.sender, to, amount, duration);
    }
```

## POC

The following POC demonstrates delegating the same amount to two different users. The user should only be entitled to delegate the amount they own.

Add the following test to `BoostController.test.js` and run npx hardhat test test/unit/core/governance/boost/BoostController.test.js

```Solidity
describe("Delegation Unlimited", () => {
        it("User can delegate unlimited amount to several different users", async () => {
            const amount = ethers.parseEther("500");
            const duration = 7 * 24 * 3600; // 7 days

            await expect(
                boostController.connect(user1).delegateBoost(user2.address, amount, duration)
            ).to.emit(boostController, "BoostDelegated")
             .withArgs(user1.address, user2.address, amount, duration);

             await expect(
                boostController.connect(user1).delegateBoost(user3.address, amount, duration)
            ).to.emit(boostController, "BoostDelegated")
             .withArgs(user1.address, user3.address, amount, duration);

            const delegation = await boostController.getUserBoost(user1.address, user2.address);
            expect(delegation.amount).to.equal(amount);
            expect(delegation.delegatedTo).to.equal(user2.address);

            const delegation2 = await boostController.getUserBoost(user1.address, user3.address);
            expect(delegation2.amount).to.equal(amount);
            expect(delegation2.delegatedTo).to.equal(user3.address);
        });
    });
```

## Tools Used
Manual Review

## Recommendations
Only allow the user to delegate the amount of boost they entitled to in total
## <a id='H-04'></a>H-04. FeeCollector does not utilize the `deposit` function in the Treasury contract which could lead to stuck fees            



## Summary

`FeeCollector` distributes the fees to a treasury address through a direct transfer call. This will bypass the accounting logic in [deposit](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/collectors/Treasury.sol#L46) which can cause the call to [withdraw](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/collectors/Treasury.sol#L64) to revert.

## Vulnerability Details

`FeeCollector` handles the distribution of fees to different actors. One of these actors is the project treasury. If we take a look at [\_processDistributions](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/collectors/FeeCollector.sol#L423), it makes a direct transfer call to the treasury address. This is problematic when we look at the deposit logic in the treasury contract. There is an accounting variable `_balances` that keeps track of the amount of a token deposited into the treasury.


```Solidity
    function deposit(address token, uint256 amount) external override nonReentrant {
        if (token == address(0)) revert InvalidAddress();
        if (amount == 0) revert InvalidAmount();
        
        IERC20(token).transferFrom(msg.sender, address(this), amount);
        _balances[token] += amount;
        _totalValue += amount;
        
        emit Deposited(token, amount);
    }
```

When a manager attempts to withdraw this token, there is a check in `withdraw` to ensure there is a sufficient amount of the token through the `_balances` variable. The problem is this variable does not account for tokens transferred directly to this contract. If all the funds are being directly transferred to this contract, that check will revert resulting in all of the tokens being stuck.


```Solidity
   function withdraw(
        address token,
        uint256 amount,
        address recipient
    ) external override nonReentrant onlyRole(MANAGER_ROLE) {
        if (token == address(0)) revert InvalidAddress();
        if (recipient == address(0)) revert InvalidRecipient();
        if (_balances[token] < amount) revert InsufficientBalance();
        
        _balances[token] -= amount;
        _totalValue -= amount;
        IERC20(token).transfer(recipient, amount);
        
        emit Withdrawn(token, amount, recipient);
    }
```

## Impact
Loss a fees for the project treasury

## Tools Used
Manual Review

## Recommendations
Deposit the fees into the treasury contract rather than doing a direct transfer.
    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Gauge reward periods are not enforced correctly due to incorrect timing logic            



## Summary

Users are able to earn rewards during either 1 week or 1 month periods through the Gauge contracts. There is an issue with how the period duration is tracked however that allows users to earn rewards past this timeframe for an indefinite amount.

## Vulnerability Details

The [\_updateReward](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/governance/gauges/BaseGauge.sol#L167) function is called through the `updateReward` modifier to recalculate the amount of rewards a user should be eligible for. `rewardPerTokenStored` is used to track the changes in the reward per token over the 1 week or 1 month duration of the period and is updated through [getRewardPerToken](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/governance/gauges/BaseGauge.sol#L568).

```Solidity
function getRewardPerToken() public view returns (uint256) {
        if (totalSupply() == 0) {
            return rewardPerTokenStored;
        }

        return rewardPerTokenStored + (
            (lastTimeRewardApplicable() - lastUpdateTime) * rewardRate * 1e18 / totalSupply()
        );
    }
```

A problem arises though in the call to `lastTimeRewardApplicable`. This gets the latest applicable reward time and assigns it to `lastUpdateTime` but the check for `block.timestamp < periodFinish()` can never return false and therefore `lastUpdateTime` will always be the latest `block.timestamp` and the period will never end.


```Solidity
function lastTimeRewardApplicable() public view returns (uint256) {
        return block.timestamp < periodFinish() ? block.timestamp : periodFinish();
    }
```

This is because `periodFinish` just adds on the period duration of either 1 week or 1 month to the `lastUpdateTime` creating an infinite deadline that will never be reached.

```Solidity
function periodFinish() public view returns (uint256) {
        return lastUpdateTime + getPeriodDuration();
    }
```

## Impact
Reward periods are not enforced allowing users to accrue rewards indefinitely
## Tools Used
Manual Review
## Recommendations
Period tracking needs to be more dynamic 
## <a id='M-02'></a>M-02. Wrong price for house could be sent if responses are not recieved in order from Oracle            



## Summary

`RAACHousePriceOracle` sends a request to the DON when it wishes to receive that latest price of a house from the Oracle. The house id is set in [\_beforeFulfill](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/oracles/RAACHousePriceOracle.sol#L34) to keep track of which house price will be sent on the next response received. An issue arises because there is no guarantee the next response received will be for that house id.

## Vulnerability Details

A request is made to the DON for the price of a house. The house id is passed in as calldata which is set in `_beforeFulfill` and will be set on the next response through [\_processResponse](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/oracles/RAACHousePriceOracle.sol#L42).

```Solidity
/**
     * @notice Hook called before fulfillment to store the house ID
     * @param args The arguments passed to sendRequest
     */
    function _beforeFulfill(string[] calldata args) internal override {
        lastHouseId = args[0].stringToUint();
    }

    /**
     * @notice Process the response from the oracle
     * @param response The response from the oracle
     */
    function _processResponse(bytes memory response) internal override {
        uint256 price = abi.decode(response, (uint256));
        housePrices.setHousePrice(lastHouseId, price);
        emit HousePriceUpdated(lastHouseId, price);
    }
```

The problem is that there is no guarantee the next response will include the response for the corresponding house ID set. Take the following example:

1. Request A is sent for House ID 1 with `sendRequest()`
2. `_beforeFulfill()` sets `lastHouseId = 1`
3. Request B is sent for House ID 2
4. `_beforeFulfill()` sets `lastHouseId = 2`
5. Response for Request A arrives first
6. `_processResponse()` uses `lastHouseId` (which is now 2) to set the price
7. The price from Request A (meant for House 1) gets incorrectly set for House 2

## Impact
Incorrect price is set for the houses
## Tools Used
Manual Review
## Recommendations
Pass the house id along with the price in the response
## <a id='M-03'></a>M-03. Inconsistent tracking of total debt and liquidity leads to incorrect utilization rate calculation            



## Summary

Both the total liquidity and debt are used to calculate the utilization rate. The problem is that total debt is being scaled down when getting the total supply but total liquidity is not leading to an incorrect utilization rate calculation.

## Vulnerability Details

Lets first take a look at how total liquidity is tracked. A user calls [deposit](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/pools/LendingPool/LendingPool.sol#L225) and transfers a certain amount of a reserve asset. In the Reserve [deposit](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/libraries/pools/ReserveLibrary.sol#L347) function it passes the amount deposited into the [updateInterestRatesAndLiquidity](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/libraries/pools/ReserveLibrary.sol#L201) function where it is directly added to the liquidity amount and not scaled by the liquidity index.

```Solidity
function updateInterestRatesAndLiquidity(ReserveData storage reserve,ReserveRateData storage rateData,uint256 liquidityAdded,uint256 liquidityTaken) internal {
        // Update total liquidity
        if (liquidityAdded > 0) {
            reserve.totalLiquidity = reserve.totalLiquidity + liquidityAdded.toUint128();
        }
```

Now if we look at how `totalUsage` is tracked we first start at the [borrow](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/pools/LendingPool/LendingPool.sol#L353) function where the debt token is minted. `newTotalSupply` is returned which is then used as the `totalUsage` amount. When we look at how the [totalSupply](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/tokens/DebtToken.sol#L234) of the debt token is calculated, it returns the scaled version of it using the usage index.

```Solidity
/**
     * @notice Returns the scaled total supply
     * @return The total supply (scaled by the usage index)
     */
    function totalSupply() public view override(ERC20, IERC20) returns (uint256) {
        uint256 scaledSupply = super.totalSupply();
        return scaledSupply.rayDiv(ILendingPool(_reservePool).getNormalizedDebt());
    }
```
This is not consistent with how the liquidity is tracked. One is being scaled by its index and the other is not.
## Impact
Incorrect utilization rate calculation
## Tools Used
Manual Review
## Recommendations
Make the total debt and total liquidity consistent.
## <a id='M-04'></a>M-04. `lastUpdateTimestamp` is returned from oracle but never checked which could lead to stale NFT prices            



## Summary

The price of the NFT users deposit is used to calculate the total amount of collateral they have deposited. The values of these NFTs are fetched through a Chainlink price oracle. An issue arises because the timestamp of which these prices were last updated is never checked which could lead to a stale price.

## Vulnerability Details

A user wishes to withdraw their NFT but they must not be undercollateralized. The price of their assets are fetched through [getNFTPrice](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/pools/LendingPool/LendingPool.sol#L591) which will make a call to a Chainlink price oracle and add them all up together. The issue is that the `lastUpdateTimestamp` is never checked to ensure the price of the NFT isn't stale. A user may be denied a withdrawal or even be wrongfully liquidated because the current price of the NFTs are not up to date.

```Solidity
/**
     * @notice Gets the current price of an NFT from the oracle
     * @param tokenId The token ID of the NFT
     * @return The price of the NFT
     *
     * Checks if the price is stale
     */
    function getNFTPrice(uint256 tokenId) public view returns (uint256) {
        (uint256 price, uint256 lastUpdateTimestamp) = priceOracle.getLatestPrice(tokenId);
        if (price == 0) revert InvalidNFTPrice();
        return price;
    }
```

## Impact
Users can be wrongfully liquidated due to outdated prices or prevent withdrawal
## Tools Used
Manual Review
## Recommendations
Check the last update timestamp




