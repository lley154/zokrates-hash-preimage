# Zokrates Hash Preimage
The goal is to compute the hash for the number 5 and then prove they know the preimage number 5 but doesn't have to reveal it.

Open a terminal window
```
$ mkdir ~/zokrates
$ cd ~/zokrates
$ pwd
$ docker run -it -v /path-to-zokrates-directory/zokrates:/workspace -ti zokrates/zokrates /bin/bash
$ cd /workspace
$ mkdir hash
$ cd hash
```

Complete all of the steps in the following example https://zokrates.github.io/examples/sha256example.html

Open a new terminal window
```
$ docker run -it -v /path-to-zokrates-directory/zokrates:/workspace --name docker-forge ghcr.io/foundry-rs/foundry:stable bash
$ anvil
```

Open another new terminal window
```
$ cd ~/zokrates
$ pwd
$ docker exec -it docker-forge bash
$ cd /workspace/hash
$ git config --global user.email your@email.com
$ git config --global user.name your.name
$ forge init zk-verifier
$ cd zk-verifier
```

Open a new terminal window and create a test file in ~/zokrates/hash/zk-verifier/test/Verifier.t.sol
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/Verifier.sol";

contract VerifierTest is Test {
    Verifier public verifier;

    function setUp() public {
        verifier = new Verifier();
    }

    function testVerification() public view {
        // Create the proof using the Verifier.Proof struct
        Verifier.Proof memory proof = Verifier.Proof({
            a: Pairing.G1Point(
                0x0fd7c25f1711c81edc5f72fbd03b661d25a2ca56f3681b190dfe9ac082debfb4,
                0x1520eafee703713031737a7e2c0d242c2f3acc772cd6394374f38e0f82816a97
            ),
            b: Pairing.G2Point(
                [0x0f41db00ce6c03f4fb0030f83a7a3d69f2b31802c3c1bd101cfc82ac9ca2e9ed,
                 0x00c6e1bcaef00f1836c5ab20d1519a810beef7ec7dfb4ebae9a889355b4e623a],
                [0x2918e13f8998780765dbf5c4005a2b35ebef1faba57879ef2694a0066623ee32,
                 0x243f39ec11249d10dafbcbe65b348041ec1f0ace4c08d2bddbe059bfe9dc93f3]
            ),
            c: Pairing.G1Point(
                0x14018dc0751d2d806c9ef0e434f8df20a4a7d2ce44fb28e41ad9b51cc4742a70,
                0x04a841ae4430fe56c0f82aece45bae89ef1af773db41f8763eb417f9537568a7
            )
        });

        bool result = verifier.verifyTx(proof);
        assertFalse(result, "Proof verification failed");
    }
}

```

Now copy the generated solidity file to the source forges ```src``` directory
```
cp ~/zokrates/hash/verifier.sol ~/zokrates/hash/zk-verifier/src/Verifier.sol
```

Inside the forger container, test that the verifier fails when presented the wrong (dummy) proof.
```
$ forge test -vv
[⠊] Compiling...
[⠘] Compiling 1 files with Solc 0.8.28
[⠊] Solc 0.8.28 finished in 747.65ms
Compiler run successful!

Ran 1 test for test/Verifier.t.sol:VerifierTest
[PASS] testVerification() (gas: 220429)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 20.47ms (19.85ms CPU time)

Ran 2 tests for test/Counter.t.sol:CounterTest
[PASS] testFuzz_SetNumber(uint256) (runs: 256, μ: 32043, ~: 32354)
[PASS] test_Increment() (gas: 31851)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 22.50ms (22.07ms CPU time)

Ran 2 test suites in 23.86ms (42.97ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

Create a deployment script ~/zokrates/hash/zk-verifier/scripts/Verifier.s.sol
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "../src/Verifier.sol";

contract VerifierScript is Script {
    function run() external {
        // Use the first test private key from Anvil
        uint256 deployerPrivateKey = 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80;
        
        vm.startBroadcast(deployerPrivateKey);
        
        Verifier verifier = new Verifier();
        console.log("Verifier deployed to:", address(verifier));
        
        vm.stopBroadcast();
    }
}
```

Inside the forge container, deploy the verifier script to local network
```
# Deploy using Anvil's RPC URL
forge script script/Verifier.s.sol:VerifierScript \
--rpc-url http://docker-anvil:8545 \
--broadcast
```





