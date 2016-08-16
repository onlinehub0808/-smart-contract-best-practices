# SCS Vulnerability Assessment Guide

## Code Review Checklist

### Source Code Inspection Codes
#####Forbidden
1.  Storing or providing private [sensitive] date
2.  Prematurely revealing contract state via mempool
3.  Use of call().value()
4.  Poor RNG
5.  Two- or n- party contracts 'hangs' when participation ceases
6.  Mutex that can be claimed and never relinquished
7.  Narrow timestamp or block number dependencies in control flow
8.  Inability to repair broken contracts
9.  Absence of emergency stop or circuit breaker
10. Absence of speed bumps where reasonable
11. Absence of rate limits where reasonable
12. Absence of assert guards where reasonable
13. Unjustified complexity


####Conditional
1.  If external call, is external call trusted
2.  If external function call, is error case handled
3.  If untrusted external call, is at end of control flow
4.  If untrusted external call, is reflected in name space
5.  If internal call, is at end of control flow if call chain contains UEC
6.  If payment, is pull payment and not push payment
7.  If integer division, is loss of precision acceptable
8.  If division, is divide by zero case explicitly handled
9.  If fallback function, is limited to event logging
10. If private state variable, is visibility explicit
11. If public function, is visibility explicit
12. If event, is name prefix explicit
13. If low-level function, is false return value error case handled
14. If call to send, does false return value error case call throw
15. If state is shared, is mutex used if relevant
16. If looping, is state preserved if loop breaks

### Attacks to Pursue in Dynamic Analysis
1.  Re-entry
2.  Call stack depth
3.  Malicious contract (e.g., DoS)

### Exploit Bounty Tiers
1. DoS and Race Condition
2. Call Depth and Dependency [timestamp, transaction ordering]
3. Unknown and Theft of Funds

