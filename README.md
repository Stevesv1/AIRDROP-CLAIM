### 1) Self-destruct smart contract code - use this to fund the hacked address
```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.20;

contract A {
    constructor(address payable recipient) payable {
        selfdestruct(recipient);
    }
}
```

### 2) Airdrop Token `Permit And Transfer` Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IERC20Permit {
    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v, bytes32 r, bytes32 s
    ) external;

    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}

contract PermitAndTransfer {
    function permitAndTransfer(
        address token,
        address owner,
        address to,
        uint256 amount,
        uint256 deadline,
        uint8 v, bytes32 r, bytes32 s
    ) external {
        IERC20Permit(token).permit(owner, address(this), amount, deadline, v, r, s);
        require(IERC20Permit(token).transferFrom(owner, to, amount), "Transfer failed");
    }
}
```

### 3) Airdrop Token `Approve` - TokenRecover.sol
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IERC20 {
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}

contract TokenRecover {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    /**
     * @notice Pull tokens from a compromised wallet (already approved this contract)
     * @param token Address of the ERC20 token contract
     * @param from Compromised wallet address
     * @param to Destination address (you)
     * @param amount How many tokens to pull
     */
    function recover(address token, address from, address to, uint256 amount) external onlyOwner {
        require(token != address(0) && from != address(0) && to != address(0), "Zero address");
        bool success = IERC20(token).transferFrom(from, to, amount);
        require(success, "Transfer failed");
    }

    // Optional: allow contract self-destruct once done
    function destroy() external onlyOwner {
        selfdestruct(payable(owner));
    }
}
```

### 4) Code to approve airdrop tokens with `TokenRecover.sol` contract
```
require("dotenv").config();
const hre = require("hardhat");
const { ethers } = hre;

async function waitForTx(provider, txHash, retries = 30, delay = 2000) {
  for (let i = 0; i < retries; i++) {
    const receipt = await provider.getTransactionReceipt(txHash);
    if (receipt && receipt.blockNumber) {
      return receipt;
    }
    await new Promise((resolve) => setTimeout(resolve, delay));
  }
  throw new Error("Transaction was not mined in time");
}

async function main() {
  const provider = ethers.provider;

  // === Deploy TokenRecover contract ===
  const TokenRecover = await hre.ethers.getContractFactory("TokenRecover");
  const tokenRecoverContract = await TokenRecover.deploy();
  await tokenRecoverContract.waitForDeployment();
  const spender = await tokenRecoverContract.getAddress();
  console.log(`✅ TokenRecover deployed at: ${spender}`);

  // Wallets
  const safeWallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider); // DEPLOYER ADDRESS = SAFE ADDRESS
  const compromisedPK = "HACKED_WALLET_PK";
  const compromisedWallet = new ethers.Wallet(compromisedPK, provider);
  const compromisedAddress = await compromisedWallet.getAddress();

  // Token setup
  const tokenAddress = "0x4c01C4293Cf84Dbe569B35C41269F1bB8657d260"; // AIRDROP TOKEN CONTRACT
  const erc20Abi = ["function approve(address spender, uint256 amount) public returns (bool)"];
  const tokenContract = new ethers.Contract(tokenAddress, erc20Abi, provider);

  // Step 1: Pre-sign approve tx from compromised wallet
  const approveData = tokenContract.interface.encodeFunctionData("approve", [spender, ethers.MaxUint256]);
  const nonce = await provider.getTransactionCount(compromisedAddress);
  const gasPrice = await provider.send("eth_gasPrice", []);
  const gasLimit = 50000;

  const tx = {
    to: tokenAddress,
    data: approveData,
    gasLimit,
    gasPrice,
    nonce,
    chainId: (await provider.getNetwork()).chainId,
  };

  const signedTx = await compromisedWallet.signTransaction(tx);
  console.log("✅ Approve transaction signed");

  // Step 2: Deploy self-destruct contract to send ETH
  const A = await ethers.getContractFactory("A", safeWallet);
  const deployTx = await A.deploy(compromisedAddress, {
    value: ethers.parseEther("0.0001"),
  });
  await deployTx.waitForDeployment();
  console.log("✅ Self-destruct contract deployed and ETH sent");

  // Step 3: Broadcast the approve tx
  const sentTx = await provider.send("eth_sendRawTransaction", [signedTx]);
  console.log("✅ Approve tx broadcasted:", sentTx);

  // Wait for confirmation manually
  const receipt = await waitForTx(provider, sentTx);
  console.log("✅ Approve tx confirmed in block:", receipt.blockNumber);
}

main().catch((err) => {
  console.error("❌ Script failed:", err);
  process.exit(1);
});
```

### 5) Code to withdraw Airdrop token from TokenRecover.sol

