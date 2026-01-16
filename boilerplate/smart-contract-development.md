# Why Use Both Hardhat + Foundry in One Project

## Benefits

1. **Forge for testing** — faster execution, better error traces, built-in fuzzing, Solidity-native tests
2. **Hardhat for scripts** — JS/TS flexibility, rich plugin ecosystem, complex deployment logic
3. **Best tooling from each** — no compromise, pick the right tool for each task
4. **Same contracts** — both read from the same source folder, no duplication

## Tradeoffs

- Two configs to maintain (`foundry.toml` + `hardhat.config.js`)
- Two sets of dependencies (Rust + Node)
- Slightly more setup complexity

## Recommendation

Worth it for serious projects. For simple stuff, just pick one.

# Commands

## Hardhat

## Foundry

## Hardhat + Foundry

# Source
- https://hardhat.org/docs/getting-started
- https://getfoundry.sh/config/hardhat
