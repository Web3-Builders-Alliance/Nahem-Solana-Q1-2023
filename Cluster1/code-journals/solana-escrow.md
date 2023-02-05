# entrypoint.rs

Entrypoints are the only way to call programs; all calls go through the function declared as the entrypoint.

The required crates are brought into scope using the keyword `use`. Basically, it's importing from the `solana_program` library some structs and macros that we'll use next.

    use solana_program::{
        account_info::AccountInfo, entrypoint, entrypoint::ProgramResult, pubkey::Pubkey,
    };

Solana Programs (known as smart contracts on other blockchains) are the executable code that interprets the instructions sent inside of each transaction on the blockchain.

The following line is bringing into scope the struct `Processor` from the `processor.rs` file located in the `src` folder.

    use crate::processor::Processor;

Solana Programs need to have a single entry point. This means that a Solana Program that depends on other Solana Programs needs a way to disable the other entry points. This is done using the macro `entrypoint!` for the `process_instruction` function.

    entrypoint!(process_instruction);

Then we define a function called `process_instruction` that will reveive the `program_id` public key, the `accounts` vector and the `instruction_data` vector, all passed as reference, and will return a struct of type `ProgramResult`.

    fn process_instruction(
        program_id: &Pubkey,
        accounts: &[AccountInfo],
        instruction_data: &[u8],
    ) -> ProgramResult {
        Processor::process(program_id, accounts, instruction_data)
    }

# intruction.rs

We define here the enum `EscrowInstruction` that will contain all the possible instructions our program will be able to execute.
The keyword `pub` means public and is used so we can access it from other files.
In this case we have two possible options:

1. InitEscrow starts the trade by creating and populating an escrow account and transferring ownership of the given temp token account to the PDA. It takes `amount` as a parameter to set the amount party A expects to receive of token Y.

2. Exchange is used to accept a trade that party A already initialized. It alse takes `amount` as a parameter and represents the amount the taker expects to be paid in the other token, as a u64 because that's the max possible supply of a token.

`pub enum EscrowInstruction {
    InitEscrow {
        amount: u64,
    },
    Exchange {
        amount: u64,
    },
}`

We should also define the accounts that each enum will be interacting with as we'll need to iterate over them in the `processor.rs` file later. In the brakets we define whether the account needs to be `signer`, `writable`, or ` ` (read-only).

Accounts expected for `InitEscrow`:

0. `[signer]` The account of the person initializing the escrow
1. `[writable]` Temporary token account that should be created prior to this instruction and owned by the initializer
2. `[]` The initializer's token account for the token they will receive should the trade go through
3. `[writable]` The escrow account, it will hold all necessary info about the trade.
4. `[]` The rent sysvar
5. `[]` The token program

Accounts expected for `Exchange`:

0. `[signer]` The account of the person taking the trade
1. `[writable]` The taker's token account for the token they send
2. `[writable]` The taker's token account for the token they will receive should the trade go through
3. `[writable]` The PDA's temp token account to get tokens from and eventually close
4. `[writable]` The initializer's main account to send their rent fees to
5. `[writable]` The initializer's token account that will receive tokens
6. `[writable]` The escrow account holding the escrow info
7. `[]` The token program
8. `[]` The PDA account

We use the keyword `impl` to implement two functions for the `EscrowInstruction` enum.

The first function, `unpack()`, unpacks a byte buffer into a [EscrowInstruction] enum. It needs to be public as it will be called from outside this file and it returns the `amount` wrapped in a `Result` struct, or a `ProgramError` if there's an error in the execution.

The `unpack()` function splits the `u8` array into two pieces, the first byte is going to represent the `tag` or the instruction code which we'll match with some values:

- `0` for `InitEscrow`.
- `1` for `Exchange`.
- `_` means anything else, should we get any other number we'll return a `InvalidInstruction` error and stop execution.

After we `match`, we use the double arm arrow `=>` to return a specific value on each case. `Self` means it will return the same estructure we are implementing this function to, `EscrowInstruction` and we'll call the next function `unpack_amount` passing the rest of the bytes.

The second part of the decoded `u8` array represents the `amount` and will need to be passed to the `unpack_amount()` function to decode the actual value as `u64`.

This function is private because it'll always be called from within this file and thus doesn't need to be exposed. It will receive the remaining `u8` bytes as an array and will return an `u64` value wrapped into a `Result`, or a `ProgramError`.

Actually, what this function is doing under the hood is trying to `get` the remaining bytes until the 8th one, if the array is not empty, it'll then `slice` it and `map` it to little indian bytes and return the `amount` wrapped into a `Result`. if something goes wrong, it'll return an `InvalidInstruction` error instead.

