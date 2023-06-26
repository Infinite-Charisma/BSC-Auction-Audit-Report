# BSC Auction Audit Report

# Summary
This is the auction smart contract that users deposit BNB or USDT to the program and then after it is finished, they can claim the ERC20 token according to the deposit amount.

In this program, the Owner will set the total amount to distribute and the minimal amount to raise.
Then the users deposit the BNB or USDT.
And after the auction is finished, users can claim the ERC20 token according to the deposit amount 
`amountOfTokens = supplyToDistribute * userDepositedUSD / totalRaisedUSD`

# In scope
The single-file smart contract.
<a href="https://gist.github.com/yuriy77k/edf8b3bcddbc3d43967f5765edf4727e">testAuction.sol</a>

# Findings
In total, issues were reported including:
- 5 High severity issues
- 2 Medium severity issues
- 6 Low severity issues
- 3 Informal severity issues

# Severity Issues
## 1. Decimal mismatch.
<b>Severity: high</b>

<b>Description</b>

To calculate the total deposit amount by USD, in the smart contract this is determined by decimal 18 but for the OraclePrice, decimal 8.

```ps
Auction.auctionEnd() (line#92-99)
  uint256 totalRaisedUSD = totalUSDT + totalBNB * bnbPrice;(line#97)
```

For example: 
```
`totalBNB` = 10e18, `totalUSDT` = 1000e18  in other words, 10BNB and 1000$ -> totally, 1000+2619 = 3619$
`totalRaisedUSD` = 1000e18 + 261.9e8*10e18 = 261901000$ this is the unexpected giant value
```

## 2. Protected Function
<b>Severity: high</b>

<b>Description</b>

The function `refundAll()` should be called by only owner.

```ps
Auction.refundAll() (line#62-77)
```

This function should be (onlyOwner) function because if all the deposit amount is smaller than `softCap` then the function caller should refund to the bidders.
In other words, the caller should pay for transfer gas fee. This is the owner function so should add `onlyOwner` modifier on the function.

## 3. Unchecked transfer
<b>Severity: high</b>

<b>Description</b>

The return value of an external transfer/transferFrom call is not checked.

```ps
  Auction.setCaps(uint256,uint256) (line#53-59) 
ignores return value by IERC20(offeringToken).transferFrom(msg.sender,address(this),_supplyToDistribute) (line#56)
  Auction.refundAll() (line#62-77) 
ignores return value by IERC20(USDT_TOKEN).transfer(addressList[i],user.usdtAmount) (line#73)
  Auction.claimTokens() (line#79-90) 
ignores return value by IERC20(offeringToken).transfer(msg.sender,amountOfTokens) (line#88)
  Auction.depositAuction(uint256,bool) (line#112-127) 
ignores return value by IERC20(USDT_TOKEN).transferFrom(msg.sender,address(this),_amount) (line#120)
```

## 4. Reentrancy vulnerabilities
<b>Severity: high</b>

<b>Description</b>
Detection of the <a href="https://github.com/trailofbits/not-so-smart-contracts/tree/master/reentrancy">reentrancy bugs</a>. Do not report reentrancies that don't involve BNB (see `reentrancy-no-eth`)

```ps
Reentrancy in Auction.refundAll() (line#62-77):
        External calls:
        - IERC20(USDT_TOKEN).transfer(addressList[i],user.usdtAmount) (line#73)
        External calls sending BNB:
        - address(addressList[i]).transfer(user.bnbAmount) (line#69)
        State variables written after the call(s):
        - user.usdtAmount = 0 (line#74)
```

## 5. Value check
<b>Severity: high</b>

<b>Description</b>
Deposit amount should be the same as `msg.value`. In this smart contract, there is no check function to compare the deposit amount to store on the blockchain and `msg.value` which is actually sent. They shoudl be the same.

```ps
Value check missing in Auction.depositAuction() (line#112-128):
       _amount should be the same as msg.value
```

## 6. Reentrancy vulnerabilities
<b>Severity: medium</b>

<b>Description</b>
Detection of the <a href="https://github.com/trailofbits/not-so-smart-contracts/tree/master/reentrancy">reentrancy bugs</a>. Do not report reentrancies that don't involve BNB (see `reentrancy-eth`)

```ps
Reentrancy in Auction.claimTokens() (line#79-90):
        External calls:
        - IERC20(offeringToken).transfer(msg.sender,amountOfTokens) (line#88)
        State variables written after the call(s):
        - user.tokenAmount = amountOfTokens (line#89)
Reentrancy in Auction.depositAuction(uint256,bool) (line#112-127):
        External calls:
        - IERC20(USDT_TOKEN).transferFrom(msg.sender,address(this),_amount) (line#120)
        State variables written after the call(s):
        - user.usdtAmount += _amount (line#122)
```

## 7. Invalid Set Function
<b>Severity: medium</b>

<b>Description</b>
In this smart contract, the variable set function is not correct. It should be `=`.

```ps
Invalid in Auction.setCaps() (line#53-59):
        Set function operator should be `=` not `+=`:
        - supplyToDistribute += _supplyToDistribute; (line#57)
```


## 8. Double set
<b>Severity: low</b>

