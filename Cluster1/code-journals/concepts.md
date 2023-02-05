# Program Derived Address (PDA)

# Solana Programs

Solana Programs, often referred to as "smart contracts" on other blockchains, are the executable code that interprets the instructions sent inside of each transaction on the blockchain. They can be deployed directly into the core of the network as Native Programs, or published by anyone as On Chain Programs. Programs are the core building blocks of the network and handle everything from sending tokens between wallets, to accepting votes of a DAOs, to tracking ownership of NFTs.

Both types of programs run on top of the Sealevel runtime, which is Solana's parallel processing model that helps to enable the high transactions speeds of the blockchain.

## Key points

1. Programs are essentially special type of Accounts that are marked as "executable".
2. Programs can own other Accounts.
3. Programs can only change the data or debit accounts they own.
4. Any program can read or credit another account.
5. Programs are considered stateless since the primary data stored in a program account is the compiled SBF code.
6. Programs can be upgraded by their owner.

## Types of programs

The Solana blockchain has two types of programs:

1. Native programs
2. On chain programs

### On chain programs

These user written programs, often referred to as "smart contracts" on other blockchains, are deployed directly to the blockchain for anyone to interact with and execute. Hence the name "on chain"!

In effect, "on chain programs" are any program that is not baked directly into the Solana cluster's core code (like the native programs discussed below).

And even though Solana Labs maintains a small subset of these on chain programs (collectively known as the Solana Program Library), anyone can create or publish one. On chain programs can also be updated directly on the blockchain by the respective program's Account owner.

### Native programs

Native programs are programs that are built directly into the core of the Solana blockchain.

Similar to other "on chain" programs in Solana, native programs can be called by any other program/user. However, they can only be upgraded as part of the core blockchain and cluster updates. These native program upgrades are controlled via the releases to the different clusters.

Examples of native programs include:

- System Program: Create new accounts, transfer tokens, and more
- BPF Loader Program: Deploys, upgrades, and executes programs on chain
- Vote program: Create and manage accounts that track validator voting state and rewards.

## Executable

When a Solana program is deployed onto the network, it is marked as "executable" by the BPF Loader Program. This allows the Solana runtime to efficiently and properly execute the compiled program code.

## Upgradable

Unlike other blockchains, Solana programs can be upgraded after they are deployed to the network.

Native programs can only be upgraded as part of cluster updates when new software releases are made.

On chain programs can be upgraded by the account that is marked as the "Upgrade Authority", which is usually the Solana account/address that deployed the program to begin with.

# Cross-Program Invocations (CPI)

The Solana runtime allows programs to call each other via a mechanism called cross-program invocation.
Calling between programs is achieved by one program invoking an instruction of the other.
The invoking program is halted until the invoked program finishes processing the instruction.
For example, a client could create a transaction that modifies two accounts, each owned by separate on-chain programs

Solana Program needs to have a single entry point. This means a Solana Program that depends on other Solana Programs needs a way to disable the other entry points. This is done using [features] in Cargo.

Add the no-entrypoint feature to `Cargo.toml` of the spl-token crate.

    [features]
    no-entrypoint = []

Then under [dependencies] we need to use the import

    [dependencies]
    solana-program = "1.9.4"
    thiserror = "1.0.24"
    spl-token = {version = "3.2.0", features = ["no-entrypoint"]}

Lastly, add this annotation over the entrypoint! macro that you wish to disable on import (the child program):

    #[cfg(not(feature = "no-entrypoint"))]
    entrypoint!(process_instruction);
