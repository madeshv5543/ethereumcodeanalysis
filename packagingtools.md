
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre># Some basic tools for packaging
In the [Ethernet] (https://github.com/ethereum/go-ethereum) project, there are small modules that encapsulate some excellent tools in the golang ecosystem. Because of their relatively simple functions, they are too thin. However, because Ethereum's packaging of these gadgets is very elegant, it has strong independence and practicality. We do some analysis here, at least for the familiarity with the encoding of the Ethereum source code.
## metrics(probe)
In [ethdb source code analysis] (/ethdb source code analysis.md), we saw the encapsulation of the [goleveldb] (https://github.com/syndtr/goleveldb) project. Ethdb abstracts a layer on goleveldb:

[type Database interface] (https://github.com/ethereum/go-ethereum/blob/master/ethdb/interface.go#L29)

To support the use of the same interface with MemDatabase, use the probe tool under the [gometrics](https://github.com/rcrowley/go-metrics) package in LDBDatabase and start a goroutine execution.

[go db.meter(3 * time.Second)](https://github.com/ethereum/go-ethereum/blob/master/ethdb/database.go#L198)

Collect the delay and I/O data volume in the goleveldb process in a 3-second cycle. It seems convenient, but the question is how do we use the information we collect?

## log(log)
Golang's built-in log package has always been used as a slot, and the Ethereum project is no exception. Therefore, [log15] (https://github.com/inconshreveable/log15) was introduced to solve the problem of inconvenient log usage.


</pre></body></html>