## Precompiled contract

This version is based on the Byzantium precompiles. The range proof consists of 2 parts:
 * Generating the proof that a number is within an interval (outside the blockchain by the client that submits that proof).
 * Validating the proof that this number is within that interval (executed by each validating node on the blockchain).

On Ethereum validation of transactions in smart contract logic is typically done in the Ethereum Virtual Machine. However the operations involved in the validation of this range-proof are too computationally expensive to run on the EVM. Therefore we call a precompiled contract during verification. A precompiled contract is written in the native language of the Ethereum-client (in our case in Golang) and is preconfigured to live at a specific address (with a low number). In our case we use the precompile bigModExp at address 0x5, which is available in Ethereum since the Byzantium release.

 ## Gas consumption

 Ethereum uses the concept of gas which means the sender of a transaction needs to pay (i.e. Eth or Etc) for the computational steps executed by the smart-contract that is invoked by the transaction. The more complex computations the smart contract executes, the more gas will be consumed. Therefore the transaction specifies a gas limit and a gas price.

 Gas limit is there to protect you from buggy code running until your funds are depleted. The product of gasPrice and gas represents the maximum amount of Wei that you are willing to pay for executing the transaction. What you specify as gasPrice is used by miners to rank transactions for inclusion in the blockchain. It is the price in Wei of one unit of gas, in which VM operations are priced.

 The current implementation is not yet optimized for gas usage, the verification costs are around 3 million gas. On private networks this is usually not an issue, but it may be too expensive in the Ethereum mainnet.

## Usage

The usage of this go-ethereum library consists of four parts:
1. Running an Ethereum client with a chain that enables the Byzantium precompiles;
2. Generating a commitment (in Java);
3. Generating a range proof (in Java);
4. Validating range proofs with the Solidity smart contract (connecting from Java, uPort, or via console).

We will describe each step in more detail below. We have tried to make these instructions accessible for a broad audience.

#### Running the Ethereum client

Make sure that you have `geth` installed. If not, follow the instructions on https://github.com/ethereum/go-ethereum/wiki/Building-Ethereum

From the zkrangeproof working directory, you can initialize a private chain using the provided `genesis.json` file, for example:
```
geth init ./data/genesis.json
geth --targetgaslimit 99900000000 --networkid 15997 --rpc --rpcaddr 0.0.0.0 --rpcport 8545 --rpccorsdomain "*"
```

#### Generating commitments

Open a project in your favourite IDE and make sure that you have an IDE (plugin) that supports Gradle, or another dependency manager.
Note that this is a go-ethereum library; you cannot 'run' anything directly.

You can now i.e. add a new java package to the project.
The input for the commitment is a 'secret' `BigInteger` (`java.math.BigInteger`) and a `SecretOrderGroup`. You can generate the `SecretOrderGroup` with:
```java
// import com.ing.blockchain.zk.components.SecretOrderGroupGenerator
// import com.ing.blockchain.zk.dto.SecretOrderGroup
new SecretOrderGroupGenerator(512).generate();
```
where `512` is the bitlength (higher is safer, but also reduces the efficiency).

You can now generate the TTPMessage, which contains the commitment, using:
```java
// import com.ing.blockchain.zk.TTPGenerator
// import com.ing.blockchain.zk.dto.TTPMessage
TTPMessage ttpMessage = TTPGenerator.generateTTPMessage(x, group);
```
where `x` is the 'secret' `BigInteger` and `group` the `SecretOrderGroup`.

This TTPMessage contains six attributes: the first four together describe the commitment, whereas the latter two are private variables for the user that are also needed to generate range proofs.
The commitment can be retrieved with the `getCommitment()` method, and the private variables can be retrieved with the `getX()` and `getY()` methods.
The four different segments of the commitment can then be retrieved by calling the methods `getCommitmentValue()`, `getGroup().getN()`, `getGroup().getG()`, and `getGroup().getH()` on the result of `getCommitment()`.

In the precompiled smart contract we expect the commitment to be (only) the first four variables of the TTPMessage, separated by commas.

#### Generating range proofs

Given the commitment and the user's private variables, we can generate proofs for specific ranges. Hence, we need the TTPMessage here.
Other inputs are a lower bound and an upper bound for the range proof. The bounds need to match with the bounds that are being checked by the relying party.
The range must be supplied as a `ClosedRange`:
```java
// import com.ing.blockchain.zk.dto.*
ClosedRange range = ClosedRange.of(lower, upper);
```
where `lower` and `upper` are BigIntegers.

A range proof should only be generated if this range contains the secret value:
```java
if (range.contains(ttpMessage.getX()))
```

The range proof can now be generated by calling:
```java
// import com.ing.blockchain.zk.RangeProof
BoudotRangeProof rangeProof = RangeProof.calculateRangeProof(ttpMessage, range);
```

And can directly be verified by:
```java
RangeProof.validateRangeProof(rangeProof, ttpMessage.getCommitment(), range);
```

The range proof can be exported for the Solidity contract using the provided ExportUtils:
```java
ExportUtil.exportForEVM(commitment))

        System.out.println("Proof = ");
        System.out.println(DatatypeConverter.printHexBinary(ExportUtil.exportForEVM(rangeProof, commitment, range)));
```

In the precompiled smart contract we expect the range proof to be the above variables (in this order), separated by commas.

#### Validating proofs by calling the precompiled smart contract

Another way to verify a range proof is by calling the precompiled smart contract that is embedded in our modified Geth client. Make sure that the modified Geth is running (see the first section of this instruction).

You first need to set up an Ethereum client in Java (i.e. via web3j). Once set up, you can call the precompiled contract at address `0x0000000000000000000000000000000000000009`. You can call the following method to validate the range proof:
```java
validate(lower, upper, commitment, rangeProof);
```
which returns `true` iff the range proof is valid for this range and commitment.

It is also possible to call the smart contract from uPort by instantiating another web3 instance that connects to the local modified Geth client. Please refer to the uPort documentation for calling smart contracts. Once instantiated by providing an ABI (embedded in the code snippet below), you can call the `validate` method on the smart contract as illustrated in the snippet above.

For testing purposes you can also call the precompiled smart contract from the command line. Start the modified Geth client with the argument `console`, or open a 
terminal using `geth attach` to be able to execute commands from the command line.
First, instantiate the smart contract:
```
var precompiled = eth.contract([{"constant":true,"inputs":[{"name":"lower","type":"uint64"},{"name":"upper","type":"uint64"},{"name":"commitment","type":"string"},{"name":"proof","type":"string"}],"name":"validate","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"view","type":"function"}]).at("0x0000000000000000000000000000000000000009");
```
this always returns `undefined`. Now, call the contract by filling in the variables below:
```
precompiled.validate(lower, upper, "commitment", "rangeProof");
```
which returns `true` iff the range proof is valid for this range and commitment.

Please note that the variables `commitment` and `rangeProof` should be the comma separated combination of the `BigInteger` variables specified in this instruction. The commitment is only the first four variables of the TTPMessage.