<b>Description</b>
In this smart contract, the variable `totalRaisedUSD` is set as `public`. But it is set 2 times.

```ps
Auction.claimTokens() (line#79-90)
        uint256 totalRaisedUSD = totalUSDT + totalBNB * bnbPrice;  (line#84)
Auction.auctionEnd() (line#92-99)
        uint256 totalRaisedUSD = totalUSDT + totalBNB * bnbPrice;  (line#97)
```

## 9. Missing events arithmetic
<b>Severity: low</b>

<b>Description</b>
Detect missing events for critical arithmetic parameters.

```ps
Auction.setCaps(uint256,uint256) (line#53-59) should emit an event for:
        - supplyToDistribute += _supplyToDistribute (line#57)
        - softCap = _softCap (line#58)
```

## 10. Missing zero address validation
<b>Severity: low</b>

<b>Description</b>
Detect missing zero address validation.

```ps
Auction.constructor(address,uint256,uint256)._offeringToken (line#41) lacks a zero-check on :
                - offeringToken = _offeringToken (line#43)
```

## 11. Calls inside a loop
<b>Severity: low</b>

<b>Description</b>
Calls inside a loop might lead to a denial-of-service attack.

```ps
Auction.refundAll() (line#62-77) has external calls inside a loop: address(addressList[i]).transfer(user.bnbAmount) (line#69)
Auction.refundAll() (line#62-77) has external calls inside a loop: IERC20(USDT_TOKEN).transfer(addressList[i],user.usdtAmount) (line#73)
```

## 12. Reentrancy vulnerabilities
<b>Severity: low</b>

<b>Description</b>
Detection of the <a href="https://github.com/trailofbits/not-so-smart-contracts/tree/master/reentrancy">reentrancy bugs</a>. Do not report reentrancies that don't involve BNB (see `reentrancy-eth`, `reentrancy-no-eth`)

```ps
Reentrancy in Auction.depositAuction(uint256,bool) (line#112-127):
        External calls:
        - IERC20(USDT_TOKEN).transferFrom(msg.sender,address(this),_amount) (line#120)
        State variables written after the call(s):
        - totalUSDT += _amount (line#121)
Reentrancy in Auction.setCaps(uint256,uint256) (line#53-59):
        External calls:
        - IERC20(offeringToken).transferFrom(msg.sender,address(this),_supplyToDistribute) (line#56)
        State variables written after the call(s):
        - softCap = _softCap (line#58)
        - supplyToDistribute += _supplyToDistribute (line#57)
```

## 13. Block timestamp
<b>Severity: low</b>

<b>Description</b>
Dangerous usage of `block.timestamp`. `block.timestamp` can be manipulated by miners.

```ps
Auction.setCaps(uint256,uint256) (line#53-59) uses timestamp for comparisons
        Dangerous comparisons:
        - require(bool,string)(block.timestamp < startTime,auction started) (line#55)
Auction.auctionEnd() (line#92-99) uses timestamp for comparisons
        Dangerous comparisons:
        - require(bool,string)(block.timestamp >= endTime,Auction not end) (line#93)
Auction.checkConditions(uint256) (line#101-105) uses timestamp for comparisons
        Dangerous comparisons:
        - require(bool,string)(block.timestamp > startTime && block.timestamp < endTime,not auction time) (line#102)
```

## 14. Boolean equality
<b>Severity: informational</b>

<b>Description</b>
Detects the comparison to boolean constants.

```ps
Auction.checkConditions(uint256) (line#101-105) compares to a boolean constant:
        -require(bool,string)(isClosed == false,funding is reached) (line#104)
```

## 15. Dead-code
<b>Severity: informational</b>

<b>Description</b>
Functions that are not sued.

```ps
Context._msgData() (node_modules/@openzeppelin/contracts/utils/Context.sol#21-23) is never used and should be removed
```

## 16. Reentrancy vulnerabilities
<b>Severity: informational</b>

<b>Description</b>
Detection of the <a href="https://github.com/trailofbits/not-so-smart-contracts/tree/master/reentrancy">reentrancy bugs</a>. Only report reentrancy that is based on `transfer` or `send`.

```ps
Reentrancy in Auction.refundAll() (line#62-77):
        External calls:
        - address(addressList[i]).transfer(user.bnbAmount) (line#69)
        State variables written after the call(s):
        - user.bnbAmount = 0 (line#70)
        - user.usdtAmount = 0 (line#74)
```

# Conclusion
No critical vulnerabilities were detected. 
But some reported issues are very important to calculate and execute. Also should add some check functions to compare the real values.
The auction smart contract can satisfy the main goal.

```
This smart contract's internal logic:
  when you deploy the contract you should set the offering token's address, start time and end time.
  after deploy and before start, owner should set the supplyToDistribute and softCap.
  after start, users can deposit the BNB or USDT - the deposit amount will be stored on blockchain.
  after end, anyone can finish the auction and check if it is closed and success.
  if the desposit amount is over the softCap, users can claim the offeringToken according to the depositAmount
      but not owner can refundAll to the bidders
  
  uint256 amountOfTokens = supplyToDistribute * userDepositedUSD / totalRaisedUSD;
```
It is highly recommended to complete a bug bounty before use.
