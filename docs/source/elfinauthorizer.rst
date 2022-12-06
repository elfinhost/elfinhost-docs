==============================================================
Elfin Authorizer: the new service of NaaS providers
==============================================================

-------------------------------------------
Blockchain-based Sophisticated Access Control
-------------------------------------------

In the Web 2.0 era, account-based access control is very common. You must register to a website or App. Then it grants you a "level", such as Guest, VIP, SVIP, etc. With different levels, you have different service quality. For example, some high-quality contents are only available to VIP, and SVIP account can view a new TV show much earlier than a guest account. 

As we come into the Web 3.0 era, authorization based on blockchain accounts are more and more popular. Many websites and Apps support "login with Web3 wallets". But just login is not enough. We also want to use on-chain state to assign different levels to the accounts. When a website decides whether to provide a service to a specific account, it may want to consider:

1. Historical contract interaction. Did it even send transactions to call some contracts?

2. Historical events. Did it ever receive some ERC20 token or NFT?

3. The latest state of the account. Does it own some ERC20 token or NFT? Does it belong to some DAO?

These conditions can be AND-ed/OR-ed together to form more sophisticated conditions.

Node-as-a-service (NaaS) providers runs plenty of nodes of different blockchains. Accessing on-chain state is a regular task for them. It would be very easy for them to add a new service: Elfin authorizers. 

Elfin authorizers can assign access permissions to accounts according to such sophisticated conditions. A website or App can outsource the permissioning task to them.

Elfin authorizers are oracles which endorse facts by signing them using their private keys. This article introduces the different facts they can sign and how to express sophisticated conditions using these signed facts.

Primitives provided by authorizers
==================================

Endorse historical contract interaction
------------------------------------

A historical transaction can be described by such a solidity struct:

.. code-block:: solidity

    struct TxInfo {
        uint chainId; // which chain did this log happen on?
        uint timestamp; // when did this log happend at?
        uint txid; // the transaction's hash id
        address fromAccount;
        address toAccount;
        uint value;
        bytes callData;
    }

If a transaction really happend in a blockchain's history, the Elfin authorizer generates a `TxInfo` struct to describe it. Then it uses `abi.encodePacked` to serialize this struct into raw bytes and sign the keccak256 hash of the raw bytes. In such a way it endorses this transaction using its private key.

Endorse historical events
------------------------------------

Events are implemented using EVM logs. A EVM log can be described by such a solidity struct:

.. code-block:: solidity

    struct LogInfo {
        uint chainId; // which chain did this log happen on?
        uint timestamp; // when did this log happend at?
        address sourceContract; // which contract generates this log?
        bytes32[] topics; // a log has 0~4 topics
        bytes data; // a log has variable-length data
    }

If a transaction really happend in a blockchain's history, the Elfin authorizer generates a `LogInfo` struct to describe it. Then it uses `abi.encodePacked` to serialize this struct into raw bytes and sign the keccak256 hash of the raw bytes. In such a way it endorses this event using its private key.

Endorse the outputs of eth_call
------------------------------------

A requestor asks the Elfin authorizer to query the `eth_call` endpiont of Web3 RPC servers. The 16~35 bytes of the call data used for `eth_call` must equal the authorizer's EVM address calculated from its private key. The authorizer collects the related information about this `eth_call` to fill the following struct:

.. code-block:: solidity

    struct EthCallInfo {
        uint chainId;
        uint timestamp;
        address fromAccount;
        address targetContract;
        bytes4 functionSelector;
        bytes outData;
    }

Then it uses `abi.encodePacked` to serialize this struct into raw bytes and sign the keccak256 hash of the raw bytes. In such a way it endorses the outputs of `eth_call` using its private key.

Granting secrets to account owners
------------------------------------

A requestor asks the Elfin authorizer to query the `eth_call` endpiont of Web3 RPC servers. The 16~35 bytes of the call data used for `eth_call` must equal the authorizer's EVM address calculated from its private key. The from-account for `eth_call` must be the requestor's EVM address (a `personal_sign` signature is required to ensure this). The authorizer collects the related information about this `eth_call` to fill the following struct:

.. code-block:: solidity

    struct SecretSeed {
        uint chainId;
        bytes4 functionSelector;
        address targetContract;
        bytes outData;
    }

Then it uses `abi.encodePacked` to serialize this struct into raw bytes and calculate the keccak256 hash of the raw bytes. With its private key and this hash, it generates a VRF (verifiable random function) output and a proof. The VRF output is a secret that only qualified requestor can get.

For granting secrets, authorizers also supports a recryptor mode, which requires the request comes from a recryptor's enclave. In the recryptor mode, the raw bytes' sha256 hash is used for VRF, instead of keccak256 hash.

