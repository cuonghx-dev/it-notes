# hardhat.config.ts
- Option 1:
```ts
  solidity: {
    ...
    npmFilesToBuild: ["@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol"],
    ...
  },
```
- Option 2:

```ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import "hardhat-dependency-compiler";

const config: HardhatUserConfig = {
  solidity: "0.8.28",
  networks: {
    hardhat: {
      allowUnlimitedContractSize: true, // Required for Safe contracts
    },
  },
  dependencyCompiler: {
    paths: [
      "@safe-global/safe-contracts/contracts/proxies/SafeProxyFactory.sol",
      "@safe-global/safe-contracts/contracts/Safe.sol",
    ],
  },
};

export default config;
```
