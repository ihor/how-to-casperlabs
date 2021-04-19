# DRAFT - NOT FINAL

# DEVxDAO Casper Testnet Rewards

## Introduction

Thank you for participating in the DEVxDAO's Casper Testnet program. Your voluntary participation in this Testnet is enabling 
stakeholders across the Casper eco-system, including core node developers, validators, dApp developers and tool developers,
to test their software in a non-production environment, and to preview upcoming software changes. 
As announced at the onset of the Testnet program, the DEVxDAO is planning to award grants to Testnet participants, contingent
upon their adherence to the [Code of Conduct](testnet.md), based on an algorithmically calculated score that incorporates
certain performance criteria (see below), and subject to certain limits and final DEVxDAO approval through its governance process.

## Reward Calculation

### Principles

* Rewards are calculated on a weekly basis
* Weeks begin on Monday at 0:00:00 UTC, and end on Sunday at 23:59:59 UTC
* Rewards are awarded to the top-ranked `n` of Testnet participants in a given week:
    * `n` is subject to a future vote of the DEVxDAO
    * the ranking each week is calculated as explained below 
    * all participants tied for the last reward-eligible position will be awarded
* Rewards are accrued and paid at a later date, subject to Casper token lock-up periods and DEVxDAO governance and grant specifics
* The total reward amount available to the program is subject to a DEVxDAO vote, per its governance rules

### Weekly Reward Calculation

The basic scoring algorithm is:

```shell
100 * uptime_percentage
```
where `uptime_percentage` is defined as your node running within 1 block from the "tip" of the blockchain. Your node is regularly scanned
on port `8888` to check your block height, so if your port `8888` is closed, you are automatically not considered "up". `uptime_percentage`
is calculated on a 24-hour basis, during each UTC day. 

#### Deductions

Your cumulative weekly score can be reduced if any of the following events occur:

##### Network Weight

If your node at any time during the weekly calculation period has a network weight of 3% or more, your score for that 
week will be reduced by 10%

##### Node Software Version

If your node is not running the expected casper-node software version by the `era` following the `era` that a software
upgrade is slated to take effect, your score for that week will be reduced by 10%. As an example, if a software upgrade is
slated for `era 234` and your node is still not upgraded at the beginning of `era 235`, you will lose 10% of your score.


_Please note that the DEVxDAO's Casper Testnet program is implemented by the [DEVxDAO](https://devxdao.com) by providing rewards 
through the [Emerging Technology Association](https://www.emergingte.ch) (ETA), a Swiss nonprofit association which supports open source 
and transparent scientific research of emerging technologies for community building. 
Any rewards will be granted and calculated by the ETA. MAKE Technology LLC is not affiliated
with the DEVxDAO, the ETA nor the Casper Foundation, and has no control over the program sponsorship or the incentivized
reward program, and is hosting these guides and documents as a service to the DEVxDAO and the Casper community only._