
## General Philosophy

Ethereum and complex blockchain programs are new and highly experimental. Therefore, you should expect constant changes in the security landscape, as new bugs and security risks are discovered, and new best practices are developed. Following the security practices in this document is therefore only the beginning of the security work you will need to do as a smart contract developer.

Smart contract programming requires a different engineering mindset than you may be used to. The cost of failure can be high, and change can be difficult, making it in some ways more similar to hardware programming or financial services programming than web or mobile development. It is therefore not enough to defend against known vulnerabilities. Instead, you will need to learn a new philosophy of development:

- **Prepare for failure**. Any non-trivial contract will have errors in it. Your code must therefore be able to respond to bugs and vulnerabilities gracefully.
  - Pause the contract when things are going wrong ('circuit breaker')
  - Manage the amount of money at risk (rate limiting, maximum usage)
  - Have an effective upgrade path for bugfixes and improvements

- **Roll out carefully**. It is always better to catch bugs before a full production release.
  - Test contracts thoroughly, and add tests whenever new attack vectors are discovered
  - Provide bug bounties starting from alpha testnet releases
  - Rollout in phases, with increasing usage and testing in each phase

- **Keep contracts simple**. Complexity increases the likelihood of errors.
  - Ensure the contract logic is simple
  - Modularize code to keep contracts and functions small
  - Prefer clarity to performance whenever possible
  - Only use the blockchain for the parts of your system that require decentralization

- **Stay up to date**. Use the resources listed in the next section to keep track of new security developments.
  - Check your contracts for any new bug that's discovered
  - Upgrade to the latest version of any tool or library as soon as possible
  - Adopt new security techniques that appear useful

- **Rethink programming concepts you know**. While much of your programming experience will be relevant to Ethereum programming, there are new and unique pitfalls to be aware of.
  - Be extremely careful about external contract calls, which may execute malicious code and change control flow.
  - Understand that your public functions are public, and may be called maliciously. Your private data is also viewable by anyone.
  - Keep gas costs and the gas block limit in mind.
