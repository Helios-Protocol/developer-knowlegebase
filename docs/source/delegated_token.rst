Helios protocol delegated token smart contract
==============================================


First, we specify the compiler version to use. Helios Protocol has developed a slightly improved version of solity that has additional features exclusive to us. Our version is 100.5.12 and above. This separates us from the vanilla solidity versions.

Thas being said, smart contracts compiled with standard solidity will still work on Helios Protocol. It is just that they won't make use of our additional features. We highly reccomend using our version of solidity to compile your code.

Download the example code to follow along
-----------------------------------------


.. code-block:: Bash

    cd ~/
    git clone https://github.com/Helios-Protocol/helios-code-examples
    cd helios-code-examples/smart_contracts/solidity/

Now you can open the file called deploy_token.py with your favorite python editor. It will be located at ~/helios-code-examples/smart_contracts/solidity/delegated_token.sol

You can also see the file we are going through here: https://github.com/Helios-Protocol/helios-code-examples/blob/master/smart_contracts/solidity/delegated_token.sol

Solidity code
-----------------------------------------

First we specify our compiler version. Here we are telling it that we need a compiler version greater than 100.5.11:
.. code-block:: Solidity

    pragma solidity ^100.5.11;

Next, we will import all of the helper contracts that we need:

.. code-block:: Solidity

    import "./helpers/smart_contract_chain.sol";
    import "./helpers/safe_math.sol";
    import "./helpers/execute_on_send.sol";
    import "./helpers/ownable.sol";

Next, we define our contract

.. code-block:: Solidity

    contract HeliosDelegatedToken is ExecuteOnSend, Ownable, SmartContractChain {
        using SafeMath for uint256;

Here, we have said our contract is an extension of ExecuteOnSend, Ownable, SmartContractChain, and we are using the SafeMath library for uint256. These will all be explained as we look at our functions below.

Next, we define variables used by the contract:

.. code-block:: Solidity

    // The balance of this token on the currently executing chain.
    uint256 balance;

    // Variables for the smart contract chain.
    // Change these to suit your token needs!
    string  public constant name = "My new token!";
    string  public constant symbol = "token symbol";
    uint8   public constant decimals = 18;
    uint256 public constant totalSupply = 300000000 * (10 ** uint256(decimals));

    // If you are creating your own standard token, there is no need to edit anything below this line

The balance variable contains the token balance of only this chain. It is simply a uint256, and not an array. Every wallet that gets sent some tokens will create their own balance variable that lives within the state of that particular chain. This differs significantly from Ethereum, where the smart contract state and variables are stored in the smart contract.

We also define some constants that can be modified to customize the token for your needs. Things like the name, symbol, decimals, and total supply. Feel free to set these 4 variables to whatever you want for your token.

Next, we define a function that is used to mint tokens. This function has the onlyFromSmartContractChain modifier. This means that it can only be called from a transaction that is generated computationally from this smart contract. This protects it from being called by anyone else who would be able to mint tokens on demand. Note, this is different than calling the function private. We want this function to be able to execute on other chains, so it cannot be private. But we require that the transaction that calls this function be sent from and only from the smart contract chain. We will see further down that the only way this contract can be called is when the token is first deployed, in which it will mint the total supply to the contract creator.

.. code-block:: Solidity

    /**
    * @dev Mint tokens onto whatever chain the transaction is sent to
    * This can only be initiated from this smart contract.
    * This is usually only called in the constructor when the contract is deployed.
    */
    function mintTokens(uint256 amount) public onlyFromSmartContractChain{
        balance = balance.add(amount);
    }

Next, we look at a function that will allow the smart contract to send the mintTokens command back to the creator's chain. This function creates a surrogatecall that sends a transaction back to the creator chain. This transaction will set the code_address to equal this smart contract so that it executes the minTokens function from this contract. We use Solidity assembly to send the surrogatecall.

.. code-block:: Solidity

    /**
    * @dev Create a surrogatecall transaction to send the mintTokens command
    * to a chain.
    * Be sure to leave this private so that it can only be called by this smart contract
    */
    function sendMintTokens(address _to, uint256 amount) private {
        bytes4 sig = bytes4(keccak256("mintTokens(uint256)")); //Function signature
        address _this = address(this);

        assembly {
            let x := mload(0x40)   //Find empty storage location using "free memory pointer"
            mstore(x,sig) //Place signature at beginning of empty storage (4 bytes)
            mstore(add(x,0x04),amount) //Place first argument directly next to signature (32 byte int256)

            let success := surrogatecall(100000, //100k gas
                                        _this, //Delegated token contract address
                                        0,       //Value
                                        0,      //Execute on send?
                                        _to,   //To addr
                                        x,    //Inputs are stored at location x
                                        0x24 //Inputs are 36 bytes long
                                        )

        }

    }


Next, we add a function that allows you to check the balance of this token any chane on Helios Protocol:

.. code-block:: Solidity

    /**
    * @dev Gets the balance of tokens on the currently executing chain
    */
    function getBalance() public view returns (uint256) {
        return balance;
    }

If you call this from chain A, you will be returned the balance of tokens on chain A. If you call this from chain B, you will be returned the balance of tokens on chain B etc...

Next, we define a function to transfer tokens from one chian to another:

.. code-block:: Solidity

    /**
    * @dev Transfers tokens from one chain to another. This function needs to be called using a transaction
    * with execute_on_send = True
    */
    function transfer(uint256 amount) public requireExecuteOnSendTx {
        if(is_send()){
            // This is the send side of the transaction. Here we subtract the amount from balance.
            require(amount <= balance);
            balance = balance.sub(amount);

        }else{
            // This is the receive side of the transaction. Here we add the amount to balance.
            balance = balance.add(amount);
        }
    }

There is a lot going on here. First of all, we have the requireExecuteOnSendTx modifier. This requires that the transaction used to call this function is being executed on the send transaction, and being executed on the receive transaction. This is very important to ensure tokens cannot be destroyed or minted by only executing one side of the computation. The next thing you notice is there is an is_send() part, and another part that is executed when the transaction is received. Basically what is happening, is when a transaction is sent calling this function, the balance is subtracted from the sender, and then when the receiver receives the transaction, the balance is added to their chain. This works in a way that is analogous to HLS coins on Helios Protocol.

Next, we look at the constructor. This function is executed once when the contract is deployed:

.. code-block:: Solidity

    constructor() public {

        // Mint the entire supply of tokens on the msg.sender's chain using a surrogatecall.
        // Here we are sending a surrogatecall transaction back to the owner. When the owner receives
        // this transaction, their balance will be increased by totalSupply
        address _this = address(this);
        address _owner = msg.sender;
        sendMintTokens(_owner, totalSupply);
    }

As you can see, when the contract is deployed, it sends a transaction back to the creator wallet. This transaction will tell the creator wallet to mint the totalSupply of tokens on to their chain. After this, the creator will have the total supply to send to whoever they want.

Lastly, we add a small function that will cause the virtual machine to not allow anyone to send any HLS to this contract. If someone sends HLS to this contract by accident, it will be rejected and sent back.

.. code-block:: Solidity

    // do not allow deposits
    function() external{
        revert();
    }
