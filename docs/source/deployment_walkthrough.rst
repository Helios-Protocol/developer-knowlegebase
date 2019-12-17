.. _deployment_walkthrough:


Token deployment walkthrough
===============================================================================================

This walkthrough will guide you through the process of deploying a smart contract on the Helios Protocol testnet, known as Hypothesis Testnet. The particular smart contract the we will deploy in this walkthrough is a token smart contract. This walkthrough is designed for linux, but should be fairly easy to follow along using other operating systems.

We will be deploying the contract using our python web3, with python version 3.6.5. You will need to use python to follow along with this walkthrough. You can download it here: https://www.python.org/


Download the example code to follow along
-----------------------------------------


.. code-block:: Bash

    cd ~/
    git clone https://github.com/Helios-Protocol/helios-code-examples
    cd helios-code-examples/web3_py/delegated_token

Now you can open the file called deploy_token.py with your favorite python editor. It will be located at ~/helios-code-examples/web3_py/delegated_token/deploy_token.py

You can also see the file we are going through here: https://github.com/Helios-Protocol/helios-code-examples/blob/master/web3_py/delegated_token/deploy_token.py


Set up your token parameters
----------------------------------------

The smart contract example located at https://github.com/Helios-Protocol/helios-code-examples/blob/master/smart_contracts/solidity/delegated_token.sol has several parameters that can be easily edited for your personalized token. You can edit them and get your token online within minutes!

.. code-block:: Solidity

    // Variables for the smart contract chain.
    // Change these to suit your token needs!
    string  public constant name = "My new token!";
    string  public constant symbol = "token symbol";
    uint8   public constant decimals = 18;
    uint256 public constant totalSupply = 300000000 * (10 ** uint256(decimals));

The variables are quite self explanatory. You can edit them to change the name, symbol, decimals, and total supply fo your token. The total supply in tokens will be the number in front of "* (10 ** uint256(decimals));"


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

The first thing we will do is import the packages we need and define some constants:

.. code-block:: Python

    import time
    from eth_keys import keys
    import eth_keyfile
    from helios_web3 import HeliosWeb3 as Web3
    from helios_web3 import IPCProvider, WebsocketProvider
    from helios_web3.utils.block_creation import prepare_and_sign_block
    from helios_solc import install_solc, compile_files
    from hvm.utils.address import generate_contract_address
    from eth_utils import encode_hex

    W3_TX_DEFAULTS = {'gas': 0, 'gasPrice': 0, 'chainId': 0}


Next, we install the Helios Protocol version of the solidty compiler. If it is already installed, this will be skipped.

.. code-block:: Python

    install_solc('v100.5.12')


Now that we have the compiler installed, we will go ahead and compile the delegated token contract.

.. code-block:: Python

    # Next, compile your file. We will compile the delegated token contract
    solidity_file = '../../smart_contracts/solidity/delegated_token.sol'
    contract_name = 'HeliosDelegatedToken'
    compiled_sol = compile_files([solidity_file])

    # get the contract interface. This contains the binary, the abi etc...
    contract_interface = compiled_sol['{}:{}'.format(solidity_file, contract_name)]


The contract_interface variable contains everything related to the compiled code. This includes the bin, the abi etc... We will now set the connection variables, and load your private key that you will use to deploy the smart contract. Be sure to replace the keystore_path and the path to your keystore that you generated above. Also replace the keystore_password with your password.

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

Now we will initialize our web3 and tell it to connect to the testnet node via websockets. After that, we will create the contract factory to help us deploy our token

.. code-block:: Python

    # Create web3
    w3 = Web3(WebsocketProvider(websocket_url))

    # Create the web3 contract factory
    HeliosDelegatedToken = w3.hls.contract(
        abi=contract_interface['abi'],
        bytecode=contract_interface['bin']
    )

You can see here that we provided the w3.hls.contract function with parts of the compiled solidity code.

Next, we will build the transaction and block that will be sent to the network to deploy our smart contract:

.. code-block:: Python

    # Build transaction to deploy the contract.
    w3_tx1 = HeliosDelegatedToken.constructor().buildTransaction(W3_TX_DEFAULTS)


    transaction = {
                    'to': CREATE_CONTRACT_ADDRESS,
                    'gas': 20000000, #make sure this is enough to cover deployment
                    'value': 0,
                    'chainId': network_id,
                    'data': w3_tx1['data']
                }

    # Give the transaction the correct nonce and prepare the header
    signed_block, header_dict, transactions = prepare_and_sign_block(w3, private_key, [transaction])

Now that our block is all ready, we will send it to the network:

.. code-block:: Python

    # Send it to the network
    response = w3.hls.sendRawBlock(signed_block['rawBlock'])

That's it! Your contract has now been deployed. But there are a few things we still want to do before quitting. The first thing is we want to find out what address the contract deployed to:

.. code-block:: Python

    # How do I figure out the deployed contract address?
    deployed_contract_address = generate_contract_address(private_key.public_key.to_canonical_address(), transactions[0]['nonce'])
    print("Contract deployed to address {}".format(encode_hex(deployed_contract_address)))

Now you can use this address whenever you want to interact with your smart contract. To send tokens from one account to another, for example.

Lastly, this smart contract has been programmed to generate a new transaction that is sent back to whoever deployed it. This transaction will automatically mint the max supply of tokens on your chain. But for this to happen, we need to receive the transaction that the smart contract made.


.. code-block:: Python

    #
    # After the deploy takes place, it will send us a new transaction to mint the tokens on our chain. Lets receive that transaction
    #
    # We must wait 10 seconds before we can add the next block
    print("Waiting 10 seconds before adding new block")
    time.sleep(10)

    # Get receivable transactions from the node
    receivable_transactions = w3.hls.getReceivableTransactions(private_key.public_key.to_canonical_address())

    # Prepare the header
    signed_block, header_dict, transactions = prepare_and_sign_block(w3, private_key, receivable_transactions = receivable_transactions)

    # Send it to the network
    response = w3.hls.sendRawBlock(signed_block['rawBlock'])

    print('Transaction received successfully!')

Now you have minted the total supply of tokens onto your chain. Congratulations! Next, you will want to take a look at our walkthrough for interacting with the smart contract