Write authorization contract to express sophisticated conditions
==================================================================

Suppose we want to provide a file-sharing service only to such qualified accounts:

1. Someone who is explicitly marked as qualified member by a superuser

2. Someone who has called a contract and received a given ERC20 token in the recent two months

The `isQualified` function of the following `Membership` contract can check if `msg.sender` is a qualified account:

.. code-block:: solidity

    import "@openzeppelin/contracts/access/Ownable.sol";
    
    struct Signature {
            uint8 v;
            bytes32 r;
            bytes32 s;
    }
    
    contract Membership is Ownable {
        mapping(address => bool) public isMember;
        mapping(uint => bool) public forbiddenFiles;
        address immutable public erc20Token;
        address immutable public calledContract;
        bytes32 constant private TransferEvent = keccak256("transfer(address,address,uint256)");
        string constant private PREFIX = "\x19Ethereum Signed Message:\n32";
    
        constructor(address _erc20Token, address _calledContract) Ownable() {
            erc20Token = _erc20Token;
            calledContract = _calledContract;
        }
    
        function setMembership(address addr, bool ok) public onlyOwner {
            isMember[addr] = ok;
        }
    
        function getHash(TxInfo calldata t) internal pure returns (bytes32) {
            bytes32 h = keccak256(abi.encodePacked(t.chainId, t.timestamp, t.txid, t.fromAccount, t.toAccount, t.value, t.callData));
            return keccak256(abi.encodePacked(PREFIX, h));
        }
    
        function getHash(LogInfo calldata l) internal pure returns (bytes32) {
            bytes32 h = keccak256(abi.encodePacked(l.chainId, l.timestamp, l.sourceContract, l.topics, l.data));
            return keccak256(abi.encodePacked(PREFIX, h));
        }
    
        function isQualified(address authorizer, TxInfo calldata txInfo, Signature calldata txSig,
                     LogInfo calldata logInfo, Signature calldata logSig) public view returns (bool) {
            if(isMember[msg.sender]) return true;
            require(authorizer==ecrecover(getHash(txInfo), txSig.v, txSig.r, txSig.s), "invalid-txSig");
            require(authorizer==ecrecover(getHash(logInfo), logSig.v, logSig.r, logSig.s), "invalid-logSig");
            uint twoMonthAgo = block.timestamp - 60 days;
            return txInfo.toAccount==calledContract && txInfo.fromAccount == msg.sender &&
                logInfo.topics[0]==TransferEvent && logInfo.topics[2]==bytes32(bytes20(msg.sender)) &&
                twoMonthAgo < txInfo.timestamp && twoMonthAgo < logInfo.timestamp;
        }
    }

Before calling `isQualifed`, a requestor must query the authorizer to get `TxInfo` and `LogInfo`, which will be used as the arguments to call `isQualified`. The first argument must be the authorizer's address, which is used to ensure the `TxInfo` and `LogInfo` were really generated by the same authorizer.

When the authorizer endorses the `EthCallInfo` struct after calling `isQualified`, the requestor has a proof that he is a qualified account.

Now, we want to upgrade this file-sharing service to support encryption and decryption. The files are decrypted with symmetric keys which is only known to the qualified accounts. Any qualified account can use the symmetric key of current time to encrypt and upload files. But different accounts have different permissions in decryption: 

1. Someone who is explicitly marked as qualified member by a superuser, can decrypt all the files.

2. Someone who has called a contract and received a given ERC20 token in the recent two months, can only decrypt the files which are encrypted in recent five days.

We add a new function `getSecret` to the `Membership` contract:

.. code-block:: solidity

    function setForbidden(uint fileid, bool foridden) public onlyOwner {
        forbiddenFiles[fileid] = foridden;
    }

    function getSecret(address authorizer, uint fileid, TxInfo calldata txInfo, Signature calldata txSig,
             LogInfo calldata logInfo, Signature calldata logSig, uint shareTime) public view returns (uint, uint) {
        if(forbiddenFiles[fileid]) return (0, 0);
        if(isMember[msg.sender]) return (shareTime, fileid);
        require(authorizer==ecrecover(getHash(txInfo), txSig.v, txSig.r, txSig.s), "invalid-txSig");
        require(authorizer==ecrecover(getHash(logInfo), logSig.v, logSig.r, logSig.s), "invalid-logSig");
        uint twoMonthAgo = block.timestamp - 60 days;
        bool qualified = txInfo.toAccount==calledContract && txInfo.fromAccount == msg.sender &&
            logInfo.topics[0]==TransferEvent && logInfo.topics[2]==bytes32(bytes20(msg.sender)) &&
            twoMonthAgo < txInfo.timestamp && twoMonthAgo < logInfo.timestamp;
        if(qualified && block.timestamp - 5 days < shareTime && shareTime < block.timestamp + 1 hours) {
            return (shareTime, fileid);
        }
        return (0, 0);
    }

