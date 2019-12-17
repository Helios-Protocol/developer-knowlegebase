Voting smart contract deployment and usage
===================================================================

In this walkthrough, we will deploy and interact with the voting smart contract taken directly from the Ethereum Solidity documentation found here: https://solidity.readthedocs.io/en/latest/solidity-by-example.html. This is an example showing how standard Ethereum Solidity code can be migrated to Helios Protocol easily without any modification.

We will be deploying the contract using our python web3, with python version 3.6.5. You will need to use python to follow along with this walkthrough. You can download it here: https://www.python.org/

Download the example code to follow along
-----------------------------------------


.. code-block:: Bash

    cd ~/
    git clone https://github.com/Helios-Protocol/helios-code-examples
    cd helios-code-examples/web3_py/ethereum_solidity_examples

Now you can open the file called voting.py with your favorite python editor. It will be located at ~/helios-code-examples/web3_py/ethereum_solidity_examples/voting.py

You can also see the file we are going through here: https://github.com/Helios-Protocol/helios-code-examples/blob/master/web3_py/ethereum_solidity_examples/voting.py


Create hypothesis testnet account/wallet
----------------------------------------

In order to follow along with this walkthrough, you will need to create a new account/wallet on the Hypothesis testnet. You can do this by loading our online wallet at: https://heliosprotocol.io/wallet/, and generate a new keystore. After you have a new keystore created, copy the wallet address, and go to our faucet to get some testnet HLS sent to it: https://heliosprotocol.io/faucet

Now you have a new testnet account, and it has some HLS in it.

Install dependencies
--------------------

To begin, we need to make sure we have all of the dependencies installed. Install them by running the following commands:

.. code-block:: Bash

    pip install helios-web3
    pip install py-helios-solc
    pip install eth-keys
    pip install eth-keyfile
    pip install py-helios-node
    pip install eth-utils


Python code walkthrough
-----------------------

First we will deploy the smart contract. For details of each part of this, see :ref:`deployment_walkthrough`:

.. code-block:: Python

    import time
    from eth_keys import keys
    import eth_keyfile
    from helios_web3 import HeliosWeb3 as Web3
    from helios_web3 import IPCProvider, WebsocketProvider
    from helios_web3.utils.block_creation import prepare_and_sign_block

    from helios_solc import install_solc, compile_files

    from hvm.utils.address import generate_contract_address
    from eth_utils import encode_hex, to_checksum_address, to_wei

    from hvm.constants import CREATE_CONTRACT_ADDRESS, GAS_TX

    W3_TX_DEFAULTS = {'gas': 0, 'gasPrice': 0, 'chainId': 0}

    #
    # First we compile our solidity file.
    #

    # First install the solidity binary v100.5.12 and above is helios solc


    install_solc('v100.5.12')

    # Next, compile your file. We will compile the delegated token contract
    solidity_file = '../../smart_contracts/solidity/ethereum_solidity_examples/voting.sol'
    contract_name = 'Ballot'
    compiled_sol = compile_files([solidity_file])

    # get the contract interface. This contains the binary, the abi etc...
    contract_interface = compiled_sol['{}:{}'.format(solidity_file, contract_name)]


    #
    # Next, we deploy the compiled contract to the network
    #

    # Websocket URL for hypothesis testnet bootnode. If you change this to mainnet, make sure you change network id too.
    websocket_url = 'wss://hypothesis1.heliosprotocol.io:30304'
    network_id = 42

    # Use this code to load a private key from a keystore file. You will deploy the contract from this account
    # We have provided a test keystore file that may contain a small amount of testnet HLS. But you should replace it
    # with your own.
    keystore_path = '../test_keystore.txt' # path to your keystore file
    keystore_password = 'LVTxfhwY4PvUEK8h' # your keystore password
    private_key = keys.PrivateKey(eth_keyfile.extract_key_from_keyfile(keystore_path, keystore_password))

    # Create web3
    w3 = Web3(WebsocketProvider(websocket_url))

    # Create the web3 contract factory
    Ballot = w3.hls.contract(
        abi=contract_interface['abi'],
        bytecode=contract_interface['bin']
    )

    # Build transaction to deploy the contract.
    w3_tx1 = Ballot.constructor([b'proposal1', b'proposal2']).buildTransaction(W3_TX_DEFAULTS)


    transaction = {
                    'to': CREATE_CONTRACT_ADDRESS,
                    'gas': 20000000, #make sure this is enough to cover deployment
                    'value': 0,
                    'chainId': network_id,
                    'data': w3_tx1['data']
                }

    # Give the transaction the correct nonce and prepare the header
    signed_block, header_dict, transactions = prepare_and_sign_block(w3, private_key, [transaction])

    # Send it to the network
    response = w3.hls.sendRawBlock(signed_block['rawBlock'])

    #Done! Your contract is now deployed.

    # How do I figure out the deployed contract address?
    deployed_contract_address = generate_contract_address(private_key.public_key.to_canonical_address(), transactions[0]['nonce'])
    print("Contract deployed to address {}".format(encode_hex(deployed_contract_address)))


