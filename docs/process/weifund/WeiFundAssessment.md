### ASSESSMENT OUTLINE

Code Review
 - SCS Security Code Review Spreadsheet
 - Miscellaneous Notes
 - Attack Vector Assessments
 - Recommendations

Static Analysis
 - <pending tool availability>

Runtime Analysis

### ASSESSMENT

## Code Review Spreadsheet [to be created]

## Miscellaneous
WeiFund - Token.sol is extraneous
StandardCampaign::contributeMsgValue() does not return contributionID

## Attack Vector Assessments

# Timestamp attack discussion
Following guidance of internal security best practices, the WeiFund SCS appears vulnerable to various timestamp attacks that may be perpretrated by miners.  A miner may manipulate timestamp value to force a state change in CrowdfundingState.  This manipulation may lead to an invalid refund, a premature balance transfer to beneficiary, and contributions outside of the contribution window.  A state change attack of highest concern may be a failed campaign where the funds are attempted to be moved to the beneficiary.  This attack would also require the manipulation of the amountRaised data and as a result the SCS does not appear susceptible to this attack.  A state change attack of secondary concern may be a contributor attempting to remove funds during the operational or successful state (i.e., force a refund). It does appear that in the case of timestamp manipulation this SCS is vulnerable to this attack while in the operational state.

 # Pull payout discussion
Is there a justification why we are pushing payment to beneficiary in contrast to (security best practice of) replacing push payment with pull payment?  [If yes (e.g., beneficiary is by definition trusted), then we can add a comment here]

# Re-entry exploit discussion
StandardRefundCampaign:claimRefundOwed() may present a vulnerability that is subject to attack both in isolation (in the CampaignFailure state) and in combination with the timestamp attack above (where the CampaignOperational state is spoofed into a CampaignFailure state).  *This vulnerability is mitigated by the gas limit on the fallback function*, but may become an issue if the gas limit on the fallback function is increased or removed in the future.  Campaign contributor addresses are untrusted addresses, providing an opportunity for malicious actions including re-entry exploits. In StandardRefundCampaign:claimRefundOwed() an untrusted address's fallback method is called and we set the refund as having happened until after this point in the contract code.  The result is that potential re-entrant recursion initiated by a malicious contributor in the CampaignFailure state.  Campaigns do not have to fail to be susceptible to this vulnerability.  In the case of a nearly-funded campaign reaching its expiry, the miner role is incentivized to:  (1) contribute from a malicious address into the fund; (2) spoof the timestamp to force the contract into a Failure state; (3) re-entrantly claim and extract a refund to the malicious address.

# Call depth attack discussion
In the call-depth-attack section of smart contract best practices in the secure auction:withdrawRefund() the control flow pattern of [1] set current state to next expected state, [2] initiate a call to the fallback method, [3] revert current state in case of failure of fallback method via throw() is used in contrast to the control flow pattern of [1] initiate a call to the fallback method, [2a] in Success case set current state to next state, [2b] in Failure case revert transaction without any meaningful change in state found in StandardRefundCampaign:claimRefundOwed().  We should determine the desired control flow pattern and bring the contract and the document into alignment if they do not have equivalent results in response to an attack.

# Speed bump discussion
There is an absence of speed bumps throughout the SCS.  Areas that may justify inclusion of speed bumps include campaign failure to refund window and campaign success to beneficiary window.

## Recommendations
1. explicitly label untrusted methods
2. determine best pattern for the StandardRefundCampaign:claimRefundOwed() call to fallback method
3. determine how to upgrade contract (e.g., in case of gas limit removal)
4. justify push payment in comment or replace with pull payment design
5. remove Token.sol
6. return contributionID in success case for StandardCampaign::contributeMsgValue()
7. consider addition of speed bumps