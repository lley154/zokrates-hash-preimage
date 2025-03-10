# Zokrates Hash Preimage
The goal is to compute the hash for the number 5 and then prove a user knows the preimage number 5 but doesn't have to reveal it.

### Setup
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
### Create Zokrate files
Complete all of the steps in the following example https://zokrates.github.io/examples/sha256example.html

### Launch Foundry
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

### Unit Testing
Open a new terminal window and create a test file in ```~/zokrates/hash/zk-verifier/test/Verifier.t.sol```
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
                0x093f4eed52256476b4bce850f5dad3b40f895f26ef252d4b44ccdf773e423b0d,
                0x0abd19f4ad34143f3e355e9b02e0833b11f01003f321ef100eb426954c38f405
            ),
            b: Pairing.G2Point(
                [0x2e2a1ef324cba4806fecba90bd1bc1ed80060f35dfd32e39b6462f252a214197,
                 0x23376b37bddc4ce3c83a7adf282d8cd1bcaefd8697c2b29cb0e9944decabeec0],
                [0x1705791c96252eb31e6559ee8037c468d71e523ec8e6386618422024cf646b42,
                 0x004f96531b514308c0f0f50d664ecac1dbca89b9e20a4a491778c5896680a7ac]
            ),
            c: Pairing.G1Point(
                0x0ba1512fa348d0e0732f8c0b31884628fc2cfc5b3d0caf38ac9f82beaa3bb0a2,
                0x2a58e67ccb61bc6070061713109e50b31cc279a8e45d2455ba94d895810d5ffc
            )
        });

        bool result = verifier.verifyTx(proof);
        assertTrue(result, "Proof verification failed");
    }
}
```
Replace the Pairing Points with updated values that can be found in the ```~/zokrates/proof.json```

Now copy the generated solidity file to the source forges ```src``` directory
```
cp ~/zokrates/hash/verifier.sol ~/zokrates/hash/zk-verifier/src/Verifier.sol
```

Inside the forge container, run the test to confirm success
```
$ forge test -vv
[⠊] Compiling...
[⠘] Compiling 1 files with Solc 0.8.28
[⠃] Solc 0.8.28 finished in 725.06ms
Compiler run successful!

Ran 1 test for test/Verifier.t.sol:VerifierTest
[PASS] testVerification() (gas: 220429)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 9.68ms (9.15ms CPU time)

Ran 2 tests for test/Counter.t.sol:CounterTest
[PASS] testFuzz_SetNumber(uint256) (runs: 256, μ: 31887, ~: 32354)
[PASS] test_Increment() (gas: 31851)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 14.98ms (14.26ms CPU time)

Ran 2 test suites in 16.89ms (24.66ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```
### Integration Testing
Create a deployment script ```~/zokrates/hash/zk-verifier/scripts/DeployVerifier.s.sol```
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script} from "forge-std/Script.sol";
import {Verifier} from "../src/Verifier.sol";

contract DeployVerifier is Script {
    function run() external returns (Verifier) {
        vm.startBroadcast();
        Verifier verifier = new Verifier();
        vm.stopBroadcast();
        return verifier;
    }
} 
```

Now create the verifier script ```~/zokrates/hash/zk-verifier/scripts/VerifyProof.s.sol```
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script} from "forge-std/Script.sol";
import {Verifier} from "../src/Verifier.sol";
import {Pairing} from "../src/Verifier.sol";

contract VerifyProof is Script {
    function run() external {
        // Get the contract address from command line arguments
        address verifierAddress = vm.envAddress("VERIFIER_ADDRESS");
        
        Verifier verifier = Verifier(verifierAddress);
        
        // These values match the verification key in the contract
        Verifier.Proof memory proof = Verifier.Proof({
            a: Pairing.G1Point(
                0x093f4eed52256476b4bce850f5dad3b40f895f26ef252d4b44ccdf773e423b0d,
                0x0abd19f4ad34143f3e355e9b02e0833b11f01003f321ef100eb426954c38f405
            ),
            b: Pairing.G2Point(
                [0x2e2a1ef324cba4806fecba90bd1bc1ed80060f35dfd32e39b6462f252a214197,
                 0x23376b37bddc4ce3c83a7adf282d8cd1bcaefd8697c2b29cb0e9944decabeec0],
                [0x1705791c96252eb31e6559ee8037c468d71e523ec8e6386618422024cf646b42,
                 0x004f96531b514308c0f0f50d664ecac1dbca89b9e20a4a491778c5896680a7ac]
            ),
            c: Pairing.G1Point(
                0x0ba1512fa348d0e0732f8c0b31884628fc2cfc5b3d0caf38ac9f82beaa3bb0a2,
                0x2a58e67ccb61bc6070061713109e50b31cc279a8e45d2455ba94d895810d5ffc
            )
        });

        vm.startBroadcast();
        bool result = verifier.verifyTx(proof);
        vm.stopBroadcast();

        require(result, "Proof verification failed");
    }
} 
```
Replace the Pairing Points with updated values that can be found in the ```~/zokrates/proof.json```

#### Deploy the contract
```
# In another terminal:
# Deploy
$ forge script script/DeployVerifier.s.sol:DeployVerifier --rpc-url http://localhost:8545 --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 --broadcast
[⠊] Compiling...
No files changed, compilation skipped
Script ran successfully.

== Return ==
0: contract Verifier 0x5FbDB2315678afecb367f032d93F642f64180aa3

## Setting up 1 EVM.

==========================

Chain 31337

Estimated gas price: 2.000000001 gwei

Estimated total gas used for script: 1391588

Estimated amount required: 0.002783176001391588 ETH

==========================

##### anvil-hardhat
✅  [Success] Hash: 0xd0ed0b0bc3ac4302127d23295ddb6fd5dc37bd53efc1502aa3c3d06145c52d25
Contract Address: 0x5FbDB2315678afecb367f032d93F642f64180aa3
Block: 1
Paid: 0.001070453001070453 ETH (1070453 gas * 1.000000001 gwei)

✅ Sequence #1 on anvil-hardhat | Total Paid: 0.001070453001070453 ETH (1070453 gas * avg 1.000000001 gwei)
                                                                                               

==========================

ONCHAIN EXECUTION COMPLETE & SUCCESSFUL.

Transactions saved to: /workspace/hash/zk-verifier/broadcast/DeployVerifier.s.sol/31337/run-latest.json

Sensitive values saved to: /workspace/hash/zk-verifier/cache/DeployVerifier.s.sol/31337/run-latest.json

```

#### Execute and verify (set the VERIFIER_ADDRESS to Contract Address from deployment)
```
$ export VERIFIER_ADDRESS=0x5FbDB2315678afecb367f032d93F642f64180aa3
$ forge script script/VerifyProof.s.sol:VerifyProof --rpc-url http://localhost:8545 --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 --broadcast
[⠊] Compiling...
[⠘] Compiling 1 files with Solc 0.8.28
[⠃] Solc 0.8.28 finished in 657.75ms
Compiler run successful!
Script ran successfully.
```

Change one of the Pairing Points to ```0x00d6f17b9eb558d2e8b8d3a653ded8edaf5d2314224509447600eb71ae76e6cb``` and execute again ```VerifyProof.s.sol``` and see what happens.