The argument `shareTime` is the  time when this file was encrypted and shared. The `fileid` is a unque id assigned to each shared file. The superuse can disable the sharing of individual files by calling `setForbidden` using `fileid`. If several files belong to a single file logically, such as the segments of the same m3u8 file, or the files of a multi-part archive, it is suggested that they share the same `fileid`.

A requestor asks the authorizer to call `getSecret` function for secret-granting. The authorizer will fill a `SecretSeed` struct and use it to generate a VRF output. This output is used as the symmetric key for encryption and decryption.

The RPC Endpoints
==================================================================

An authorizer provides four RPC endpoints to support the mentioned primitives. All these endpoints returns a JSON object, with the following fields:

1. IsSuccess: If the RPC finishes successfully

2. Message: When IsSuccess equals true, it's an empty string. When IsSuccess equals false, it's the string explaining the reason

3. Result: for granting secret, this is the from-account's address and the VRF output (in recryptor mode this output is encrypted); for the other endpoints, this is the raw bytes to be signed.

4. Proof: for granting secret, this is the VRF proof; for the other endpoints, this is the signature

5. Salt:  Only used in the recryptor mode for granting secret. It's first eight bytes is the current timestamp (little endian) and the other bytes are random number generated by hardware RNG.

6. PubKey: Only used in the recryptor mode for granting secret. It's the authorizer's public key.

In recryptor mode, the recryptor calculates a secret with its private key and the authorizer's PubKey, and then uses this secret and the returned Salt to decrypt the returned Result to get VRF output.

Endorse historical contract interaction

The RPC endpiont's URL is like below:

.. code-block::

   /tx?hash=<transaction-hash-id>

The `hash` parameter is hex format and starts with "0x".

Endorse historical events
------------------------------------

The RPC endpiont's URL is like below:

.. code-block::

   /log?contract=<contract-address>&block=<blockhash>&topic1=<hex-string>&topic2=<hex-string>&topic3=<hex-string>&topic4=<hex-string>

The parameters `topic1`~`topic4` are used to filter out one single EVM log generated by the `contract` in the specified `block`. Some or all of them can be omitted, as long as exactly one EVM-log is got after filtering.

All these parameters are hex format and start with "0x".

Endorse the outputs of eth_call
------------------------------------

The RPC endpiont's URL is like below:

.. code-block::

    /call?contract=<contract-address>&data=<calldata>&from=<from-account-address>

All these three parameters are hex format and start with "0x".

Granting secrets to account owners
------------------------------------

The RPC endpiont's URL is like below:

.. code-block::

    /grantcode?time=<unix-timestamp>&contract=<contract-address>&data=<calldata>&sig=<from-account-signature>&recryptorpk=<pubkey-of-recyrptor>&out=<outdata>

The `time` parameter is a decimal integer. All the other five parameters are hex format and start with "0x".

The `recryptorpk` and `out` parameters are only used in recryptor mode, where the requestor is the recryptor enclave. The `recryptorpk` presents the public key of the recryptor and we are in recryptor mode if it is specified. In recryptor mode, the body of the http request must be the attestation report of the recryptor enclave. Authorizers check this report to ensure the request is sent from an SGX enclave.

The 20 bytes of `calldata[16:36]` will be overwritten by the authorizer's EVM address, before the authorizer uses `calldata` to query `eth_call`. Thus, the called function can read this EVM address as its first argument.

The from-account's address can be recovered from the `sig` parameter. When the `sig`  is omitted, the `from` account has zero address. The `sig` is generated using MetaMask's `personal_sign`. The signed text is:

.. code-block::

  To Authorizer: time=<unix-timestamp>, contract=<contract-evm-address>, data=<hex-encoded-calldata-with-0x-prefix>

In recryptor mode, if the recryptor wants to push a file to cloud, it uses the `out` parameter to specifiy the output of `eth_call`. Then the authorization will not query `eth_call`. Instead, it uses the `out` parameter as the output of `eth_call`.

Load Balance and Authentication
===============================

The enclave implementation of Elfin authorizer is designed to run on a single machine. The service provider can run many Elfin authorizer enclaves and use a reverse proxy to distribute the requests to them.

The provider can only provide services to a limited set of customers, such as the recryptors of a CDN vendor who has paid. The RPC endpoints provided by the enclave do not support authentication.

If the provider would like to use some authentication methods (basic auth, API keys, etc), it can use the reverse proxy to deploy them. The basic auth header and the API key parameter must be removed before forwarding the request to the enclaves.

Rate Limit
===========

The Elfin authorizer does not support rate limit. The service provider can implement rate limit in the reverse proxy.

