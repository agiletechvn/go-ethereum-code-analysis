# Some basic tools for packaging

In [go-ethereum](https://github.com/ethereum/go-ethereum) project, there is a small module golang ecosystem in some excellent tools package, due to the simple function, a chapter alone far too thin. However, because Ethereum's packaging of these gadgets is very elegant, it has strong independence and practicality. We do some analysis here, at least for the familiarity with the encoding of the Ethereum source code.

## metrics

In [ethdb-analysis](./ethdb-analysis.md), we saw the encapsulation of the[goleveldb](https://github.com/syndtr/goleveldb)project. Ethdb abstracts a layer on goleveldb.

[type Database interface](https://github.com/ethereum/go-ethereum/blob/master/ethdb/interface.go#L29)

In order to support the use of the same interface with MemDatabase, it also uses a lot of probe tools under the gometrics package in LDBDatabase , and can start a goroutine execution.

[go db.meter(3 \* time.Second)](https://github.com/ethereum/go-ethereum/blob/master/ethdb/database.go#L198)

Collect the delay and I/O data volume in the goleveldb process in a 3-second cycle. It seems convenient, but the question is how do we use the information we collect?

## log

Golang's built-in log package has always been used as a slot, and the Ethereum project is no exception. Therefore [log15](https://github.com/inconshreveable/log15) was introduced to solve the problem of inconvenient log usage.
