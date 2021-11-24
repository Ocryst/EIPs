---
eip: <to be assigned>
title: A Token Standard
description: A Vanilla Fungible Token Standard for Dao Equity Contracts
author: Doe, (@Ocryst), Liam, (@alawik>)
discussions-to: <URL>
status: Draft
type: Standards Track
category (*only required for Standards Track): ERC
created: 2021-11-24
---

## Abstract
The following standard allows for the implementation of a standard API for tokens within smart contracts to utilize DAO equity denominated options contracts. This utilizes the ERC20 token functionality and adds additional functions proposed as standard.
  
## Motivation
To standardize the use of DAO equity contracts in community directed team incentives. The treasury model popularized by olympusDao allows a dao to control its treasury and governance in unique ways. Through this token standard a dao with access to treasury and governance can integrate equity contracts to pay developers in a way that is controlled and aligned to community interest through equity in the protocol.

## Specification
This standard is backwards compatible with ERC-20, therefore, all ERC-20 functions can be called on an ERC-3643 token, the interfaces being compatible. But the functions are not implemented in the same way as a classic ERC-20 as This proposed standard is a allows treasury integration by a dao.

### Main functions
  
#### write

To be able to qualify for interaction with a dao equity contract as desribed in this proposal an address must by whitelisted by the dao initiating the contract.

Here is an example of `write` function implementation : 
  
```
    function write(
        address[] memory _recipients,
        uint256[] memory _amounts
    ) public returns (bool) {
        require(msg.sender == writer);

        require(_amounts.length > 0 && _recipients.length > 0);
        for (uint i = 0; i < _recipients.length; i++) {
            address recipient = _recipients[i];
            require(recipient != address(0));
            balanceOf[recipient] = _amounts[i];
            circulatingSupply +=  _amounts[i];
        }
    }
```

#### begin

To be able to perform a write to initiate the contract it must be called by the writer, theoretically a dao contract. This returns a bool which emits a start time for the contract. The initiation of time is important to setting deadlines and calculating bonus rewards.

Here is an example of `Begin` function implementation : 
  
```
    function begin() public returns (bool) {
        require(msg.sender == writer);
        startTime = block.timestamp;
        emit Start(startTime);
        return true;
    }
```
  
#### redeemable

The redeemable function checks to determine if the contract is redeemable. The date range of redeemability being set by the begin function.

Here is an example of `Begin` function implementation : 
  
```
    function redeemable(address account) public view returns (uint256) {
        if (started()) {
            return balanceOf[msg.sender].div(10000).mul(multiplier());
        } else return uint256(0);
    }
```
                                                
#### bonusTerms

In order to incentivise work paid for via equity contracts there is an optional bonus system integrated via the bonusTerms function. The function checks the work completion date and pays a bonus for early completion.

Here is an example of `bonusTerms` function implementation : 
  
```
    function bonusTerms() public view returns (uint256) {
        return expiryTime.sub(block.timestamp).div(bonusTerm);
    }
```
                                                
#### multiplier

A multiplier can be set and added to the bonusTerm payments
                                                
Here is an example of `bonusTerms` function implementation : 
  
```
    function multiplier() public view returns (uint256) {
        uint128 boom = uint128(bonusRate) ** uint128(bonusTerms());
        require(boom < type(uint128).max);
        return uint256(boom);
    }
```
  .
#### payment

The payment functions similarly to an options contract by multiplying against the strike.
                                                
Here is an example of `payment` function implementation : 
  
```
    function payment(uint256 amount) public view returns (uint256) {
        return amount.mul(strike);
    }
```
  
#### started

The started function when called will return the block timestamp of start to check if the equity contract has started.
                                                
Here is an example of `started` function implementation : 
  
```
    function started() public view returns (bool) {
        if (startTime == uint256(0)) return false;
        return block.timestamp > startTime ? true : false;
    }
```
  
#### expired

The expired function when called will return the block timestamp of start to check if the equity contract has expired.
                                                
Here is an example of `expired` function implementation : 
  
```
    function expired() public view returns (bool) {
        return block.timestamp < expiryTime ? true : false;
    }
```

#### exercise

The eexercise function checks the balance of the address then takes the returned values from `transferFrom`, `underwrite`, and `transfer` before emitting and `exercised` event to the blockchain as true.
                                                
Here is an example of `exercise` function implementation : 
  
```
    function exercise(uint256 amount) public returns (bool) {
        require(balanceOf[msg.sender] >= amount);
        require(IERC20(principle).balanceOf(msg.sender) >= payment(amount));

        balanceOf[msg.sender] -= amount;
        
        // This is safe because the sum of all user
        // exercised can't exceed type(uint256).max!
        unchecked {
            exercised[msg.sender] += amount;
        }
        
        IERC20(principle).transferFrom(msg.sender, writer, payment(amount));
        IWriter(writer).underwrite(amount);
        IERC20(principle).transfer(msg.sender, amount);

        emit Exercised(msg.sender, amount);
        return true;
    }
```       
  
#### exerciseFrom

The exerciseFrom function checks the allowance of the sender and returnsed a bool while emitting an `exercised` event to the blockchain.
                                                
Here is an example of `exerciseFrom` function implementation : 
  
```
    function exerciseFrom(address from, uint256 amount) public returns (bool) {
        if (allowance[from][msg.sender] != type(uint256).max) {
            allowance[from][msg.sender] -= amount;
        }

        balanceOf[from] -= amount;

        // This is safe because the sum of all user
        // exercised can't exceed type(uint256).max!
        unchecked {
            exercised[from] += amount;
        }

        emit Exercised(from, amount);
        return true;
    }
```  
  

  
  
                                           
## Rationale
The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages.

## Backwards Compatibility
All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.

## Test Cases
Test cases for an implementation are mandatory for EIPs that are affecting consensus changes.  If the test suite is too large to reasonably be included inline, then consider adding it as one or more files in `../assets/eip-####/`.

## Reference Implementation
An optional section that contains a reference/example implementation that people can use to assist in understanding or implementing this specification.  If the implementation is too large to reasonably be included inline, then consider adding it as one or more files in `../assets/eip-####/`.

## Security Considerations
All EIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. EIP submissions missing the "Security Considerations" section will be rejected. An EIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