Next, we will create a new wallet account and give it the right to vote:

.. code-block:: Python

    # First, we must wait 10 seconds before we can add another block to our chain
    print("Waiting 10 seconds before sending the next block")
    time.sleep(10)

    # We have to re-create the contract factory and give it the address of the contract
    # Create the web3 contract factory
    Ballot = w3.hls.contract(
        address=to_checksum_address(deployed_contract_address),
        abi=contract_interface['abi'],
    )
    # Create a new account
    new_account = w3.hls.account.create()
    new_private_key = new_account._key_obj

    w3_tx1 = Ballot.functions.giveRightToVote(new_private_key.public_key.to_canonical_address()).buildTransaction(W3_TX_DEFAULTS)

    transaction = {
                    'to': deployed_contract_address,
                    'gas': 20000000, #make sure this is enough to cover deployment
                    'value': 0,
                    'chainId': network_id,
                    'data': w3_tx1['data'],
                }

    # Give the transaction the correct nonce and prepare the header
    signed_block, header_dict, transactions = prepare_and_sign_block(w3, private_key, [transaction])

    # Send it to the network
    response = w3.hls.sendRawBlock(signed_block['rawBlock'])

    print("Successfully gave {} the right to vote".format(new_private_key.public_key.to_checksum_address()))


Now lets delegate our vote to the new account:

.. code-block:: Python

    # First, we must wait 10 seconds before we can add another block to our chain
    print("Waiting 10 seconds before sending the next block")
    time.sleep(10)

    w3_tx1 = Ballot.functions.delegate(new_private_key.public_key.to_canonical_address()).buildTransaction(W3_TX_DEFAULTS)

    transaction = {
                    'to': deployed_contract_address,
                    'gas': 20000000, #make sure this is enough to cover deployment
                    'value': 0,
                    'chainId': network_id,
                    'data': w3_tx1['data'],
                }

    # Give the transaction the correct nonce and prepare the header
    signed_block, header_dict, transactions = prepare_and_sign_block(w3, private_key, [transaction])

    # Send it to the network
    response = w3.hls.sendRawBlock(signed_block['rawBlock'])

    print("Successfully delegated our vote to {}".format(new_private_key.public_key.to_checksum_address()))

Now lets vote with the new account. First we have to send it some HLS so it can pay for gas

.. code-block:: Python

    # First we have to send some HLS to the new account
    print("Waiting 10 seconds before sending the next block")
    time.sleep(10)
    transaction = {
                    'to': new_private_key.public_key.to_canonical_address(),
                    'gas': GAS_TX, #make sure this is enough to cover deployment
                    'value': to_wei(1, 'ether'),
                    'chainId': network_id,
                }

    # Give the transaction the correct nonce and prepare the header
    signed_block, header_dict, transactions = prepare_and_sign_block(w3, private_key, [transaction])

    # Send it to the network
    response = w3.hls.sendRawBlock(signed_block['rawBlock'])

    print("Successfully sent 1 HLS to {}".format(new_private_key.public_key.to_checksum_address()))

Now we receive the HLS on the new account

.. code-block:: Python

    receivable_transactions = w3.hls.getReceivableTransactions(new_private_key.public_key.to_canonical_address())

    # Prepare the header
    signed_block, header_dict, transactions = prepare_and_sign_block(w3, new_private_key, receivable_transactions = receivable_transactions)

    # Send it to the network
    response = w3.hls.sendRawBlock(signed_block['rawBlock'])


    print("Successfully received HLS at {}".format(new_private_key.public_key.to_checksum_address()))


Now we vote with the new account

.. code-block:: Python

    print("Waiting 10 seconds before sending the next block")
    time.sleep(10)

    w3_tx1 = Ballot.functions.vote(1).buildTransaction(W3_TX_DEFAULTS)

    transaction = {
                    'to': deployed_contract_address,
                    'gas': 20000000, #make sure this is enough to cover deployment
                    'value': 0,
                    'chainId': network_id,
                    'data': w3_tx1['data'],
                }

    # Give the transaction the correct nonce and prepare the header
    signed_block, header_dict, transactions = prepare_and_sign_block(w3, new_private_key, [transaction])

    # Send it to the network
    response = w3.hls.sendRawBlock(signed_block['rawBlock'])

    print("Successfully voted for proposal 1")


Finally, lets find the winning proposal.

.. code-block:: Python

    transaction = {
                    'from': private_key.public_key.to_canonical_address(),
                    'to': deployed_contract_address,
                }

    winning_proposal = Ballot.caller(transaction=transaction).winnerName()

    print("The winning proposal is {}".format(winning_proposal))

And the winning proposal was returned correctly!