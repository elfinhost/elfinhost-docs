===================
ElfinHost WhitePaper
===================

-------------------------------------------------------------------
An Access-Control Protocol for Decentralized File Hosting
-------------------------------------------------------------------

Web2.0 heavily depends on file hosting and CDN to deliver mass multimedia contents to users. The authors spend lots of effort to create such multimedia contents, and it costs storage and bandwidth resources to deliver such contents to end users. The authors and delivers need to earn money from the end users directly (through fees) or indirectly (through advertisement). In web2.0 era, they have used a lot of centralized methods to control who can and how to access these contents. If anyone can access any content on internet in anyway she wants, no business can continue.

Web3.0 already has several decentralized file hosting solutions, such as IPFS/Filecoin, Swarm, and Arweave. However, they only incentive the storage providers. The authors and delivers cannot earn money from such solutions, because there are no access control methods for the contents.

Blockchains build the internet of value. Smart contracts enable programmable value. Can the traditional internet of information be built with blockchains as well? Can smart contracts program information accessing? 

ElfinHost answers these questions with "Yes!". It allows the authors to program who can and how to access the contents hosted on decentralized storages. Many blockchain-oriented business models can be invented based on it.

ElfinHost has following innovative features:

1. Hardware-based protection. It doesn't rely on law, morality, discipline, or human-controlled permissions.
2. Easy to program with solidity. The authorization is based on smart contracts and on-chain states
3. No single-point failure. Content delivery and authorization are decentralized.

This article introduces ElfinHost's technique step-by-step. At each step, we refine the solution a little bit and finally we reach the whole picture.

Enforce permissions on public data
-----------------------------------

Although there isn't a common definition of web3.0, most people believe it will let users control their own data, have better privacy and prevent centralized platforms from monopolizing the acquisition, recommendation, and censorship of information. The decentralization of data storage and delivery is a very important cornerstone of web3.0.

Decentralized storage is easily accessible nowadays. You can pin a video on IPFS with a lot of utilities. However, it is very hard to ensure all your audience can play the video smoothly. Decentralized delivery is a hard task.  One of the most important values provided by centralized platforms is that they help authors easily deliver contents to mass audience.

High-resolution pictures, long articles with many figures, and long audios and videos need large files. Large files are expensive to transfer, even more expensive than to store. Because the cost of bandwidth is higher than storage.  Content delivery networks (CDN) can reduce the bandwidth cost by caching data in edge nodes that are near the target audience. In the web3.0 era, we still need CDN.

CDN providers only serve the websites that pay them. They use the "same origin policy" to decide whether to serve a request. If website B contains an image or video from website A, the CDN provider will not serve B by providing A's contents. 

However, decentralized storage schemes do not explicitly show a file's origin, nor constrain access to files according to some policy. Thus, we can only encourage CDN providers to serve permissionless data to the public, ignoring which website or App is requesting the data, which severely limited the use cases.

We want to enforce permissions on the public contents on decentralized storage and ensure only the target audience can get contents.

Encryption enclaves can help. Encryption enclaves connect to each other and share one symmetric key for encryption/decryption. This key is generated inside one encryption enclave and broadcasted to others through secure channels.  Hardware enclaves protect the symmetric key safely. An original file, together with its target audience' addresses, is encrypted into one file by an encryption enclave. This encrypted file is stored on IPFS.

When a user request the original file, a CDN provider fetches the encrypted data, and uses an encryption enclave to decrypt it. The enclave will check if the requester's address is listed as the file's target audience. If so, it decrypts the file and the CDN provider sends the original file to the requester.

End-to-end encryption
-----------------------------------

In web2.0 platforms, users updated original files without encryption. These files are possibly leaked to the public or malicious attackers, even if the users request the platforms to keep them private. Law, morality, and company discipline are all against privacy disclosure, but the uploaded files must be stored in databases or filesystems, which use human-controlled permissions to constrain access. We know that software has vulnerabilities and that humans are not always reliable and honest. So privacy disclosure accidents keep happening.

At CDN providers' backend, the operators can view any content, regardless of the access constraint of the served websites. In our above example, the CDN providers can keep a copy of the decrypted file, which is against the author's will.

The solution is to upgrade encryption enclaves to "recryptors". A recryptor takes an encrypted file as input, decrypts it into the original content and re-encrypts it with another key.

