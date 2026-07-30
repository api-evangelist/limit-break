# LBAMM Local Development Environment

LBAMM is currently in audit and not yet deployed to public networks. To test LBAMM-adjacent contracts locally, use the `lbamm-examples` repository which provides a complete local environment. It deploys everything from `apptoken-dev` (Transfer Validator, PermitC, Trusted Forwarder, Payment Processor, TokenMaster, Wrapped Native) plus the full LBAMM stack (AMM core, pool types, hooks, handlers). If you only need the base Apptoken protocols without LBAMM, use `apptoken-dev` directly -- see `apptoken-general/references/project-setup.md`.

## Setup

```bash
git clone https://github.com/limitbreakinc/lbamm-examples.git
cd lbamm-examples/environment/lbamm-deploy
bash script/start.sh
```

This single command:
1. Starts an Anvil node on `localhost:8545` (chain ID 1776411)
2. Deploys all infrastructure: Transfer Validator V5, Payment Processor, TokenMaster, Wrapped Native
3. Deploys all LBAMM contracts: AMM core, pool types, hooks, handlers
4. Writes all deployed addresses to the root `.env` file

## Environment Variables

After `start.sh` completes, the root `.env` file contains all deployed contract addresses. Key variables:

| Variable | Description |
|----------|-------------|
| `RPC_URL` | `http://localhost:8545` |
| `LIMIT_BREAK_AMM` | Main AMM contract address |
| `TRANSFER_VALIDATOR` | Transfer Validator V5 |
| `SETTINGS_REGISTRY` | Creator Hook Settings Registry |
| `STANDARD_HOOK` | AMM Standard Hook |
| `DYNAMIC_POOL_TYPE` | Dynamic pool type address |
| `FIXED_POOL_TYPE` | Fixed pool type address |
| `SINGLE_PROVIDER_POOL_TYPE` | Single provider pool type address |
| `PERMIT_TRANSFER_HANDLER` | Permit-based swap handler |
| `CLOB_TRANSFER_HANDLER` | CLOB order book handler |

## Using .env in Generated Scripts

Foundry scripts should load addresses from environment variables so they work against the local dev environment:

```solidity
import "forge-std/Script.sol";

contract DeployMyToken is Script {
    function run() external {
        address amm = vm.envAddress("LIMIT_BREAK_AMM");
        address transferValidator = vm.envAddress("TRANSFER_VALIDATOR");
        address settingsRegistry = vm.envAddress("SETTINGS_REGISTRY");
        address standardHook = vm.envAddress("STANDARD_HOOK");
        address dynamicPoolType = vm.envAddress("DYNAMIC_POOL_TYPE");

        vm.startBroadcast();
        // ... deploy and configure token ...
        vm.stopBroadcast();
    }
}
```

Run with:
```bash
source /path/to/lbamm-examples/.env
forge script script/DeployMyToken.s.sol --rpc-url $RPC_URL --broadcast
```

## Default Anvil Account

The deployer private key for local testing:
```
0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```
Address: `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`

## Example: Trading Tax Token

The `examples/trading-tax-token/` directory in `lbamm-examples` demonstrates the full flow:
1. Deploy an ERC-20C token
2. Configure Transfer Validator (whitelist-only, OTC disabled)
3. Set up 10% paired token tax via AMM Standard Hook
4. Create a dynamic pool (token / Wrapped Native)
5. Seed single-sided liquidity

This is a good reference pattern for any LBAMM-adjacent token deployment.

## Stopping

```bash
cd lbamm-examples/environment/lbamm-deploy
bash script/stop.sh
```