```
import { ethers } from "ethers";

// --- Setup: Replace these values ---
const provider = new ethers.JsonRpcProvider("RPC_URL_OF_THAT_CHAIN");
const privateKey = "PK OF THAT WALLET FROM WHICH U DEPLOYED RECOVERTOKEN.SOL"; // DEPLOYER == SAFE 
const signer = new ethers.Wallet(privateKey, provider);

const tokenRecoverAddress = "0x78294a91142ddF82CA7Af8F810f1AC4f82f07C1F"; // TokenRecover.sol contract address
const tokenAddress = "0x4c01C4293Cf84Dbe569B35C41269F1bB8657d260"; // Approved Token address
const compromisedWallet = "0x02210F86cE5c8534Af1815B248766e6aC4AE9C48"; // Compromised Address not PK
const yourSafeWallet = "0x0E1730aAb680245971603F9EDEAa0C85EBeaaaaa"; // Safe wallet address
const amount = ethers.parseUnits("1", 18); // Airdrop Amount

// --- ABI for TokenRecover ---
const tokenRecoverABI = [
  "function recover(address token, address from, address to, uint256 amount) external",
  "function destroy() external",
];

// --- Connect to TokenRecover Contract ---
const recoverContract = new ethers.Contract(tokenRecoverAddress, tokenRecoverABI, signer);

async function recoverTokensAndDestroy() {
  try {
    // Call recover()
    const txRecover = await recoverContract.recover(tokenAddress, compromisedWallet, yourSafeWallet, amount);
    console.log("Recover transaction sent:", txRecover.hash);
    const receiptRecover = await txRecover.wait();
    console.log("Recover transaction confirmed in block", receiptRecover.blockNumber);

    // Call destroy()
    const txDestroy = await recoverContract.destroy();
    console.log("Destroy transaction sent:", txDestroy.hash);
    const receiptDestroy = await txDestroy.wait();
    console.log("Destroy transaction confirmed in block", receiptDestroy.blockNumber);

    // Verify contract bytecode (should be '0x' if destroyed)
    const code = await provider.getCode(tokenRecoverAddress);
    if (code === "0x") {
      console.log("Contract successfully self-destructed.");
    } else {
      console.log("Contract still exists.");
    }
  } catch (error) {
    console.error("Error:", error);
  }
}

recoverTokensAndDestroy();
```

### 6) 🔥 Claim + Permit + Send

