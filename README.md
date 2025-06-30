### **How to Read This Documentation**

To understand the full process of minting NFTs, follow this reading flow:

1. **Start with the Minting Logic ( Step 5)**
    - Begin by reading the **Minting Logic** section. This is the core of the entire process. It explains how the minting flow works, how the MetaMask wallet is connected, and how token balances are checked before minting.
2. **Then, Dive into Minting Functions (Step 4)**
    - After understanding the main minting logic, explore the **Minting Functions** section. This explains how the minting is handled using either **Stablecoin (ERC20 tokens)** or **ETH**.
3. **Next, Check the Helper Functions (Step 3)**
    - Once you're familiar with how minting is executed, read through the **Helper Functions**. These functions handle MetaMask connection, fetching balances, getting mint prices, and managing the RPC URL. You'll understand how each piece of the puzzle fits together.
4. **Finally, Go Through the Code in Order** 
    - After reading the individual sections, the best way to solidify your understanding is to go through the **full code** in order (from **1. Introduction > 5.Minting Logic > 4.Minting Functions > 3.Helper Functions and 2.Contract ABI**) to see how the entire flow integrates.

## **NFT Minting with MetaMask and StableCoin / ETH**

This document explains the entire process and code used to mint NFTs using MetaMask, supporting both **StableCoin** (ERC20 tokens) and **ETH** (native token) for payment. It includes wallet connection, token balance checking, mint price retrieval, and minting using either ERC20 tokens or ETH.

---

### **1. Introduction**

The goal of this implementation is to:

- Allow users to connect their MetaMask wallet to the DApp.
- Check whether a stablecoin (ERC20 token) or ETH is used for minting.
- Check if the user has sufficient funds (either stablecoins or ETH).
- Mint an ERC721 NFT using the selected token or ETH.

---

### **2. Contract ABIs**

Two contract ABIs are used: one for **ERC721 NFT** and one for **ERC20 token**.

### **ERC721 Contract ABI:**

This ABI defines functions to interact with the ERC721 NFT contract.

```jsx

const ERC721_ABI = [
  "function isTokenWhitelisted(address tokenAddress) public view returns (bool)",
  "function getTokenAmount(address tokenAddress) public view returns (uint256)",
  "function mintWithToken(address tokenAddress) public",
  "function mintPrice() public view returns (uint256)",
  "function mint() public payable",
]

```

### **ERC20 Contract ABI:**

This ABI defines functions for interacting with an ERC20 token contract.

```jsx

const ERC20_ABI = [
  "function approve(address spender, uint256 amount) external returns (bool)",
  "function allowance(address owner, address spender) external view returns (uint256)",
  "function balanceOf(address account) external view returns (uint256)",
  "function totalSupply() external view returns (uint256)",
]

```

---

### **3. Helper Functions**

### **3.1. Get RPC URL**

```jsx

export const getRpcUrl = (chainId) => {
  const idAndUrls = evmRPCURL.split("~");

  for (let i = 0; i < idAndUrls?.length; ++i) {
    const [id, url] = idAndUrls[i].split(",");

    if (chainId.toString() === id) {
      return url;
    }
  }

  return null;
}

```

- This function retrieves the RPC URL based on the given `chainId`.
- It maps the chain ID to its corresponding URL from the environment variable `VITE_EVM_RPC_URL`.

### **3.2. Get MetaMask Provider**

```jsx

export const getMetaMaskProvider = () => {
  if (window.ethereum && window.ethereum.providers) {
    const providers = window.ethereum.providers;
    const metaMaskProvider = providers.find((provider) => provider.isMetaMask);
    return metaMaskProvider;
  } else if (window.ethereum && window.ethereum.isMetaMask) {
    return window.ethereum;
  } else {
    return null;
  }
}

```

- This function checks if MetaMask is installed and returns the MetaMask provider.
- It handles cases where multiple wallet providers are available and specifically looks for the MetaMask provider.

### **3.3. MetaMask Connection**

