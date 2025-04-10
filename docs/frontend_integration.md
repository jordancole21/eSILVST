# eSILVST Wallet PWA Front-End Integration with Next.js

## Overview
This document outlines how a potential front end for the eSILVST Wallet Progressive Web App (PWA), built with Next.js, can connect to the existing eSILVST smart contracts (SilverBackToken and SilverTrading) deployed on the Ethereum blockchain, specifically on the Sepolia testnet with potential for Mainnet deployment. The goal is to create a user-friendly interface that enables users to interact with the silver-backed token system through a modern, mobile-first web application.

## Technology Stack
The front-end stack is designed to leverage Next.js for its server-side rendering and static site generation capabilities, ensuring optimal performance and SEO for a PWA. Key technologies include:
- **Next.js**: Framework for building the React-based front end, providing routing, API routes, and PWA support via plugins like `next-pwa`.
- **React**: For creating reusable UI components corresponding to the app's pages (e.g., Dashboard, Trade).
- **Ethers.js**: Library for interacting with Ethereum blockchain, connecting to smart contracts, and handling wallet interactions.
- **Web3-React or WalletConnect**: Optional libraries for streamlined wallet provider integration (e.g., MetaMask).
- **Tailwind CSS (Optional)**: For styling the UI based on the "Silver Vault Modernism" theme outlined in the design brief, ensuring responsive, mobile-first design.

This stack is compatible with the existing Solidity smart contracts built using Foundry, as detailed in the project documentation.

## Setup and Configuration
To set up a Next.js project for eSILVST Wallet PWA, follow these initial steps:
1. **Create Next.js Project**: Initialize a new Next.js app using `npx create-next-app@latest eSILVST-wallet --typescript --use-npm`, selecting TypeScript for type safety.
2. **Install Dependencies**: Add necessary packages for blockchain interaction and PWA support:
   ```
   npm install ethers @web3-react/core @web3-react/injected-connector next-pwa
   ```
3. **Configure Environment Variables**: Create a `.env.local` file in the project root to store sensitive data like contract addresses and RPC URLs (e.g., Sepolia RPC URL from Alchemy or Infura). Example:
   ```
   NEXT_PUBLIC_SILVER_BACK_TOKEN_ADDRESS=<Sepolia Token Address>
   NEXT_PUBLIC_SILVER_TRADING_ADDRESS=<Sepolia Trading Address>
   NEXT_PUBLIC_RPC_URL=https://eth-sepolia.g.alchemy.com/v2/<your_api_key>
   ```
4. **Enable PWA Features**: Configure `next-pwa` in `next.config.js` to enable offline capabilities and home screen installation prompts for mobile users.
   ```
   const withPWA = require('next-pwa')({ dest: 'public' });
   module.exports = withPWA({ /* other Next.js config */ });
   ```
5. **Project Structure**: Organize the Next.js project with pages (e.g., `/pages/dashboard.tsx`, `/pages/trade.tsx`) mapping to the site structure, components for reusable UI elements, and a utility folder for blockchain interaction logic.

## Connecting to Smart Contracts
To interact with the eSILVST smart contracts, use Ethers.js to establish connections to the deployed instances of SilverBackToken and SilverTrading:
1. **Fetch Contract ABIs**: Obtain the ABI (Application Binary Interface) for each contract from the compiled artifacts in the Foundry project (e.g., `out/SilverBackToken.sol/SilverBackToken.json`) or via Etherscan if verified. Store these as JSON files in the front-end project (e.g., `/abis/` folder).
2. **Set Up Provider**: Detect the user's wallet provider (e.g., MetaMask) or use a fallback RPC provider for read-only operations. Example in a utility file `/utils/blockchain.ts`:
   ```
   import { ethers } from 'ethers';

   export const getProvider = () => {
     if (typeof window !== 'undefined' && window.ethereum) {
       return new ethers.BrowserProvider(window.ethereum);
     }
     return new ethers.JsonRpcProvider(process.env.NEXT_PUBLIC_RPC_URL);
   };
   ```
3. **Initialize Contract Instances**: Create contract objects for interaction using the provider, ABI, and contract addresses from environment variables:
   ```
   import SilverBackTokenABI from '../abis/SilverBackToken.json';
   import SilverTradingABI from '../abis/SilverTrading.json';

   export const getContracts = async () => {
     const provider = getProvider();
     const signer = await provider.getSigner().catch(() => provider); // Fallback to read-only if no wallet
     const tokenContract = new ethers.Contract(
       process.env.NEXT_PUBLIC_SILVER_BACK_TOKEN_ADDRESS,
       SilverBackTokenABI,
       signer
     );
     const tradingContract = new ethers.Contract(
       process.env.NEXT_PUBLIC_SILVER_TRADING_ADDRESS,
       SilverTradingABI,
       signer
     );
     return { tokenContract, tradingContract };
   };
   ```

## Wallet Integration
Integrate with wallet providers to enable user authentication and transaction signing:
1. **Detect Wallet Connection**: Use Ethers.js or `@web3-react` to check for an injected provider like MetaMask. Prompt user to connect if none detected:
   ```
   import { useWeb3React } from '@web3-react/core';
   import { InjectedConnector } from '@web3-react/injected-connector';

   const injected = new InjectedConnector({ supportedChainIds: [11155111, 1] }); // Sepolia and Mainnet

   export const ConnectWalletButton = () => {
     const { activate, active } = useWeb3React();
     return (
       <button onClick={() => activate(injected)}>
         {active ? 'Connected' : 'Connect Wallet'}
       </button>
     );
   };
   ```
