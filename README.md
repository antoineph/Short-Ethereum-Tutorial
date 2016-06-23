# Problems with Ethereum


Let's go through an example where we deploy a contract from one node, and interact with it on another node on the same machine. 

The contract we will use is a simple ticker contract:


```javascript
contract ticker {
	uint public val;

	function tick () { 
		val += 1;
	}
}
```

To run a 2-node network on one machine, first paste the following genesis file in a file
called genesis.json. You can modify this as you feel fit, if you know what the paramaters mean.


```javascript
{
   "nonce": "0x0000000000000123",
   "timestamp": "0x0",
   "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
   "extraData": "0x0",
   "gasLimit": "0x8000000",
   "difficulty": "0x400",
   "mixhash": "0x0000000000000000000000000000000000000000000000000000000000000000",
   "coinbase": "0x3333333333333333333333333333333333333333",
   "alloc": {}
}

```

Now open 2 terminals. In the first, run


```bash 
 $ geth --datadir node_0 --genesis genesis.json --port 30303 --networkid 123 console 2> node_0.log
```

When the prompt appears, get the 'enode id', via

```javascript
> admin.nodeInfo.enode
"enode://82df6993f8fe1961973b06f9bf3ae0fd8fb26e1a49f0169ca546e880905b0e7ba3f3ffe7fe90fe9033d4beaaca4d20dbfb09ddfc3a48c18461d9902ad70221a9@[::]:30303"
```

In the second terminal, run


```bash
$ geth --datadir node_1 --genesis genesis.json --port 30304 --networkid 123 console 2> node_1.log
```

Wait for the prompt, then paste the above enode id into the argument of `admin.addPeer`:

```javascript
> admin.addPeer("enode://82df6993f8fe1961973b06f9bf3ae0fd8fb26e1a49f0169ca546e880905b0e7ba3f3ffe7fe90fe9033d4beaaca4d20dbfb09ddfc3a48c18461d9902ad70221a9@[::]:30303");
true
```

Do the following in both terminals to set up the context.


```javascript
> personal.newAccount("<some-password>"); // make an account
"0x0b102091f3f88ffb4d1e0fbb6cbc27ed6e53f14f"
> web3.eth.defaultAccount = web3.eth.accounts[0] // make it the default for transactions
> miner.setEtherbase(web3.eth.accounts[0]); // make it the account that gets the mined ether
> miner.start(); // start mining
```

Then build the contract from the source:

```javascript
> source = 'contract ticker { uint public val; function tick () { val += 1; } }'; 
> compiled = web3.eth.compile.solidity(source).ticker;  // compile it
> ... // suppressed output for brevity
> code = compiled.code;  // get the bytecode
> abi = compiled.info.abiDefinition; //get the abi definition
```
Now create a contract from the abi:

```
> ticker_contract = web3.eth.contract(abi);
```

Before deploying, be sure to unlock your account:

```javacsript
> personal.unlockAccount(web3.eth.accounts[0], "<your-password-here>");
```


You will deploy the contract from the first node, then use the address obtained from mining to interact with that contract in the second node. To deploy the contract, run

```javascript
> ticker = ticker_contract.new({from: web3.eth.accounts[0], data: code});
```

After some delay (dependent on your machine), the variable `ticker.address` will have a value. 

```javascript
> ticker.address
undefined
> ... // wait
> ticker.address
"0x2d77014f5f8fff4f5efdc4bf98f0e051aeca2fdf"
```

Instead of waiting and checking repeatedly, a more standardized approach is to use a callback function like this:


```javascript
function tx_callback(e, contract) {
	if (!e) {
    if (!contract.address) {
        console.log("Contract transaction sent: TransactionHash: " + contract.transactionHash + " waiting to be mined...");
    } else {
        console.log("Contract mined. Address: " + contract.address);        
    }
   }
}
```

So, alternatively, you can paste this into your session, then invoke the `ticker_contract.new` function through:

```javascript
> ticker = ticker_contract.new({from: web3.eth.accounts[0], data: code}, tx_callback);
Contract transaction sent: TransactionHash: 0x5fa60491b0aa3d65e90b487588712c966498dae13b6941777f49871541ad2e12 waiting to be mined...
{
  abi: [{
      constant: true,
      inputs: [],
      name: "val",
      outputs: [{...}],
      type: "function"
  }, {
      constant: false,
      inputs: [],
      name: "tick",
      outputs: [],
      type: "function"
  }],
  address: undefined,
  transactionHash: "0x5fa60491b0aa3d65e90b487588712c966498dae13b6941777f49871541ad2e12"
}
> Contract mined. Address: 0xd888023025ad33df516a1bcb2a0a4dce483acceb
```

In the second now, you can access this contract like so:

```javascript
> ticker = ticker_contract.at("0xd888023025ad33df516a1bcb2a0a4dce483acceb");
```

Verify both nodes are interacting with the same contract by executing the tick function and inspecting the value of the ticker. Check the value of the ticker is 0 on both nodes: 

```javascript
> parseInt(ticker.val()); // check the current value of the ticker
> 0
```

Then, on the first node,

```javascript
> ticker.tick();
"0x03025bfeb25bf5e8c2e1abf9a9e5b7a4cf50837c471b34ca6a5769d52541078d"
> parseInt(ticker.val()); 
1
```

The second node should as well yield 1:

```javascript
> parseInt(ticker.val());
1
```

###### Note: This is obviously not the most efficient procedure for interacting with source code. For a more natural approach, consider using Node.js. There are a number of different resources for this, but one, by Mohammad Kidwai, can be found on his github, [here](https://github.com/kidwai/ethereum-tutorial).
