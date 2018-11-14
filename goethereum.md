
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>## go-ethereumSource Resolution
Because go ethereum is the most widely used Ethereum client, subsequent source code analysis is analyzed from the code above github.

### Build a go ethereum debugging environment

#### windows 10 64bit
First download the go installation package to install, because GO's website is walled, so download it from the address below.

Https://studygolang.com/dl/golang/go1.9.1.windows-amd64.msi

After installation, set the environment variable, add the C:\Go\bin directory to your PATH environment variable, then add a GOPATH environment variable, and set the GOPATH value to the code path of your GO language download (I set it up C:\GOPATH)

![image](https://raw.githubusercontent.com/wugang33/go-ethereum-code-analysis/master/picture/go_env_1.png)

Install the git tool, please refer to the tutorial on the network to install the git tool, go language automatically download code from github requires git tool support

Open the command line tool to download the code for go-ethereum

Go get github.com/ethereum/go-ethereum

After the command is successfully executed, the code will be downloaded to the following directory, %GOPATH%\src\github.com\ethereum\go-ethereum
If it appears during execution

# github.com/ethereum/go-ethereum/crypto/secp256k1
Exec: "gcc": executable file not found in %PATH%

You need to install the gcc tool, we download and install from the address below

Http://tdm-gcc.tdragon.net/download

Next install the IDE tool. The IDE I use is Gogland from JetBrains. Can be downloaded at the address below

Https://download.jetbrains.com/go/gogland-173.2696.28.exe

Open the IDE after the installation is complete. Select File -&gt; Open -&gt; select GOPATH\src\github.com\ethereum\go-ethereum to open it.

Then open go-ethereum/rlp/decode_test.go. Right-click on the edit box to select Run. If the run is successful, the environment setup is complete.

![image](https://raw.githubusercontent.com/wugang33/go-ethereum-code-analysis/master/picture/go_env_2.png)

### Ubuntu 16.04 64bit
&nbsp;
Go installation package for installation

Apt install golang-go git -y

Golang environment configuration:

Edit the /etc/profile file and add the following to the file:
Export GOROOT=/usr/bin/go
Export GOPATH=/root/home/goproject
Export GOBIN=/root/home/goproject/bin
Export GOLIB=/root/home/goproject/
Export PATH=$PATH:$GOBIN:$GOPATH/bin:$GOROOT/bin
Execute the following command to make the environment variable take effect:&lt;br/&gt;

# source /etc/profile

Download source code:

#cd /root/home/goproject; mkdir src; cd src #Enter the go project directory, create the src directory, and enter the src directory
#git clone https://github.com/ethereum/go-ethereum

Open it with vim or another IDE;


### go ethereum Directory Introduction
The organization structure of the go-ethereum project is basically a directory divided by functional modules. The following is a brief introduction to the structure of each directory. Each directory is also a package in the GO language. I understand that the package in Java should be similar. meaning.


Accounts implements a high-level Ethereum account management
Implementation of bmt binary Merkel tree
Build is mainly some scripts and configurations compiled and built
Cmd command line tool, a lot of command line tools, one by one
/abigen Source code generator to convert Ethereum contract definitions into easy to use, compile-time type-safe Go packages
/bootnode starts a node that only implements network discovery
/evm Ethereum virtual machine development tool to provide a configurable, isolated code debugging environment
/faucet
/geth Ethereum command line client, the most important tool
/p2psim provides a tool to simulate the http API
/puppeth Wizard to create a new Ethereum network
/rlpdump provides a formatted output of RLP data
/swarm swarm network access point
/util provides some public tools
/wnode This is a simple Whisper node. It can be used as a standalone boot node. In addition, it can be used for different testing and diagnostic purposes.
Common provides some common tool classes
Compression Package rle implements the run-length encoding used for Ethereum data.
Consensus provides some consensus algorithms for Ethereum, such as ethhash, clique(proof-of-authority)
Console console class
Contract
Core Ethereum core data structure and algorithm (virtual machine, state, blockchain, Bloom filter)
Crypto encryption and hash algorithm,
Eth implements the agreement of Ethereum
Ethclient provides the RPC client of Ethereum
Ethdb eth database (including the actual use of leveldb and the in-memory database for testing)
Ethstats provides a report on the status of the network
Event handles real-time events
Les implements a lightweight protocol subset of Ethereum
Light provides on-demand retrieval for Ethereum lightweight clients
Log provides log information that is friendly to humans
Metrics provide disk counters
Miner provides block creation and mining in Ethereum
Mobile some warpper used by mobile
Node Ethereum's various types of nodes
P2p Ethereum p2p network protocol
Rlp Ethereum serialization
Rpc remote method call
Swarm swarm network processing
Tests
Trie Ethereum's important data structure Package trie implements Merkle Patricia Tries.
Whisper provides a protocol for whisper nodes.

It can be seen that the code of Ethereum is still quite large, but roughly speaking, the code structure is still quite good. I hope to analyze from some relatively independent modules. Then delve into the internal code. The focus may be on modules such as p2p networks that are not covered in the Yellow Book.
</pre></body></html>