When an author wants a recryptor to encrypt a file before uploading it to IPFS, he must encrypt the original file beforehand with a shared secret that is only known to him and the recryptor. When a user requests a recryptor to decrypt a file from IPFS, he will get a file encrypted with the shared secret, instead of the original file.

During the whole process of storing and delivering, only the inside logic of the recryptor enclave can access the original content, and hardware enclaves ensure it will never be leaked. So there will no other human can view the original content other than the author and the target audience.

Authorization contracts
-----------------------------------

Traditionally, CDN providers need an authorization server for sophisticated dynamical access control. An authorization server is run by a CDN provider's customer. For example, if Bob request to view a URL of website A, the CDN provider will ask the authorization server running by website A whether Bob can be served. Website A may allow Bob to view it if he is a VVIP, or make Bob wait until Friday if he is a VIP, or just deny Bob if he is nobody.  In the web3.0 era, authors need an easier way for dynamical access control. Running an authorization server is too hard for ordinary authors.

On-chain smart contracts can be used for dynamic access control. For example, only the holder of specific NFTs or ERC20 tokens can view the author's contents. A recryptor uses `eth_call` to invoke a smart contract's function with the requestor's address as the argument, if it returns "true" then the requestor is granted. The file uploaded to IPFS specifies which contracts to call and how to call them, instead of a static set of the target audience.

Multi-Grant from Authorities
-----------------------------------

When a recryptor needs `eth_call` for authorization, it will face witch attacks. To query `eth_call`, you need a blockchain node to provide RPC endpoints. You can run a node youself but in most cases, you will rent a node from NaaS (node as a service) provider.

Although the recryptors' internal data and logic are safe under the protection of enclaves, the input data it gets through `eth_call` may be incorrect. A CDN provider may get incorrect information from a NaaS because it does not configure the recryptor's DNS and TLS setting properly. A node run by a CDN provider may return incorrect information if it is hacked because of vulnerabilities.

Any CDN provider may have security problems. Trusting one single CDN provider is not good for the authors.

The solution is to divide the task of authorization out from the recryptors and use dedicated authorizers to query `eth_call`. Authorizers are run by several trustworthy authorities which have good security measures and reputations.

To further protect the symmetric key, we use a "multi-grant" scheme which is similar to "multi-signature". The author specifies N authorities and a threshold number M. Before uploading, the recryptor must encrypt the original file with all the N grant codes. Before the recryptor decrypts a file for a requestor, the requestor must collect at least M grant codes from the specified authorities.

All the authorizers run by the same authority have the same "grant root". For each individual file, an authorizer derives a unique grant code from the grant root, after it ensures the requestor is allowed to access the file. The derive root is generated inside enclaves and shared among enclaves, even an operator employed by the authority cannot get its value. The grant codes are sent from authorizers to recryptors through secure channels which prevent any third party from viewing them. To sure the grant codes only go into trustable enclaves, authorizers always attest to the recryptors, before they open secure channels.

The encryption/decryption algorithm for "multi-grant" will be introduced in another article.

File sharding to mitigate risks of enclaves' bugs
-----------------------------------

The enclaves are the key to the system's security. If the hardware and/or firmware have bugs, enclaves may also leak information. Although there have been no real attacks reported on CPUs with hyperthreading disabled, the risks are always there.

Currently, enclaves can be implemented using Intel's SGX and TDX, AMD's SEV-SNP, ARM's TEE, and AWS's Nitro. SGX is the most mature and popular solution while the others are also evolving quickly. We divide enclaves into different zones. The enclaves in each zone use the same CPU technology and the same firmware. For example, all the enclaves based on Intel SGX and Azure's firmware are in the same zone.

It is very unlikely that all the zones are exploited by hackers at the same time. So an author can further protect his file by sharding it into multiple parts, each of which is protected by a different enclave zone. For example, one file is divided into three parts: part #1 is handled by Azure's SGX enclaves, part #2 by GCP's SEV-SNP enclaves, and part #3 by AWS's nitro enclaves. A requestor must get all three parts to recover the original file.

The whole picture
-------------------

Now, the whole picture of ElfinHost is here: it needs recryptors and authorizers running inside several enclave zones to provide services; the authors use smart contracts for files' access control; the audience rely on authorizers to grant requests, and recryptors to decrypt files on IPFS (or other decentralized storages).

