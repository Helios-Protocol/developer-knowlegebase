Token smart contract interaction walkthrough
===============================================================================================

This walkthrough will guide you through the process of interacting with a smart contract on the Helios Protocol testnet, known as Hypothesis Testnet. The particular smart contract the we will be interacting with in this walkthrough is a token smart contract located here: https://github.com/Helios-Protocol/helios-code-examples/blob/master/smart_contracts/solidity/delegated_token.sol. This walkthrough is designed for linux, but should be fairly easy to follow along using other operating systems.

We will be interact with the contract using our python web3, with python version 3.6.5. You will need to use python to follow along with this walkthrough. You can download it here: https://www.python.org/


Download the example code to follow along
-----------------------------------------


.. code-block:: Bash

    cd ~/
    git clone https://github.com/Helios-Protocol/helios-code-examples
    cd helios-code-examples/web3_py/delegated_token

Now you can open the file called interact_with_contract.py with your favorite python editor. It will be located at ~/helios-code-examples/web3_py/delegated_token/interact_with_contract.py

You can also see the file we are going through here: https://github.com/Helios-Protocol/helios-code-examples/blob/master/web3_py/delegated_token/interact_with_contract.py

Dependencies and pre-requisites
-------------------------------

It is assumed that you have completed the walkthrough to deploy your token. You need to have the token contract deployed, you need to have minted your tokens on your chain, and you need to have your keystore file ready to be loaded. If you have not done this walkthough yet, then you can do so here :ref:`deployment_walkthrough`

Python code walkthrough
-----------------------

In this code, we will first check the balance of tokens on your chain, then we will send tokens to another chain, then we will check the balance on that chain, and finally, we will receive gas refunds.

Starting from the top of the example code, we will do the same thing as before, and make sure we have the correct solidty installed, and then compile the smart contract:

.. code-block:: Python

    from eth_keys import keys
    import eth_keyfile
    from eth_utils import to_checksum_address, encode_hex
    from helios_web3 import HeliosWeb3 as Web3
    from helios_web3 import IPCProvider, WebsocketProvider
    from helios_web3.utils.block_creation import prepare_and_sign_block
    from helios_solc import install_solc, compile_files
    import time

    W3_TX_DEFAULTS = {'gas': 0, 'gasPrice': 0, 'chainId': 0}

    #
    # First we compile our solidity file.
    #

    # First install the solidity binary v100.5.12 and above is helios solc
    from hvm.constants import CREATE_CONTRACT_ADDRESS

    install_solc('v100.5.12')

    # Next, compile your file. We will compile the delegated token contract
    solidity_file = '../../smart_contracts/solidity/delegated_token.sol'
    contract_name = 'HeliosDelegatedToken'
    compiled_sol = compile_files([solidity_file])

    # get the contract interface. This contains the binary, the abi etc...
    contract_interface = compiled_sol['{}:{}'.format(solidity_file, contract_name)]


We had to compile the code again so that we can provide web3 with the abi to let it know what functionality the contract has. Next, lets set the variables to tell web3 how to connect to the node, and to load our private key from a keystore. We will also set some variables that contain the deployed contract address, and the address of the chain that deployed the contract (the one that has all of the tokens minted to it. It is assumed that this is the chain that is loaded from the keystore):

.. code-block:: Python

    # Websocket URL for hypothesis testnet bootnode. If you change this to mainnet, make sure you change network id too.
    websocket_url = 'wss://hypothesis1.heliosprotocol.io:30304'
    network_id = 42

    # Use this code to load a private key from a keystore file. You will deploy the contract from this account
    # We have provided a test keystore file that may contain a small amount of testnet HLS. But you should replace it
    # with your own.
    keystore_path = 'test_keystore.txt' # path to your keystore file
    keystore_password = 'LVTxfhwY4PvUEK8h' # your keystore password
    private_key = keys.PrivateKey(eth_keyfile.extract_key_from_keyfile(keystore_path, keystore_password))

    deployed_contract_address = '0xa5df294e3ee433b748d7cfc9814112fc5ae5bd27' # Replace this with the address of your contract
    deployer_wallet_address = private_key.public_key.to_checksum_address()

Next, we will initialize web3 to connect to the node using websockets, then we will check the balance on our chain by using the "caller" function on the web3 contract factory. This function allows you to call any function in the smart contract without creating a new transaction. This will also return the result of whatever function we are calling without the need to log the output as you would do with a normal transaction.

.. code-block:: Python

    # Create web3
    w3 = Web3(WebsocketProvider(websocket_url))

    # Create the web3 contract factory
    HeliosDelegatedToken = w3.hls.contract(
        address=to_checksum_address(deployed_contract_address),
        abi=contract_interface['abi'],
    )

    # Build transaction to deploy the contract.

    transaction = {
                    'from': deployer_wallet_address,
                    'to': deployer_wallet_address,
                    'codeAddress': deployed_contract_address # The code address tells it where the smart contract code is.
                }

    balance = HeliosDelegatedToken.caller(transaction=transaction).getBalance()

    print("The balance on chain {} before the transfer is {}".format(deployer_wallet_address, balance))