```jsx

export const metaConnection = async (chainId) => {
  const metaMaskProvider = getMetaMaskProvider();

  if (!metaMaskProvider) {
    const installUrl = "https://metamask.io/download.html";
    toast.info("Opening MetaMask installation page in a new tab...");
    window.open(installUrl, "_blank");
    return;
  }

  try {
    const accounts = await metaMaskProvider.request({ method: "wallet_requestPermissions", params: [{ eth_accounts: {} }] });
    const selectedAccounts = await metaMaskProvider.request({ method: "eth_requestAccounts" });

    if (!selectedAccounts || selectedAccounts.length === 0) {
      toast.error("No wallet selected. Connection cancelled.");
      return;
    }

    const userChainId = await metaMaskProvider.request({ method: "eth_chainId" });
    const targetChainId = `0x${parseInt(chainId, 10).toString(16)}`;

    if (userChainId !== targetChainId) {
      try {
        await metaMaskProvider.request({ method: "wallet_switchEthereumChain", params: [{ chainId: targetChainId }] });
        toast.success(`Switched to chain ID: ${targetChainId}`);
      } catch (switchError) {
        if (switchError.code === 4902) {
          try {
            const chainDetails = await fetchChainDetails(chainId);
            await metaMaskProvider.request({ method: "wallet_addEthereumChain", params: [chainDetails] });
            toast.success(`Added and switched to chain ID: ${targetChainId}`);
          } catch (addError) {
            toast.error("Failed to add or switch to the required chain");
            return "chainerror";
          }
        } else {
          toast.error("Please change your chain network manually");
          return "chainerror";
        }
      }
    }

    return selectedAccounts[0];
  } catch (error) {
    console.error("MetaMask connection error:", error);
    toast.error("Failed to connect MetaMask wallet.");
    return "chainerror";
  }
}

```

- This function connects the MetaMask wallet to the DApp, handles account selection, and ensures the user is on the correct network.
- If the user's network is different, it attempts to switch to the correct one or prompts them to do so manually.

### **3.4. Get Mint Price for Stablecoin**

```jsx
export const getMintPriceForEVMInStableCoin = async (contractAddress, chainId, erc20Address) => {
  const provider = new ethers.JsonRpcProvider(getRpcUrl(chainId));
  const nftContract = new ethers.Contract(contractAddress, ERC721_ABI, provider);

  try {
    const whitelisted = await nftContract.isTokenWhitelisted(erc20Address);
    if (!whitelisted) {
      throw new Error("Token not whitelisted for minting");
    }
    const mintPrice = await nftContract.getTokenAmount(erc20Address);
    return mintPrice;
  } catch (error) {
    throw error;
  }
}

```

- This function checks if the ERC20 token is whitelisted for minting and retrieves the mint price for the selected stablecoin.

### **3.5. Get Token Balance**

```jsx

export const getTokenBalance = async (chainId, tokenAddress, walletAddress) => {
  const provider = new ethers.JsonRpcProvider(getRpcUrl(chainId));
  const erc20Contract = new ethers.Contract(tokenAddress, ERC20_ABI, provider);
  const senderBalance = await erc20Contract.balanceOf(walletAddress);
  return senderBalance;
}

```

- This function checks the user's token balance for a given ERC20 token.

---

### **4. Minting Functions**

### **4.1. Mint NFT with Stablecoin**

```jsx
export const mintNftWithToken = async (erc20Address, nftAddress, chainId) => {
  try {
    const metaMaskProvider = getMetaMaskProvider();
    const accounts = await metaMaskProvider.request({ method: "eth_requestAccounts" });
    const from = accounts[0];

    const web3Provider = new ethers.BrowserProvider(metaMaskProvider);
    const signer = await web3Provider.getSigner();

    const nftContract = new ethers.Contract(nftAddress, ERC721_ABI, signer);
    const erc20Contract = new ethers.Contract(erc20Address, ERC20_ABI, signer);

    const whitelisted = await nftContract.isTokenWhitelisted(erc20Address);
    if (!whitelisted) {
      throw new Error("Token not whitelisted for minting");
    }

    const mintPrice = await nftContract.getTokenAmount(erc20Address);

    const currentAllowance = await erc20Contract.allowance(from, nftAddress);
    if (currentAllowance < mintPrice) {
      const approveTx = await erc20Contract.approve(nftAddress, mintPrice);
      await approveTx.wait();
    }

    const finalAllowance = await erc20Contract.allowance(from, nftAddress);
    if (finalAllowance < mintPrice) {
      throw new Error(`Insufficient allowance: ${finalAllowance.toString()} < ${mintPrice.toString()}`);
    }

    const mintTx = await nftContract.mintWithToken(erc20Address);
    const mintReceipt = await mintTx.wait();

    return { success: true, result: mintTx.hash, receipt: mintReceipt };
  } catch (error) {
    console.error("Minting error:", error);
    return { success: false, code: error?.message || "Unknown error occurred" };
  }
}

```

