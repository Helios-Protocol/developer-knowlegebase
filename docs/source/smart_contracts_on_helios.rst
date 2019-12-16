Smart contracts on Helios Protocol
==================================

Helios Protocol supports Solidity smart contracts. If you have programmed your smart contract for Ethereum, and would like to use a platform that can support the growth of your dapp, then deploy it on Helios Protocol today!

In addition to the vanilla Solidity, we have created our own modified Solidity compiler that supports new features specifically for Helios Protocol. These new features are as follows:

Additions and modifications for Helios Solidity
-----------------------------------------------
Helios Protocol is not proof of work (PoW), so it has no need for a COINBASE opcode. Because of this, the COINBASE opcode has been replaced with the EXECUTEONSEND opcode. It uses the same '0x41' value that COINBASE had previously used. This new opcode will tell you whether the currently executing transaction has been configured to execute on the send transaction and receive transaction side. If EXECUTEONSEND == True, then it will execute on both sides, if EXECUTEONSEND == False, then it will only execute on the receive side. You can access this variable within your solidity code via msg.executeonsend and tx.executeonsend.

Helios Protocol is multi chain. Smart contracts that want to interact with another smart contract, or another wallet, have to send transactions to each other. This can result in chains of transactions from contract to contract. The variable tx.origin, within your solidity code, will always refer to the original address that sent the first transaction to initiate the chain of transactions.

Helios Protocol solidity adds a new kind of call. This is called a surrogatecall. This kind of call allows you to send a transaction to any wallet or smart contract, but it will execute code from another smart contract that you define. It is in some ways similar to a delegatecall. For example, I can send a token from chain A to chain B. But the smart contract code is located on chain C. I do this by sending a transaction from chain A to chain B, but reference chain C as the code_address. Please see the delegated token example below to see it in action.

If you would like to check the code_address of the currently executing transaction, you can use the tx.code_address variable within your solidity code. It will tell you the address of the code that is currently being executed. This CODEADDRESS opcode has replaced the DIFFICULTY opcode. It uses the same '0x44' value.




If you would like more information about programming Solidity smart contracts, please refer to the original Ethereum documentation at https://solidity.readthedocs.io/en/v0.5.12/.  We share 99% of the same specifications.

Below is a list of smart contracts that you can use as is, or build on top of.

Example smart contracts
------------------------

.. toctree::
    :maxdepth: 2

    delegated_token.rst