`impl EscrowInstruction {
pub fn unpack(input: &[u8]) -> Result<Self, ProgramError> {
let (tag, rest) = input.split_first().ok_or(InvalidInstruction)?;

        Ok(match tag {
            0 => Self::InitEscrow {
                amount: Self::unpack_amount(rest)?,
            },
            1 => Self::Exchange {
                amount: Self::unpack_amount(rest)?,
            },
            _ => return Err(InvalidInstruction.into()),
        })
    }
    fn unpack_amount(input: &[u8]) -> Result<u64, ProgramError> {
        let amount = input
            .get(..8)
            .and_then(|slice| slice.try_into().ok())
            .map(u64::from_le_bytes)
            .ok_or(InvalidInstruction)?;
        Ok(amount)
    }

}
`

# processor.rs

In the `processor.rs` file is where the logic of our program lays.

We first define a public struct `Processor` and then implement the main function called `process`. We need to pass as parameters the `program_id` of type `PubKey`, the `accounts` is an array of `AccountInfo`, and the `instruction_data` which is an array of `u8`.

First, we unpack the instruction data to get the type of `instruction`. Then we match `instruction` with the two options we expect: `InitEscrow {amount}` or `Exchange {amount}` and then map them to their respective function passing the `accounts`, the `amount` and the `program_id`.

The `process` function returns a ProgramResult which we previously brought into scope with the keyword `use`

`
pub struct Processor;
impl Processor {
pub fn process(
program_id: &Pubkey,
accounts: &[AccountInfo],
instruction_data: &[u8],
) -> ProgramResult {
let instruction = EscrowInstruction::unpack(instruction_data)?;

        match instruction {
            EscrowInstruction::InitEscrow { amount } => {
                msg!("Instruction: InitEscrow");
                Self::process_init_escrow(accounts, amount, program_id)
            }
            EscrowInstruction::Exchange { amount } => {
                msg!("Instruction: Exchange");
                Self::process_exchange(accounts, amount, program_id)
            }
        }
    }

`

Once the two instructions are mapped to their respective functions, we need to define them.

In the `process_init_escrow` function we iterate over the accounts and assign them to their variables.

First, we check if the `initializer` is the `signer`, otherwise we want to return a `MissingRequiredSignature` error.

Then for the `token_to_receive_account`, we dereference it with the `*` sign and then validate if the `owner` and the `spl_token::id()` are the same or we return a `IncorrectProgramId` error.

We validate if the escrow account is rent exempt or return another error, `NotRentExempt`.

To get the state of the program we need to load `escrow_info` and then unpack it with the `unpack_unchecked` trait to be able to access `state` key/values. As we're going to be modifying those values and then saving them to state again, we want this variable to be mutable. For that we use the keyword `mut`.

Also we need to make sure that the escrow is not `initialized` as we can only `initialize` it once, otherwise we return a `AccountAlreadyInitialized` error.

Now we want to change `escrow_info` to include the values after initialization which will be packed into state later.

`TODO:` continue where I left it... `owner_change_ix`

`fn process_init_escrow(
accounts: &[AccountInfo],
amount: u64,
program_id: &Pubkey,
) -> ProgramResult {
let account_info_iter = &mut accounts.iter();
let initializer = next_account_info(account_info_iter)?;

        if !initializer.is_signer {
            return Err(ProgramError::MissingRequiredSignature);
        }

        let temp_token_account = next_account_info(account_info_iter)?;

        let token_to_receive_account = next_account_info(account_info_iter)?;
        if *token_to_receive_account.owner != spl_token::id() {
            return Err(ProgramError::IncorrectProgramId);
        }

        let escrow_account = next_account_info(account_info_iter)?;
        let rent = &Rent::from_account_info(next_account_info(account_info_iter)?)?;

        if !rent.is_exempt(escrow_account.lamports(), escrow_account.data_len()) {
            return Err(EscrowError::NotRentExempt.into());
        }

        let mut escrow_info = Escrow::unpack_unchecked(&escrow_account.try_borrow_data()?)?;
        if escrow_info.is_initialized() {
            return Err(ProgramError::AccountAlreadyInitialized);
        }

        escrow_info.is_initialized = true;
        escrow_info.initializer_pubkey = *initializer.key;
        escrow_info.temp_token_account_pubkey = *temp_token_account.key;
        escrow_info.initializer_token_to_receive_account_pubkey = *token_to_receive_account.key;
        escrow_info.expected_amount = amount;

        Escrow::pack(escrow_info, &mut escrow_account.try_borrow_mut_data()?)?;
        let (pda, _nonce) = Pubkey::find_program_address(&[b"escrow"], program_id);

        let token_program = next_account_info(account_info_iter)?;
        let owner_change_ix = spl_token::instruction::set_authority(
            token_program.key,
            temp_token_account.key,
            Some(&pda),
            spl_token::instruction::AuthorityType::AccountOwner,
            initializer.key,
            &[&initializer.key],
        )?;

        msg!("Calling the token program to transfer token account ownership...");
        invoke(
            &owner_change_ix,
            &[
                temp_token_account.clone(),
                initializer.clone(),
                token_program.clone(),
            ],
        )?;

        Ok(())
    }`