- This function handles the minting process with stablecoin (ERC20 token).
- It checks if the token is whitelisted, verifies the allowance, approves the transfer, and mints the NFT.

### **4.2. Mint NFT with ETH**

```jsx

export const mintNFT = async (contractAddress, chainId) => {
  const metaMaskProvider = getMetaMaskProvider();
  try {
    const accounts = await metaMaskProvider.request({ method: "eth_requestAccounts" });
    const from = accounts[0];

    const web3Provider = new ethers.BrowserProvider(metaMaskProvider);
    const signer = await web3Provider.getSigner();
    const nftContract = new ethers.Contract(contractAddress, ERC721_ABI, signer);

    const mintPrice = await nftContract.mintPrice();
    const mintData = new ethers.Interface([ "function mint() public payable" ]).encodeFunctionData("mint", []);
    const transactionParams = {
      to: contractAddress,
      from: from,
      data: mintData,
      value: "0x" + mintPrice.toString(16),
    };

    const result = await metaMaskProvider.request({ method: "eth_sendTransaction", params: [transactionParams] });
    return { success: true, result };
  } catch (error) {
    return { success: false, code: error?.message };
  }
}

```

- This function handles the minting process using ETH.
- It retrieves the mint price and sends a transaction with the required ETH payment.

---

### **5. Full Minting Logic**

The full minting logic uses the above helper functions to:

1. Connect MetaMask.
2. Check the balance (either ERC20 token or ETH).
3. Call the appropriate minting function (`mintNftWithToken` or `mintNFT`).

```jsx

try {
  startLoading();  // Start the loading indicator

  const metaMaskProvider = getMetaMaskProvider();
  const metaConnectionResponse = await metaConnection(chainId);

  if (metaConnectionResponse === "chainerror") {
    return;
  } else {
    if (data?.token?.isUSDStableCoin) {
      const requiredTokenAmount = await getMintPriceForEVMInStableCoin(contractAddress, chainId, data?.token?.address);
      const tokenBalance = await getTokenBalance(chainId, data?.token?.address, metaConnectionResponse);

      if (tokenBalance < requiredTokenAmount) {
        toast.error(`Insufficient ${data?.token?.name} : You need ${Number(Number(requiredTokenAmount) / 10 ** data?.token?.decimals)?.toFixed(2)} ${data?.token?.symbol}`);
        return;
      }

      const { success, code, result } = await mintNftWithToken(data?.token?.address, contractAddress, chainId);

      if (!success) {
        toast.error(code);
        return;
      }
      if (success) {
        // Handle successful minting
      }
    } else {
      const provider = new BrowserProvider(metaMaskProvider);
      const signer = await provider.getSigner();
      const address = await signer.getAddress();
      const balance = await provider.getBalance(address);
      const balanceEth = parseFloat(formatEther(balance));

      const requiredEth = await getMintPriceForEVM(contractAddress, chainId);

      if (balanceEth < requiredEth) {
        toast.error(`Insufficient ${chainId === 42220 ? "CELO" : "ETH"} : You need at least ${requiredEth}${chainId === 42220 ? " CELO" : " ETH"}`);
        return;
      }

      const { success, code, result } = await mintNFT(contractAddress, chainId);

      if (!success) {
        toast.error(code);
        return;
      }
      if (success) {
        // Handle successful minting
      }
    }
  }
} catch (err) {
  console.warn("Failed to connect..", err);
  toast.error("An error occurred while processing your request");
} finally {
  stopLoading();  // End the loading indicator
}

```

---