2. **Handle Network Switching**: Detect current network and prompt user to switch if incorrect (e.g., not on Sepolia for testnet). Use `window.ethereum.request` to switch chains:
   ```
   const switchNetwork = async (chainId) => {
     await window.ethereum.request({
       method: 'wallet_switchEthereumChain',
       params: [{ chainId: `0x${chainId.toString(16)}` }],
     });
   };
   ```

## Implementing Core Functionalities
Map the app's core functionalities to smart contract interactions using Next.js components and Ethers.js calls:

### Balance Display (Dashboard)
- **Fetch eSILVST Balance**: Call `balanceOf` on SilverBackToken for the user's address:
  ```
  const balance = await tokenContract.balanceOf(userAddress);
  const formattedBalance = ethers.formatEther(balance); // Convert from wei to tokens
  ```
- **Fetch ETH Balance**: Use provider to get ETH balance:
  ```
  const ethBalance = await provider.getBalance(userAddress);
  const formattedEthBalance = ethers.formatEther(ethBalance);
  ```
- **Display**: Update state in a React component (e.g., `Dashboard.tsx`) to show balances dynamically.

### Trading (Buy/Sell on Trade Page)
- **Buy eSILVST with ETH**: Call `buyTokensWithETH` on SilverTrading, specifying ETH value:
  ```
  const tx = await tradingContract.buyTokensWithETH({ value: ethers.parseEther(ethAmount) });
  await tx.wait(); // Wait for transaction confirmation
  ```
- **Sell eSILVST for ETH**: First approve token spend, then call `sellTokensForETH`:
  ```
  const amount = ethers.parseEther(tokenAmount);
  await tokenContract.approve(tradingContract.target, amount);
  const tx = await tradingContract.sellTokensForETH(amount);
  await tx.wait();
  ```
- **Handle Transaction States**: Use React state to show "Pending", "Success", or "Error" notifications, with gas fee estimates before confirmation using `provider.estimateGas`.

### Price Updates (Market Page)
- **Fetch Contract Prices**: Read `silverPriceUSD` and `ethPriceUSD` from SilverTrading:
  ```
  const silverPrice = await tradingContract.silverPriceUSD();
  const ethPrice = await tradingContract.ethPriceUSD();
  const formattedSilverPrice = Number(silverPrice) / 1e8; // Adjust for 8 decimals
  const formattedEthPrice = Number(ethPrice) / 1e8;
  ```
- **Optional External API**: For real-time data beyond contract values, integrate an API like CoinGecko via Next.js API routes to fetch current silver and ETH prices securely.

### Transaction History (History Page)
- **Fetch Events**: Use Ethers.js to query contract events (`TokensToETH`, `ETHToTokens`) for the user's address:
  ```
  const filter = tradingContract.filters.TokensToETH(userAddress);
  const events = await tradingContract.queryFilter(filter, fromBlock, toBlock);
  ```
- **Alternative**: Use Etherscan API for transaction history if event logs are insufficient, fetching via a Next.js API route to hide API keys.

## User Interface Mapping
The Next.js front end maps directly to the documented page structures and user flows:
- **Pages**: Create a Next.js page for each app section (e.g., `/pages/dashboard.tsx`, `/pages/trade.tsx`) reflecting the layouts in `docs/page_structures/`.
- **Components**: Build reusable components (e.g., `BalanceCard`, `TradeForm`) for UI blocks, updating dynamically based on blockchain state fetched via Ethers.js.
- **Routing**: Use Next.js dynamic routes and `useRouter` for navigation, ensuring protected routes (e.g., Dashboard) require wallet connection via a higher-order component or context.
- **State Management**: Use React Context or Redux to manage global state (e.g., user wallet, balances), updating via contract calls to keep UI in sync with blockchain.

## Security Considerations
Ensure the front end adheres to best practices for security:
- **HTTPS**: Deploy over HTTPS to secure API and wallet interactions.
- **No Private Keys**: Never store private keys or seed phrases client-side; rely on wallet providers for signing.
- **Network Alerts**: Prompt users before transactions if on the wrong network (e.g., "Switch to Sepolia for test transactions?"), preventing costly errors.
- **Input Validation**: Sanitize user inputs (e.g., trade amounts) to prevent invalid contract calls.
- **Environment Variables**: Store contract addresses and RPC URLs in `.env.local`, not in client-side code, using Next.js `NEXT_PUBLIC_` prefix only for necessary public data.

## Deployment
Deploy the Next.js app to a static hosting service like Vercel for simplicity and scalability:
1. **Push to GitHub**: Store the front-end code in a repository, ensuring `.env.local` is ignored via `.gitignore`.
2. **Connect to Vercel**: Link the repository to Vercel, setting environment variables in the Vercel dashboard for contract addresses and RPC URLs.
3. **Custom Domain (Optional)**: Configure a custom domain for branding (e.g., `esilvst-wallet.app`).
4. **PWA Manifest**: Ensure `manifest.json` and service worker are deployed for offline functionality and home screen installation.

## Summary
This guide details how a Next.js front end for the eSILVST Wallet PWA can connect to the existing eSILVST smart contracts, covering setup, blockchain interaction with Ethers.js, wallet integration, core functionality implementation, UI mapping, security, and deployment. By following this approach, the front end will provide a seamless, mobile-first interface for users to manage silver-backed tokens, directly interfacing with the deployed SilverBackToken and SilverTrading contracts on Sepolia or Mainnet.
