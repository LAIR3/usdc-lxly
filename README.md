# USDC LXLY

## Description

[USDC LxLy](https://docs.google.com/document/d/1heUd3Cbux-ngnCJITbKJ9pdsz26BmNz1hfOn9NTuDH8/edit?pli=1)

## Contracts

- ~~USDC.e (zkEVM) - This contract will match the current USDC contract deployed on Ethereum, with all expected features. Its contract address will be different from the current “wrapped” USDC in use today, and will have the ability to issue and burn tokens as well as “blacklist” addresses.~~ [usdc-e project](https://github.com/omnifient/usdc-e)

- BridgeMinter (zkEVM) - This contract will receive messages from the LXLY bridge on zkEVM, it will hold the minter role giving it the ability to mint USDC.e based on instructions from LXLY from Ethereum only. This contract will be upgradable.

- BridgeBurner (zkEVM) - This contract will send messages to LXLY bridge on zkEVM, it will hold the burner role giving it the ability to burn USDC.e based on instructions from LXLY, triggering a release of assets on L1Escrow. This contract will be upgradable.

- L1Escrow (L1) - This contract will receive USDC from users on L1 and trigger BridgeMinter on the zkEVM via LxLy. This contract will be upgradable. This contract will hold all of the backing for USDC on zkEVM.

- NativeConverter (zkEVM) - This contract will receive BridgeWrappedUSDC on zkEVM and issue USDC.e on zkEVM. This contract will hold the minter role giving it the ability to mint USDC.e based on inflows of BridgeWrappedUSDC. This contract will also have a permissionless publicly callable function called “migrate” which when called will withdraw all BridgedWrappedUSDC to L1 via the LXLY bridge. The beneficiary address will be the L1Escrow, thus migrating the supply and settling the balance. This contract will be upgradable and pausable.

## Flows

- User Bridges from L1 to zkEVM (post upgrade to USDC.e)
  - User calls deposit() on L1Escrow, L1_USDC transferred to L1Escrow, message sent to zkEVMBridge targeted to zkEVM’s BridgeMinter.
  - Message claimed and sent to BridgeMinter, which calls mint() on NativeUSDC which mints new supply to the correct address.
- User Bridges from zkEVM to L1 (post upgrade to USDC.e)
  - User calls withdraw() on BridgeBurner which calls burn() on NativeUSDC burning the supply. Message is sent to zkEVMBridge targeted to L1Escrow.
  - Message claimed and sent to L1Escrow, which transfers L1_USDC to the correct address.
- User converts BridgeWrappedUSDC to USDC.e
  - User calls convert() on NativeConverter, BridgeWrappedUSDC is transferred to NativeConverter. NativeConverter calls mint() on NativeUSDC which mints new supply to the correct address.
  - Anyone can call migrate() on NativeConverter to have all BridgeWrappedUSDC withdrawn via the zkEVMBridge moving the L1_USDC held in the zkEVMBridge to L1Escrow.

## Testing

TODO: write a script to automate this

```bash
# 1 start anvil

## 1.1 start L1 (ethereum mainnet) anvil
## NOTE: using port 8001 for L1
anvil --fork-url <https://eth-mainnet.g.alchemy.com/v2/YOUR_ALCHEMY_KEY> --chain-id 1 --port 8001 --fork-block-number 17785773

## 1.2 start L2 (polygon zkevm) anvil
## NOTE: using port 8101 for L2
anvil --fork-url <https://polygonzkevm-mainnet.g.alchemy.com/v2/YOUR_ALCHEMY_KEY> --chain-id 1101 --port 8101 --fork-block-number 3172683

# 2. deploy and initialize usdc-e to L2
cd usdc-e/
forge script script/DeployInitUSDCE.s.sol:DeployInitUSDCE --fork-url http://localhost:8101 --broadcast --verify -vvvv

# 3. copy the output address (to be used in step 4)
FiatTokenV2_1@0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512

# 4. paste the L2_USDC address into usdc-lxly/.env
cd usdc-lxly/
ADDRESS_L2_USDC=0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512

# 5. run the usdc-lxly tests
cd usdc-lxly/
forge test -vvvvv
```

## Deployment

First, copy `.env.example` to `.env` and set the appropriate environment variables

### Deployment to Mainnet Forks

Note:

- using 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 as the admin for USDC-E
- using 0xa0Ee7A142d267C1f36714E4a8F75612F20a79720 as the admin+owner for LXLY contracts

```bash
# 1 start anvil

## 1.1 start L1 (ethereum mainnet) anvil
## NOTE: using port 8001 for L1
anvil --fork-url <https://eth-mainnet.g.alchemy.com/v2/YOUR_ALCHEMY_KEY> --chain-id 1 --port 8001 --fork-block-number 17785773

## 1.2 start L2 (polygon zkevm) anvil
## NOTE: using port 8101 for L2
anvil --fork-url <https://polygonzkevm-mainnet.g.alchemy.com/v2/YOUR_ALCHEMY_KEY> --chain-id 1101 --port 8101 --fork-block-number 3172683

# 2. deploy and initialize usdc-e to L2
cd usdc-e/
forge script script/DeployInitUSDCE.s.sol:DeployInitUSDCE --fork-url http://localhost:8101 --broadcast --verify -vvvv

# 3. copy the output address
FiatTokenV2_1@0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512

# 4. and paste into usdc-lxly/.env
ADDRESS_L2_USDC=0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512

# 5. deploy and initialize usdc-lxly
cd usdc-lxly/
forge script scripts/DeployInit.s.sol:DeployInit --broadcast -vvvv