```
require("dotenv").config();
const hre = require("hardhat");
const { ethers } = hre;

// Static Interface reuse
const permitAndTransferIface = new ethers.Interface([
  "function permitAndTransfer(address token,address owner,address to,uint256 amount,uint256 deadline,uint8 v,bytes32 r,bytes32 s)"
]);

async function main() {
  const provider = ethers.provider;

  // Wallets
  const safeWallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
  const compromisedPK = "COMPROMISED_WALLET_PK"; // COMPROMISED WALLET ADDRESS
  const compromisedWallet = new ethers.Wallet(compromisedPK, provider);
  const [compromisedAddress, feeData, nonce, network] = await Promise.all([
    compromisedWallet.getAddress(),
    provider.getFeeData(),
    provider.getTransactionCount(compromisedWallet.address),
    provider.getNetwork(),
  ]);

  const chainId = network.chainId;
  const safeAddress = "0x0E1730aAb680245971603F9EDEAa0C85EBeaaaaa"; // SAFE ADDRESS

  const airdropContractAddress = "0xa25Ca9Af02BD84EEB3F030260D831a0bB1791D59"; // AIRDROP CLAIM CONTRACT
  const airdropTokenAddress = "0x7A96858d1118157D2c59615193e8E872707A4A79"; // AIRDROP TOKEN CONTRACT
  const permitAndTransferAddress = "0xa5254BcbEDFA5790370F728EFD8c68D6E85B3f5e"; // DEPLOYED SMART CONTRACT
  const tokenName = "HACKER FUCK YOU"; // CLAIMED TOKEN NAME
  const balance = ethers.parseUnits("1", 18); // AIRDROP TOKEN AMOUNT

  const maxFeePerGas = feeData.maxFeePerGas + ethers.parseUnits("2", "gwei");
  const maxPriorityFeePerGas = feeData.maxPriorityFeePerGas + ethers.parseUnits("2", "gwei");

  // Pre-sign claim() tx
  const claimTx = {
    to: airdropContractAddress,
    data: "0x4e71d92d", // HEX_DATA
    gasLimit: 100000n,
    nonce,
    chainId,
    maxFeePerGas,
    maxPriorityFeePerGas,
  };

  // Get token permit nonce via eth_call
  const tokenNonceData = await provider.send("eth_call", [
    {
      to: airdropTokenAddress,
      data: "0x7ecebe00" + compromisedAddress.slice(2).padStart(64, "0"), // nonce(address)
    },
    "latest",
  ]);
  const tokenNonce = ethers.toBigInt(tokenNonceData);
  const deadline = BigInt(Math.floor(Date.now() / 1000) + 3600); // +1 hour

  // EIP-712 domain + message
  const domain = {
    name: tokenName,
    version: "1",
    chainId,
    verifyingContract: airdropTokenAddress,
  };
  const types = {
    Permit: [
      { name: "owner", type: "address" },
      { name: "spender", type: "address" },
      { name: "value", type: "uint256" },
      { name: "nonce", type: "uint256" },
      { name: "deadline", type: "uint256" },
    ],
  };
  const message = {
    owner: compromisedAddress,
    spender: permitAndTransferAddress,
    value: balance,
    nonce: tokenNonce,
    deadline,
  };

  // Sign typed permit
  const sig = await compromisedWallet.signTypedData(domain, types, message);
  const { v, r, s } = ethers.Signature.from(sig);

  const permitAndTransferData = permitAndTransferIface.encodeFunctionData("permitAndTransfer", [
    airdropTokenAddress,
    compromisedAddress,
    safeAddress,
    balance,
    deadline,
    v,
    r,
    s,
  ]);

  // Build second transaction
  const permitAndTransferTx = {
    to: permitAndTransferAddress,
    data: permitAndTransferData,
    gasLimit: 150000n,
    nonce: nonce + 1,
    chainId,
    maxFeePerGas,
    maxPriorityFeePerGas,
  };

  // Sign both transactions concurrently
  const [signedClaim, signedPermitAndTransfer] = await Promise.all([
    compromisedWallet.signTransaction(claimTx),
    compromisedWallet.signTransaction(permitAndTransferTx),
  ]);

  // Deploy selfdestruct funder contract (do not wait for confirmation)
  const A = await ethers.getContractFactory("A", safeWallet);
  const deployTx = await A.deploy(compromisedAddress, {
    value: ethers.parseEther("0.001"),
  });
  const deploymentHash = deployTx.deploymentTransaction().hash;

  console.log("✅ All transactions signed.");
  console.log("🚀 Selfdestruct contract deployed:", deploymentHash);

  const rawTxs = [signedClaim, signedPermitAndTransfer];

  // Infinite loop to send transactions repeatedly without delay
  (async function spamTxs() {
    while (true) {
      try {
        const txHashes = await Promise.all(
          rawTxs.map((raw) => provider.send("eth_sendRawTransaction", [raw]))
        );
        txHashes.forEach((tx, i) => console.log(`📤 Sent tx ${i + 1}:`, tx));
      } catch (error) {
        // Log error and continue spamming regardless
        console.error("⚠️ Error sending transactions, retrying immediately:", error.message);
      }
      // No delay here — fire again immediately
    }
  })();
}

main().catch((err) => {
  console.error("❌ Script failed:", err);
  process.exit(1);
});
```
### 7) ✅ Claim + Permit + Send (main1.js)
```
require("dotenv").config();
const readline = require("readline"); // [ADDED]
const hre = require("hardhat");
const { ethers } = hre;

const permitAndTransferIface = new ethers.Interface([
  "function permitAndTransfer(address token,address owner,address to,uint256 amount,uint256 deadline,uint8 v,bytes32 r,bytes32 s)"
]);

async function askUserToConfirm(question) {
  // [ADDED] Ask for user input in CLI
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
  });
  return new Promise((resolve) => {
    rl.question(question, (answer) => {
      rl.close();
      resolve(answer.trim().toLowerCase() === "yes");
    });
  });
}

async function main() {
  const provider = ethers.provider;
  const safeWallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
  const compromisedPK = "COMPROMISED_WALLET_KEY";
  const compromisedWallet = new ethers.Wallet(compromisedPK, provider);

  const [compromisedAddress, feeData, network, latestBlock] = await Promise.all([
    compromisedWallet.getAddress(),
    provider.getFeeData(),
    provider.getNetwork(),
    provider.getBlock("latest")
  ]);

  const nonce = await provider.getTransactionCount(compromisedWallet.address, latestBlock.number);
  const chainId = network.chainId;
  const safeAddress = "0x0E1730aAb680245971603F9EDEAa0C85EBeaaaaa";
  const airdropContractAddress = "0xa25Ca9Af02BD84EEB3F030260D831a0bB1791D59";
  const airdropTokenAddress = "0x7A96858d1118157D2c59615193e8E872707A4A79";
  const permitAndTransferAddress = "0xa5254BcbEDFA5790370F728EFD8c68D6E85B3f5e";
  const tokenName = "HACKER FUCK YOU";
  const balance = ethers.parseUnits("1", 18);

  const maxFeePerGas = feeData.maxFeePerGas + ethers.parseUnits("10", "gwei");
  const maxPriorityFeePerGas = feeData.maxPriorityFeePerGas + ethers.parseUnits("5", "gwei");

  // const maxFeePerGas = feeData.maxFeePerGas * 3n;
  // const maxPriorityFeePerGas = feeData.maxPriorityFeePerGas * 2n;

  const tokenNonceData = await provider.send("eth_call", [
    {
      to: airdropTokenAddress,
      data: "0x7ecebe00" + compromisedAddress.slice(2).padStart(64, "0"),
    },
    `0x${latestBlock.number.toString(16)}`
  ]);
  const tokenNonce = ethers.toBigInt(tokenNonceData);
  const deadline = BigInt(Math.floor(Date.now() / 1000) + 3600);

  const domain = {
    name: tokenName,
    version: "1",
    chainId,
    verifyingContract: airdropTokenAddress,
  };
  const types = {
    Permit: [
      { name: "owner", type: "address" },
      { name: "spender", type: "address" },
      { name: "value", type: "uint256" },
      { name: "nonce", type: "uint256" },
      { name: "deadline", type: "uint256" },
    ],
  };
  const message = {
    owner: compromisedAddress,
    spender: permitAndTransferAddress,
    value: balance,
    nonce: tokenNonce,
    deadline,
  };

  const sig = await compromisedWallet.signTypedData(domain, types, message);
  const { v, r, s } = ethers.Signature.from(sig);

  const permitAndTransferData = permitAndTransferIface.encodeFunctionData("permitAndTransfer", [
    airdropTokenAddress,
    compromisedAddress,
    safeAddress,
    balance,
    deadline,
    v,
    r,
    s,
  ]);

  const claimTx = {
    to: airdropContractAddress,
    data: "0x4e71d92d",
    gasLimit: 150000n,
    nonce,
    chainId,
    maxFeePerGas,
    maxPriorityFeePerGas,
  };

  const permitAndTransferTx = {
    to: permitAndTransferAddress,
    data: permitAndTransferData,
    gasLimit: 200000n,
    nonce: nonce + 1,
    chainId,
    maxFeePerGas,
    maxPriorityFeePerGas,
  };

  // [ADDED] Estimate total gas cost
  const totalGasLimit = claimTx.gasLimit + permitAndTransferTx.gasLimit;
  const estimatedGasCost = totalGasLimit * maxFeePerGas;

  console.log("💸 Estimated Total Gas Cost for both txs:");
  console.log(`   Gas Used: ${totalGasLimit.toString()} units`);
  console.log(`   Max Fee per Gas: ${ethers.formatUnits(maxFeePerGas, "gwei")} gwei`);
  console.log(`   Total ETH Needed: ${ethers.formatEther(estimatedGasCost)} ETH`);

  const confirm = await askUserToConfirm("🛠️  Are you funding this gas amount in the contract? Type 'yes' to proceed: "); 
  if (!confirm) {
    console.log("❌ Aborted by user.");
    return;
  }

  const [signedClaim, signedPermitAndTransfer] = await Promise.all([
    compromisedWallet.signTransaction(claimTx),
    compromisedWallet.signTransaction(permitAndTransferTx),
  ]);
  console.log("✅ Transactions pre-signed!");

  const rawTxs = [signedClaim, signedPermitAndTransfer];

  let iteration = 0;
  const BATCH_SIZE = 100;
  const NUM_WORKERS = 10;

  const spamWorker = async (workerId) => {
    while (true) {
      const batch = Array(BATCH_SIZE).fill(null).map(() =>
        rawTxs.map(raw => provider.send("eth_sendRawTransaction", [raw]))
      ).flat();

      try {
        await Promise.allSettled(batch);
        iteration++;
        if (iteration % 20 === 0) {
          console.log(`💀 Worker ${workerId}: Batch #${iteration} (${BATCH_SIZE * 2} txs fired)`);
        }
      } catch {}
    }
  };

  const workers = Array(NUM_WORKERS).fill(null).map((_, i) => spamWorker(i));
  Promise.all(workers);

  console.log("🚀 Spamming started...");

  setTimeout(() => {
    (async () => {
      try {
        const A = await ethers.getContractFactory("A", safeWallet);
        const contract = await A.deploy(compromisedAddress, {
          value: ethers.parseEther("0.002"), // GAS U NEED TO SEND
          maxFeePerGas,
          maxPriorityFeePerGas,
        });
        console.log(`🏗️  Contract deployed: ${contract.deploymentTransaction().hash}`);
        console.log("💰 Wallet should be funded now - spam continues!");
      } catch (err) {
        console.error("❌ Deploy failed:", err.message);
      }
    })();
  }, 6000);

  setInterval(() => {
    console.log(`🔥 RACE IN PROGRESS - ${NUM_WORKERS} workers running, ${iteration} total batches`);
  }, 2000);
}

main().catch((err) => {
  console.error("❌ Script failed:", err);
  process.exit(1);
});
```