You can see here that we defined a "codeAddress" in the transaction that is used for the "call". This tells the VM, that we would like to use the code located on the 'deployed_contract_address' chain, which is the delegated token we deployed earlier. We can also see that the "from" and 'to" is set to 'deployer_wallet_address', this means that we are using the local state and memory located on deployer_wallet_address's chain. This is important because the balance has been minted onto that chain, and the state of these minted tokens is also stored on this chain. If you want to find the balance of anyone else's wallet, you would change 'from' and 'to' to that wallet.

Now that we have measured the balance on that chain, we will now transfer some tokens to another chain. This will show you how to use the transfer function programmed into the smart contract. We start by creating a new account to which we will send the tokens


.. code-block:: Python

    # Create a new account to send it to
    new_account = w3.hls.account.create()
    new_private_key = new_account._key_obj

Next, we will use the web3 contract factory to create a transaction to transfer tokens to the new account. Then we will sign a block with that transaction, and then send it to the network:

.. code-block:: Python

    amount_to_transfer = 1000

    w3_tx1 = HeliosDelegatedToken.functions.transfer(amount_to_transfer).buildTransaction(W3_TX_DEFAULTS)

    transaction = {
                    'to': new_private_key.public_key.to_canonical_address(),
                    'gas': 20000000, #make sure this is enough to cover deployment
                    'value': 0,
                    'chainId': network_id,
                    'data': w3_tx1['data'],
                    'codeAddress': deployed_contract_address, # The code address tells it where the smart contract code is.
                    'executeOnSend': True, # Helios Delegated Tokens require executeOnSend = True for transfering tokens
                }

    # Give the transaction the correct nonce and prepare the header
    signed_block, header_dict, transactions = prepare_and_sign_block(w3, private_key, [transaction])

    # Send it to the network
    response = w3.hls.sendRawBlock(signed_block['rawBlock'])

    print("Sending {} tokens from {} to {}".format(amount_to_transfer, deployer_wallet_address, new_private_key.public_key.to_checksum_address()))

Because each wallet on helios protocol has it's own blockchain, transactions contain a send and receive portion. Now that we successfully sent the transaction, we need to receive it onto the other chain to complete it. We will do that now:

.. code-block:: Python

    #
    # Receive the transaction on the chain we sent it to
    #
    # Get receivable transactions from the node
    receivable_transactions = w3.hls.getReceivableTransactions(new_private_key.public_key.to_canonical_address())

    # Prepare the header
    signed_block, header_dict, transactions = prepare_and_sign_block(w3, new_private_key, receivable_transactions = receivable_transactions)

    # Send it to the network
    response = w3.hls.sendRawBlock(signed_block['rawBlock'])

    print("Receiving tokens on chain {}".format(new_private_key.public_key.to_checksum_address()))

Here, we asked the node to give us all receivable transactions for the new wallet we generated, and then we generate and sign a new block containing those transactions, and then we send it to the network. After this, the transfer of tokens from one chain to another is complete. Congratulations!

Next, to confirm that the transaction sent successfully, we will check the balance on the receiving chain. We will use the same technique as before:

.. code-block:: Python

    #
    # Check the token balance on the chain you sent them to
    #
    transaction = {
                    'from': new_private_key.public_key.to_canonical_address(),
                    'to': new_private_key.public_key.to_canonical_address(),
                    'codeAddress': deployed_contract_address # The code address tells it where the smart contract code is.
                }

    balance = HeliosDelegatedToken.caller(transaction=transaction).getBalance()

    print("The balance on chain {} is {}".format(encode_hex(new_private_key.public_key.to_canonical_address()), balance))

The balance is as expected.

Lastly, on Helios Protocol, because every wallet and smart contract executes code on it's own chain, in its own environment, there is no way of telling how much gas most computation transactions will use. So the way it works is the sender initially pays the max gas, then the transaction is sent to the receiver and executed. After execution, the leftover gas is sent back to the sender as a gas refunt (with 0 transaction fees), so that no gas is lost, and there are no wasted fees. In order to accept this gas refund, you must receive it. We do this just like receiving any other transaction. We ask the node for receivable transactions, then sign a block with them, and then send the block to the network:


.. code-block:: Python

    # We must wait 10 seconds before we can add the next block
    print("Waiting 10 seconds before receiving gas refund")
    time.sleep(10)
    # Get receivable transactions from the node
    receivable_transactions = w3.hls.getReceivableTransactions(private_key.public_key.to_canonical_address())

    # Prepare the header
    signed_block, header_dict, transactions = prepare_and_sign_block(w3, private_key, receivable_transactions = receivable_transactions)

    # Send it to the network
    response = w3.hls.sendRawBlock(signed_block['rawBlock'])

    print('Gas refunds received successfully')

Now you have learnt how to interact with the smart contract using the call functionality, and using a standard transaction. Happy coding!