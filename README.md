# GHO Stablecoin Workshop

In this workshop, you will learn how to write a Solidity smart contract to interact with the Aave-native stablecoin GHO.

## Getting started

To get started, create a new Foundry project:

```sh
forge init gho-workshop
```

Next, move into the new directory and install the following dependencies:

```sh
cd gho-workshop

forge install aave/aave-v3-core@a00f28e aave/gho-core@2abe8f7 OpenZeppelin/openzeppelin-contracts@d00acef
```

Now we need to configure the `remappings.txt` file so Foundry works well with sub-dependencies of the `aave/gho-core` and `aave/aave-v3-core` packages.

```sh
echo "aave-v3-core/=lib/aave-v3-core/
aave-v3-periphery/=lib/aave-address-book/lib/aave-v3-periphery/
ds-test/=lib/forge-std/lib/ds-test/src/
forge-std/=lib/forge-std/src/
gho-core/=lib/gho-core/
@openzeppelin/=lib/openzeppelin-contracts/
" > remappings.txt
```

In addition, we need to add the RPC url we will be using to interact with GHO. Since GHO is deployed in the Goerli network, we need to add the corresponding endpoint to the `foundry.toml` file, so it will have the following code:

```toml
[profile.default]
src = 'src'
out = 'out'
libs = ['lib']

[rpc_endpoints]
goerli = "https://rpc.ankr.com/eth_goerli"
```

### Interacting with GHO

Let's gho through 2 use cases that will clarify how GHO works and how to use it. For that, we will create a test file where we create a test case per use case.

```sh
touch ./test/GhoTest.t.sol
```

The file needs to have the following code:

```Solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/StdCheats.sol";
import {IERC20} from "aave-v3-core/contracts/dependencies/openzeppelin/contracts/IERC20.sol";
import {IPool} from "aave-v3-core/contracts/interfaces/IPool.sol";
import {GhoToken} from "gho-core/src/contracts/gho/GhoToken.sol";
import {IGhoToken} from "gho-core/src/contracts/gho/interfaces/IGhoToken.sol";

contract GhoTest is StdCheats, Test {
    IERC20 dai = IERC20(0xD77b79BE3e85351fF0cbe78f1B58cf8d1064047C);
    IPool pool = IPool(0x617Cf26407193E32a771264fB5e9b8f09715CdfB);
    GhoToken gho = GhoToken(0xcbE9771eD31e761b744D3cB9eF78A1f32DD99211);

    address WE = address(0x1);

    function setUp() public {
        vm.createSelectFork(vm.rpcUrl("goerli"), 8818553);
        // Top up our account with 100 DAI
        deal(address(dai), WE, 100e18);
        // Take control of GHO token
        address owner = gho.owner();
        vm.prank(owner);
        gho.transferOwnership(WE);
        // We start interacting
        vm.startPrank(WE);
    }

    function testMintGho() public {
        // TODO
    }

    function testFacilitator() public {
        // TODO
    }
}
```

This will allow to set up the scenario for our integration use cases:

- Initializes the Pool, DAI and GHO contracts.
- Inits an EOA we will use to perform transactions with some DAI liquidity.
- Takes control of the GHO token so we can perform special actions (e.g. become a facilitator)

#### Mint GHO

GHO works like a regular asset in the Aave Protocol with just an exception: you cannot supply GHO. In order to mint GHO, you will be effectively borrowing GHO against your collateral.

Fill in the `testMintGho` function with the following code and try it out:

```Solidity
    function testMintGho() public {
        // Approve the Aave Pool to pull DAI funds
        dai.approve(address(pool), 100e18);
        // Supply 100 DAI to Aave Pool
        pool.supply(address(dai), 100e18, WE, 0);

        // Mint 10 GHO (2 for variable interest rate mode)
        pool.borrow(address(gho), 10e18, 2, 0, WE);
        assertEq(gho.balanceOf(WE), 10e18);

        // Time flies
        vm.roll(20);

        // We send 10 GHO to a friend
        address FRIEND = address(0x1234);
        gho.transfer(FRIEND, 10e18);
        assertEq(gho.balanceOf(WE), 0);
        assertEq(gho.balanceOf(FRIEND), 10e18);
    }
```

Run the following command to try it out!

```sh
forge t --match-test testMintGho
```

#### Become a Facilitator and mint GHO

The Aave DAO can approve entities with the `Facilitator` role so they can mint and burn GHO tokens. In this test case, we will become a Facilitator and mint some GHO tokens for our friend.

Add the following code to the `testFacilitator` function:

```Solidity
    function testFacilitator() public {
        // Add Facilitator
        gho.addFacilitator(
            WE,
            IGhoToken.Facilitator({
                bucketCapacity: 1_000_000e18, // 1M GHO
                bucketLevel: 0,
                label: "WeFacilitator"
            })
        );

        // Mint 10 GHO to a friend
        address FRIEND = address(0x1234);
        gho.mint(FRIEND, 10e18);
        assertEq(gho.balanceOf(WE), 0);
        assertEq(gho.balanceOf(FRIEND), 10e18);
    }
```

Run the following command to try it out!

```sh
forge t --match-test testFacilitator
```

### Next Steps

Now that you have learned how GHO works, it's time to experiment and create your project on top of it!

Let's buidl!
