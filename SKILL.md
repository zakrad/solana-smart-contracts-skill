---
name: solana-smart-contracts
description: "Complete framework for developing Solana smart contracts using Anchor and native Rust. Covers installation, Rust fundamentals, account model, PDAs, storage, SPL tokens, Token-2022 extensions, CPIs, security patterns, testing, and sBPF internals. Use when building, debugging, or auditing Solana programs."
license: MIT
metadata:
  author: zakrad
  version: "1.0.0"
  organization: zakrad
  date: May 2025
  abstract: Comprehensive Solana smart contract development guide based on the RareSkills 60-Day Solana curriculum (https://rareskills.io/solana-tutorial). Covers 27 topic areas across 65 modules — installation, Rust fundamentals for Solidity developers, Solana account model, PDAs and storage, signers and access control, SOL transfers, sysvars, events, errors, CPIs, SPL tokens, Token-2022 extensions, Metaplex metadata, instruction introspection, Ed25519 signature verification, native Rust programs, Borsh serialization, security checks, testing with LiteSVM, Anchor constraints, sBPF internals, and Switchboard oracles. Includes complete code examples, pitfall tables, and EVM-to-Solana comparison.
---

# Solana Smart Contract Development

Full-stack framework for building, testing, and deploying Solana programs using Anchor and native Rust. Generated from the [RareSkills 60-Day Solana Book](https://rareskills.io/solana-tutorial) (65 modules) — the most comprehensive free Solana smart contract curriculum available.

---

## When to Use

- Setting up a new Solana/Anchor project
- Writing or debugging Anchor programs
- Understanding the account model and PDAs
- Working with SPL tokens or Token-2022
- Implementing CPIs between programs
- Writing native Rust programs (no Anchor)
- Security-reviewing Solana code
- Optimizing compute unit usage

---

## 1. Installation & Project Setup

### Tool Stack
```bash
# Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Solana CLI (v1.18+)
curl --proto '=https' --tlsv1.2 -sSfL https://solana-install.solana.workers.dev | bash

# Anchor via AVM
cargo install --git https://github.com/solana-foundation/anchor avm --force
avm install latest && avm use latest
# Recommended: Anchor 0.29+, Solana 1.18+, Rustc 1.77.0-nightly

# Create project
anchor init my_project
cd my_project
anchor keys sync   # always run after init or clone
```

### Three-Shell Workflow
- **Shell 1**: code / `anchor build` / `anchor test`
- **Shell 2**: `solana-test-validator` (local chain)
- **Shell 3**: `solana logs` (real-time program logs)

### Key CLI Commands
```bash
solana config set --url localhost
anchor keys sync                              # sync declare_id! with keypair
anchor build
anchor test --skip-local-validator            # reuse existing validator
anchor test --skip-local-validator --skip-deploy  # test already-deployed program
solana-test-validator --reset                 # reset state between runs
solana airdrop 100 <ADDRESS>
solana program close <ADDR> --bypass-warning  # permanent — address can never be reused
```

### Canonical Program Structure
```rust
use anchor_lang::prelude::*;

declare_id!("PROGRAM_ID_HERE");

#[program]
pub mod my_program {
    use super::*;

    pub fn my_instruction(ctx: Context<MyAccounts>, param: u64) -> Result<()> {
        require!(param > 0, MyError::InvalidParam);
        let signer = ctx.accounts.signer.key();
        msg!("Called by: {:?}, param: {}", signer, param);
        ctx.accounts.my_storage.value = param;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct MyAccounts<'info> {
    #[account(init, payer = signer, space = 8 + 8, seeds = [], bump)]
    pub my_storage: Account<'info, MyStorage>,
    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[account]
pub struct MyStorage {
    pub value: u64,
}

#[error_code]
pub enum MyError {
    #[msg("Parameter must be positive")]
    InvalidParam,
}
```

### Troubleshooting Matrix
| Error | Fix |
|---|---|
| Keys not synced | `anchor keys sync` |
| RPC port 8899 in use | `--skip-local-validator` |
| Program doesn't exist | start validator |
| Insufficient funds | `solana airdrop 100 <ADDR>` |
| `build_hasher_simple_hash_one` unstable | `cargo update -p ahash@0.8.7 --precise 0.8.6` |
| Blockstore error (macOS) | `rm -rf ~/.cache/solana/*` |

---

## 2. Rust Fundamentals for Solana Developers

### Key Differences from Solidity
| Concept | Solidity | Rust/Solana |
|---|---|---|
| Default integer | `uint256` | `u64` |
| Overflow | Silent (or SafeMath) | `checked_add` / `overflow-checks = true` |
| Error handling | `try/catch` | `Result<()>` + `?` operator |
| Null | `address(0)` | `Option<T>` |
| Classes | `contract` | `struct` + `impl` |
| Interfaces | `interface` | `trait` + `impl` |
| Constructors | `constructor()` | Explicit init function (no constructors on Solana) |
| Immutable vars | `immutable` | None (programs are upgradeable) |

### Integer Types and Overflow
```rust
// Safe arithmetic (ALWAYS prefer)
let result = a.checked_add(b).unwrap();
// Also: checked_sub, checked_mul, checked_pow, checked_div

// Enable globally in Cargo.toml:
[profile.release]
overflow-checks = true

// Default: u64 (matches eBPF native register size)
// Use smallest type for hot paths: u8 < u16 < u32 < u64 (fewer CUs)
```

### Control Flow
```rust
// If-else (no parentheses required)
if age >= 18 { msg!("adult"); } else { msg!("minor"); }
let result = if age >= 18 { "adult" } else { "minor" };  // ternary

// Match (exhaustive pattern matching)
match outcome {
    0 => msg!("NO"),
    1 => msg!("YES"),
    2 => msg!("INVALID"),
    _ => return err!(MyError::UnknownOutcome),
}

// For loops
for i in 0..10 { }             // 0..9
for i in (0..10).step_by(2) { } // 0, 2, 4, 6, 8
```

### Ownership and Borrowing
```rust
// Copy types (integers, bool): auto-copied
let x: u64 = 5;
let y = x;  // x still valid

// Non-copy types (String, Vec, structs): ownership moves
let s1 = String::from("abc");
let s2 = s1;  // s1 is INVALID now

// Borrow (&): read-only reference, original stays valid
let s1 = String::from("abc");
let s2 = &s1;  // s1 still valid
// Clone for independent copy:
let s2 = s1.clone();

// Mutable borrow (&mut): one at a time
let mut counter = 0;
counter += 1;
```

### Result and the `?` Operator
```rust
// Functions must return Result<()>
pub fn my_fn(ctx: Context<MyCtx>) -> Result<()> {
    let data: MyStruct = some_operation()?;  // propagates Err, unwraps Ok
    Ok(())
}

// Equivalent without ?:
match some_operation() {
    Ok(v) => v,
    Err(e) => return Err(e),
}
```

### Structs and Traits
```rust
struct Person { name: String, age: u8 }

impl Person {
    fn new(name: String, age: u8) -> Self { Person { name, age } }
    fn can_drink(&self) -> bool { self.age >= 21 }
}

trait Describable {
    fn describe(&self) -> String;
}
impl Describable for Person {
    fn describe(&self) -> String { format!("{} age {}", self.name, self.age) }
}
```

### Constants and Type Casting
```rust
// Constants OUTSIDE #[program] block
const TOKENS_PER_SOL: u64 = 100;
const SUPPLY_CAP: u64 = 1_000_000_000_000; // 1M with 6 decimals

// Explicit casts required
let len = v.len();  // usize
let total = len as u64 + another_u64;
```

### Vectors and HashMaps
```rust
// Vec — usable on-chain in accounts (fixed allocation)
let mut v: Vec<u64> = Vec::new();
v.push(10);
let first = v[0];

// HashMap — in-memory ONLY (not storable on-chain)
use std::collections::HashMap;
let mut map = HashMap::new();
map.insert("key".to_string(), 42u64);
```

### Macros — The Three Types
1. **Function-like** (`println!`, `msg!`, `require!`) — variadic arguments
2. **Derive macros** (`#[derive(Debug, BorshSerialize)]`) — attach `impl` blocks to structs
3. **Attribute-like** (`#[program]`, `#[derive(Accounts)]`) — can completely rewrite the annotated item

```rust
// Treat macros as black boxes — understand what they produce, not how
// #[program] → injects instruction routing
// #[derive(Accounts)] → adds validation impl
// #[account] → adds discriminator + serialization
```

---

## 3. Solana Account Model

### Solana vs Ethereum Mental Model
| EVM | Solana |
|---|---|
| `msg.sender` | Explicitly declared `Signer<'info>` |
| Contract state | Separate account owned by the program |
| `address(this).balance` | `ctx.accounts.acct.lamports()` |
| `block.timestamp` | `Clock::get()?.unix_timestamp` |
| `block.number` | `Clock::get()?.slot` |
| `require(cond)` | `require!(cond, ErrorCode)` |
| `emit Event(...)` | `emit!(MyEvent {...})` |
| Events queryable | Events real-time only; use tx history API |
| Delegatecall/proxies | Direct bytecode replacement (programs are mutable) |
| Chain ID | N/A (clusters, not IDs) |
| Block miner | Not accessible |
| try/catch | No — use `Result<()>` + `?` |

### Account Types
- **User wallets**: owned by System Program (`111...111`)
- **Programs**: owned by `BPFLoaderUpgradeable`
- **PDAs and program data accounts**: owned by the program itself

### Four Anchor Account Wrapper Types

**1. `Account<'info, T>`** — program-owned data (most common)
```rust
pub my_storage: Account<'info, MyStorage>,  // validates ownership
```

**2. `UncheckedAccount<'info>` / `AccountInfo<'info>`** — no ownership check
```rust
/// CHECK: reading balance only, not trusting data
pub some_external: UncheckedAccount<'info>,  // MUST have /// CHECK: comment
```

**3. `Signer<'info>`** — verifies the account signed the tx
```rust
pub signer: Signer<'info>,
// Access: ctx.accounts.signer.key(), ctx.accounts.signer.lamports()
```

**4. `Program<'info, T>`** — an executable program (used for CPIs)
```rust
pub system_program: Program<'info, System>,
pub token_program: Program<'info, Token>,
```

### Space Calculation (Critical)
```rust
// Always: size_of::<T>() + 8   (8 bytes = Anchor's type discriminator)
#[account(init, payer = signer, space = 8 + size_of::<MyStorage>())]
pub my_storage: Account<'info, MyStorage>,

// Typical sizes (Borsh serialization):
// bool    = 1 byte
// u8      = 1 byte
// u16     = 2 bytes
// u32     = 4 bytes
// u64     = 8 bytes
// u128    = 16 bytes
// Pubkey  = 32 bytes
// Vec<T>  = 4 bytes length + n * size_of::<T>()
// String  = 4 bytes length + n bytes
// Option<T> = 1 byte discriminant + size_of::<T>()
```

---

## 4. PDAs and Storage

### PDA (Program Derived Address) — The Core Pattern
```rust
// On-chain: deterministic address from seeds + program ID
// No private key exists — program signs for it via invoke_signed

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = signer,
        space = 8 + size_of::<MyStorage>(),
        seeds = [],        // empty = single global account
        bump              // stores the canonical bump
    )]
    pub my_storage: Account<'info, MyStorage>,
    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}

// TypeScript: derive address client-side
const seeds = [];
const [myStorage, _bump] = anchor.web3.PublicKey.findProgramAddressSync(
    seeds, program.programId
);
await program.methods.initialize().accounts({ myStorage }).rpc();
```

### Seeds as Mapping Keys (Solidity `mapping` equivalent)
```rust
// Single key mapping: seeds = [key_bytes]
#[derive(Accounts)]
#[instruction(user: Pubkey)]   // expose function args to seeds
pub struct Initialize<'info> {
    #[account(
        init, payer = signer,
        space = 8 + size_of::<UserData>(),
        seeds = [user.as_ref()],  // one account per user pubkey
        bump
    )]
    pub user_data: Account<'info, UserData>,
    // ...
}

// TypeScript:
const seeds = [userPubkey.toBuffer()];
const [userDataAddr] = anchor.web3.PublicKey.findProgramAddressSync(seeds, program.programId);

// Numeric key (u64) — must specify byte length
seeds = [&key.to_le_bytes().as_ref()]  // Rust: little-endian, 8 bytes
const seeds = [key.toArrayLike(Buffer, "le", 8)];  // TypeScript

// Nested mapping: multiple seeds
seeds = [key1.as_ref(), key2.as_ref()]  // path: [key1][key2]
```

### Reading and Writing Account Data
```rust
// Write (account must be #[account(mut)])
pub fn set(ctx: Context<Set>, new_x: u64) -> Result<()> {
    ctx.accounts.my_storage.x = new_x;  // direct field assignment
    Ok(())
}

// Mutable reference pattern (multiple writes)
let storage = &mut ctx.accounts.my_storage;
storage.x = new_x;
storage.y = new_y;

// Read (no mut needed)
pub fn print(ctx: Context<Print>) -> Result<()> {
    let x = ctx.accounts.my_storage.x;
    msg!("x = {}", x);
    Ok(())
}
```

### Account Rent and Storage Costs
```rust
// Rent-exempt formula: (128 + data_bytes) × 3,480 lamports/byte/year × 2
// Empty account ≈ 890,880 lamports ≈ 0.00089 SOL
// Use CLI: `solana rent <bytes>`

// Resize with realloc
#[derive(Accounts)]
pub struct Resize<'info> {
    #[account(
        mut,
        realloc = 8 + NEW_SIZE,
        realloc::payer = signer,
        realloc::zero = false,  // false = preserve data; true = zero-fill
        seeds = [], bump
    )]
    pub storage: Account<'info, MyStorage>,
    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}
// Limits: init max 10,240 bytes; realloc max 10,240 per op; absolute max 10 MB
```

### Closing Accounts
```rust
#[derive(Accounts)]
pub struct Delete<'info> {
    #[account(mut, close = signer)]  // zeroes lamports, transfers to signer
    pub the_pda: Account<'info, ThePda>,
    #[account(mut)]
    pub signer: Signer<'info>,
}
// TypeScript: use .fetchNullable() after close — returns null for closed accounts
let account = await program.account.thePda.fetchNullable(thePda);
```

### Account Balance
```rust
// Read any account's SOL balance
let balance = ctx.accounts.acct.to_account_info().lamports();
msg!("Balance: {} lamports", balance);
```

### PDA vs Keypair Account
| | PDA | Keypair Account |
|---|---|---|
| Address source | programId + seeds | External keypair |
| Private key | None | Exists |
| Max initial size | 10,240 bytes | 10 MB |
| Client init | `findProgramAddressSync()` | `Keypair.generate()` |
| Preferred? | Yes (deterministic) | Rarely needed |

---

## 5. Signers, Authority, and Access Control

### Single Signer (msg.sender equivalent)
```rust
#[derive(Accounts)]
pub struct MyCtx<'info> {
    #[account(mut)]
    pub signer: Signer<'info>,  // validates this account signed the tx
}

pub fn my_fn(ctx: Context<MyCtx>) -> Result<()> {
    let caller = ctx.accounts.signer.key();
    msg!("Called by: {:?}", caller);
    Ok(())
}
```

### Multiple Signers
```rust
#[derive(Accounts)]
pub struct MultiSig<'info> {
    pub signer1: Signer<'info>,
    pub signer2: Signer<'info>,
}

// TypeScript — provider wallet auto-signs; add extras to .signers([])
await program.methods.myFn()
    .accounts({ signer1: provider.publicKey, signer2: extraKeypair.publicKey })
    .signers([extraKeypair])  // ONLY non-default signers here
    .rpc();
```

### OnlyOwner Pattern
```rust
const OWNER: &str = "8os8PKYmeVjU1mmwHZZNTEv5hpBXi5VvEKGzykduZAik";

#[access_control(check(&ctx))]
pub fn admin_fn(ctx: Context<AdminCtx>) -> Result<()> {
    // ...
    Ok(())
}

fn check(ctx: &Context<AdminCtx>) -> Result<()> {
    require_keys_eq!(
        ctx.accounts.signer.key(),
        OWNER.parse::<Pubkey>().unwrap(),
        MyError::NotOwner
    );
    Ok(())
}
```

### `has_one` Constraint (Authority Pattern)
```rust
#[account]
pub struct Player {
    points: u32,
    authority: Pubkey,  // stored authorized wallet
}

#[derive(Accounts)]
pub struct TransferPoints<'info> {
    #[account(
        mut,
        has_one = authority @ MyError::NotAuthority,  // auto-validates
        constraint = from.points >= amount @ MyError::InsufficientPoints
    )]
    from: Account<'info, Player>,
    #[account(mut)]
    to: Account<'info, Player>,
    authority: Signer<'info>,  // field name must match has_one value
}
```

### User-Specific PDA (per-user storage)
```rust
// Seed includes the user's pubkey — each user gets their own PDA
#[derive(Accounts)]
pub struct CreateUser<'info> {
    #[account(
        init, payer = signer,
        space = 8 + size_of::<UserAccount>(),
        seeds = [b"user", signer.key().as_ref()],
        bump
    )]
    pub user_account: Account<'info, UserAccount>,
    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

---

## 6. Transferring SOL

### User → Program (via CPI to System Program)
```rust
use anchor_lang::system_program;

pub fn receive_sol(ctx: Context<ReceiveSol>, amount: u64) -> Result<()> {
    let cpi_ctx = CpiContext::new(
        ctx.accounts.system_program.to_account_info(),
        system_program::Transfer {
            from: ctx.accounts.signer.to_account_info(),
            to: ctx.accounts.vault.to_account_info(),
        },
    );
    system_program::transfer(cpi_ctx, amount)?;
    Ok(())
}

#[derive(Accounts)]
pub struct ReceiveSol<'info> {
    #[account(mut)]
    pub vault: UncheckedAccount<'info>,  // or Account<'info, Vault>
    pub system_program: Program<'info, System>,
    #[account(mut)]
    pub signer: Signer<'info>,
}
```

### Program PDA → User (direct lamport manipulation)
```rust
// Anchor 0.29+
pub fn withdraw(ctx: Context<Withdraw>, amount: u64) -> Result<()> {
    ctx.accounts.vault.sub_lamports(amount)?;
    ctx.accounts.signer.add_lamports(amount)?;
    Ok(())
}

// Pre-0.29 syntax:
**ctx.accounts.vault.to_account_info().try_borrow_mut_lamports()? -= amount;
**ctx.accounts.signer.to_account_info().try_borrow_mut_lamports()? += amount;
```

### Split SOL to Multiple Recipients
```rust
pub fn split_sol<'a, 'b, 'c, 'info>(
    ctx: Context<'a, 'b, 'c, 'info, SplitSol<'info>>,
    amount: u64,
) -> Result<()> {
    let each = amount / ctx.remaining_accounts.len() as u64;
    for recipient in ctx.remaining_accounts {
        system_program::transfer(
            CpiContext::new(
                ctx.accounts.system_program.to_account_info(),
                system_program::Transfer {
                    from: ctx.accounts.signer.to_account_info(),
                    to: recipient.to_account_info(),
                },
            ),
            each,
        )?;
    }
    Ok(())
}

// TypeScript: pass extra accounts via .remainingAccounts([...])
await program.methods.splitSol(amount)
    .remainingAccounts([
        { pubkey: r1.publicKey, isWritable: true, isSigner: false },
        { pubkey: r2.publicKey, isWritable: true, isSigner: false },
    ])
    .rpc();
```

---

## 7. System Variables

### Clock / Timestamp
```rust
pub fn time_check(ctx: Context<TimeCheck>) -> Result<()> {
    let clock = Clock::get()?;
    let ts: i64 = clock.unix_timestamp;  // i64, not u64
    let slot: u64 = clock.slot;
    let epoch: u64 = clock.epoch;
    msg!("Timestamp: {}, Slot: {}", ts, slot);
    Ok(())
}

// Day of week via chrono crate
use chrono::*;
let dt = chrono::NaiveDateTime::from_timestamp_opt(ts, 0).unwrap();
msg!("Day: {}", dt.weekday());
```

### Sysvars
```rust
// Method 1 — get() preferred for: Clock, EpochSchedule, Rent
let rent = Rent::get()?;
let exemption = rent.minimum_balance(data_len);

// Method 2 — AccountInfo for: StakeHistory, Instructions
#[derive(Accounts)]
pub struct WithSysvar<'info> {
    /// CHECK: reading sysvar
    pub stake_history: AccountInfo<'info>,
}
// TypeScript: pass "SysvarStakeHistory1111111111111111111111111"

// Instructions sysvar (for instruction introspection)
use anchor_lang::solana_program::sysvar::instructions;
let ix = instructions::load_instruction_at_checked(0, instructions_sysvar_account)?;
```

### Key Slot Facts
- 1 slot = 400ms window
- 432,000 slots per epoch
- `block.number` equivalent → `slot` (not all slots have blocks)

---

## 8. Events and Logging

### Defining and Emitting Events
```rust
#[event]
pub struct MyEvent {
    pub value: u64,
    pub message: String,
}

pub fn my_fn(ctx: Context<MyCtx>) -> Result<()> {
    emit!(MyEvent { value: 42, message: "hello".to_string() });
    Ok(())
}
```

### Listening in TypeScript
```typescript
const listener = program.addEventListener('MyEvent', (event, slot) => {
    console.log(`slot ${slot} event value ${event.value}`);
});

await program.methods.myFn().rpc();
await new Promise(r => setTimeout(r, 5000)); // wait for propagation

program.removeEventListener(listener);  // ALWAYS remove to avoid leaks
```

### Querying Transaction History
```typescript
const conn = new web3.Connection(web3.clusterApiUrl("mainnet-beta"));
const sigs = await conn.getSignaturesForAddress(pubKey, { limit: 10 });
for (const sig of sigs.map(s => s.signature)) {
    const tx = await conn.getParsedTransaction(sig, { maxSupportedTransactionVersion: 0 });
    console.log(tx);
}
```

### Events vs Ethereum
| Feature | Solana | Ethereum |
|---|---|---|
| Indexed fields | No | Yes |
| Historical query | No (tx history API) | Yes (bloom filter) |
| Primary use | Frontend real-time | Audit trail |
| Under the hood | `sol_log_data` syscall | LOG opcode |

---

## 9. Errors and Validation

### Defining Errors
```rust
#[error_code]
pub enum MyError {
    #[msg("Value too small, must be >= 10")]
    ValueTooSmall,
    #[msg("Value too large, must be <= 100")]
    ValueTooLarge,
    #[msg("Not the authority")]
    Unauthorized,
}
```

### require! Macro
```rust
require!(value >= 10, MyError::ValueTooSmall);
require!(value <= 100, MyError::ValueTooLarge);
require_keys_eq!(ctx.accounts.signer.key(), expected_key, MyError::Unauthorized);

// Inline constraint in #[derive(Accounts)]
#[account(constraint = vault.locked == false @ MyError::VaultLocked)]
pub vault: Account<'info, Vault>,
```

### Key Difference from Solidity
- Solidity `require` → halts with `REVERT` opcode, discards all state + logs
- Solana `require!` → returns `Err(...)`, **already-executed** `msg!` calls DO appear in logs
- All program functions MUST return `Result<()`

---

## 10. Cross-Program Invocation (CPI)

### Anchor-to-Anchor CPI
```toml
# Cargo.toml of calling program
[dependencies]
bob = { path = "../bob", features = ["cpi"] }
```

```rust
use bob::cpi::accounts::BobAddOp;
use bob::program::Bob;
use bob::BobData;

pub fn call_bob(ctx: Context<CallBob>, a: u64, b: u64) -> Result<()> {
    let cpi_ctx = CpiContext::new(
        ctx.accounts.bob_program.to_account_info(),
        BobAddOp {
            bob_data_account: ctx.accounts.bob_data_account.to_account_info(),
        },
    );
    bob::cpi::add_and_store(cpi_ctx, a, b)?;
    Ok(())
}

#[derive(Accounts)]
pub struct CallBob<'info> {
    #[account(mut)]
    pub bob_data_account: Account<'info, BobData>,
    pub bob_program: Program<'info, Bob>,
}
```

### CPI with PDA Signing (`CpiContext::new_with_signer`)
```rust
// When the calling program's PDA must authorize the CPI
pub fn mint_tokens(ctx: Context<MintTokens>, amount: u64) -> Result<()> {
    let bump = ctx.bumps.mint_authority;
    let seeds: &[&[&[u8]]] = &[&[b"mint_authority", &[bump]]];

    let cpi_ctx = CpiContext::new_with_signer(
        ctx.accounts.token_program.to_account_info(),
        MintTo {
            mint: ctx.accounts.mint.to_account_info(),
            to: ctx.accounts.token_account.to_account_info(),
            authority: ctx.accounts.mint_authority.to_account_info(),
        },
        seeds,
    );
    token::mint_to(cpi_ctx, amount)?;
    Ok(())
}
```

### Native Rust CPI (`invoke` vs `invoke_signed`)
```rust
use solana_program::program::{invoke, invoke_signed};

// invoke — passes original tx signers through
invoke(&instruction, &[account1.clone(), account2.clone()])?;

// invoke_signed — PDA signs
let seeds: &[&[&[u8]]] = &[&[b"my-seed", &[bump]]];
invoke_signed(&instruction, &[account1.clone(), account2.clone()], seeds)?;

// Return data between programs
set_return_data(&my_bytes);              // in callee (max 1024 bytes)
let data = get_return_data();           // in caller
```

### CPI Rules
- All accounts referenced in the instruction MUST be in the `account_infos` slice
- Use `to_account_info()` to convert typed accounts to `AccountInfo`
- CPI errors propagate immediately — entire tx fails
- Verify `program_id` and `executable` flag before calling unknown programs
- CPI does NOT increase the transaction's compute unit budget

---

## 11. init_if_needed and Batched Transactions

### `init_if_needed` Feature
```toml
# Cargo.toml
anchor-lang = { version = "0.29.0", features = ["init-if-needed"] }
```

```rust
#[account(
    init_if_needed,
    payer = signer,
    space = 8 + size_of::<MyPDA>(),
    seeds = [], bump
)]
pub my_pda: Account<'info, MyPDA>,
```

**Security Warning**: An account is treated as uninitialized if: lamports == 0 OR owner == System Program. Any function that can drain lamports or change owner creates a reinitialization vulnerability.

### Batched Transactions (Atomic)
```typescript
// Use .transaction() not .rpc() for batching
const initTx = await program.methods.initialize().accounts({ pda }).transaction();
const setTx = await program.methods.set(5).accounts({ pda }).transaction();

const tx = new anchor.web3.Transaction().add(initTx).add(setTx);
await anchor.web3.sendAndConfirmTransaction(connection, tx, [wallet]);
```

### Client-Side init-if-needed
```typescript
const acct = await connection.getAccountInfo(pda);
const tx = new anchor.web3.Transaction();

if (!acct || acct.lamports === 0 || acct.owner.equals(SystemProgram.programId)) {
    tx.add(await program.methods.initialize().accounts({ pda }).transaction());
}
tx.add(await program.methods.set(5).accounts({ pda }).transaction());
await anchor.web3.sendAndConfirmTransaction(connection, tx, [wallet]);
```

**Limits**: 1,232 bytes hard limit per transaction — cannot be bypassed with more fees.

---

## 12. Compute Units

### CU Costs
| Operation | CU |
|---|---|
| Each sBPF instruction | 1 |
| `msg!` syscall | ~100 base |
| `sol_log_64_` / `sol_log_compute_units_` | 100 |
| Float operations | ~5× integer ops |
| Anchor instruction name log (auto) | ~100 |
| Default per-tx limit | 200,000 |
| Max with `set_compute_unit_limit` | 1,400,000 |
| Per-block limit | 48,000,000 |

### Optimization Tips
```rust
// Use smallest sufficient integer type
let mut v: Vec<u8> = Vec::new();   // ~459 CU
let mut v: Vec<u64> = Vec::new();  // ~618 CU

// Avoid floating point in hot paths — ~5× more CU than integers

// Split large operations across multiple transactions
// Store intermediate progress in on-chain accounts
```

### Transaction Fees (Current Model)
- Fee = **1 signature × 5,000 lamports** (compute usage does NOT affect fees yet)
- All txs with same signature count cost the same regardless of CU used
- Optimizing CU matters for block inclusion during congestion and future fee models

---

## 13. SPL Tokens

### Architecture
```
Token Program: TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
ATA Program:   ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL

Account Types:
  Mint Account — global token metadata (one per token type)
  ATA (Associated Token Account) — user balance (per user per mint)

ATA derivation: PDA(user_wallet + mint_address)
```

### Mint Account Fields
- `mint_authority` — who can mint new tokens (set to None to freeze supply)
- `freeze_authority` — who can freeze user accounts
- `decimals` — display decimals (amounts stored as raw integers)
- `supply` — total minted

### ATA Fields
- `mint` — which token
- `owner` / `authority` — authorized wallet (NOT the Solana program owner)
- `amount` — balance (in base units)
- `delegate` — single approved spender (unlike ERC-20 multi-approval)

### Token Transfer (Anchor CPI)
```rust
use anchor_spl::token::{self, Transfer, Token};

pub fn transfer_tokens(ctx: Context<TransferSpl>, amount: u64) -> Result<()> {
    let cpi_ctx = CpiContext::new(
        ctx.accounts.token_program.to_account_info(),
        Transfer {
            from: ctx.accounts.from_ata.to_account_info(),
            to: ctx.accounts.to_ata.to_account_info(),
            authority: ctx.accounts.authority.to_account_info(),
        },
    );
    token::transfer(cpi_ctx, amount)?;
    Ok(())
}

#[derive(Accounts)]
pub struct TransferSpl<'info> {
    #[account(mut)]
    pub from_ata: Account<'info, TokenAccount>,
    #[account(mut)]
    pub to_ata: Account<'info, TokenAccount>,
    pub authority: Signer<'info>,
    pub token_program: Program<'info, Token>,
}
```

### Mint-as-PDA Pattern (Programmatic Minting)
```rust
// Mint authority is a PDA the program controls
#[account(
    init, payer = signer,
    mint::decimals = 6,
    mint::authority = mint,      // self-referential: program signs for mint PDA
    seeds = [b"token_mint"],
    bump
)]
pub mint: Account<'info, Mint>,

// When minting: program signs with mint's own seeds
pub fn mint_tokens(ctx: Context<MintCtx>, amount: u64) -> Result<()> {
    let bump = ctx.bumps.mint;
    let seeds: &[&[&[u8]]] = &[&[b"token_mint", &[bump]]];
    let cpi_ctx = CpiContext::new_with_signer(..., seeds);
    token::mint_to(cpi_ctx, amount)?;
    Ok(())
}
```

### Token Amount Math
```rust
// ALWAYS account for decimals
// 100 tokens with 6 decimals = 100_000_000 base units
const TOKENS_PER_SOL: u64 = 100;
let amount_in_base_units = TOKENS_PER_SOL
    .checked_mul(10u64.pow(decimals as u32))
    .unwrap();

// Check supply cap before minting
let new_supply = mint.supply.checked_add(amount).unwrap();
require!(new_supply <= SUPPLY_CAP, MyError::SupplyExceeded);
```

### NFT Pattern
- `decimals = 0`
- `supply = 1`
- `mint_authority = None` (revoke after minting)
- Requires Metaplex metadata account for name/image

### SPL Token Rules
- Balances live in Token Accounts, NOT in the Mint account
- Recipient ATA must exist before transfer — create with `create_associated_token_account_idempotent`
- Only one delegate per ATA (unlike ERC-20)
- Closing an ATA requires zero balance

---

## 14. Metaplex Token Metadata

### Program ID
```
metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s
```

### Metadata PDA Seeds
```rust
// ["metadata", token_metadata_program_id, mint_address]
// Derive off-chain:
const [metadataAddr] = await anchor.web3.PublicKey.findProgramAddress(
    [Buffer.from("metadata"), TOKEN_METADATA_PROGRAM_ID.toBuffer(), mint.toBuffer()],
    TOKEN_METADATA_PROGRAM_ID
);
```

### Key Metadata Fields
- `update_authority` — can modify; `null` = permanently immutable
- `is_mutable` — once `false`, metadata is frozen forever
- `seller_fee_basis_points` — royalty suggestion 0–10000; NOT enforced on-chain
- `creators` — max 5 addresses, must sum to 100%; each must individually sign to verify
- `primary_sale_happened` — irreversible once true

### Token Standards
| Standard | supply | decimals |
|---|---|---|
| Fungible (2) | > 1 | > 0 |
| FungibleAsset (1) | = 1 | > 0 |
| NonFungible (0) | = 1 | = 0 |

### Add Metadata via CPI
```toml
# Cargo.toml
anchor-spl = { version = "0.31.0", features = ["token"] }
mpl-token-metadata = "5.1.0"
```

```rust
use mpl_token_metadata::instructions::CreateMetadataAccountV3Cpi;

let cpi = CreateMetadataAccountV3Cpi::new(
    &token_metadata_program_info,
    CreateMetadataAccountV3CpiAccounts {
        metadata, mint, mint_authority, payer, update_authority,
        system_program, rent,
    },
    CreateMetadataAccountV3InstructionArgs {
        data: DataV2 {
            name: "My Token".to_string(),
            symbol: "MTK".to_string(),
            uri: "https://arweave.net/...".to_string(),
            seller_fee_basis_points: 0,
            creators: None,
            collection: None,
            uses: None,
        },
        is_mutable: true,
        collection_details: None,
    },
);
cpi.invoke()?;  // or cpi.invoke_signed(signer_seeds)?
```

**Workflow**: Deploy to devnet → Create mint → Upload image to Irys/Arweave → Upload JSON → Call metadata instruction.

---

## 15. Token-2022 Extensions

### Key Differences from SPL Token
```
Program: TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb
Account wrappers: InterfaceAccount<'info, Mint>, Interface<'info, TokenInterface>
Encoding: TLV (Type-Length-Value) — extensions appended after base layout
```

### Critical Rule: Extensions Must Be Set Before Initialization
```rust
// Correct order:
// 1. Calculate mint size with extensions
let mint_size = ExtensionType::try_calculate_account_len::<PodMint>(
    &[ExtensionType::InterestBearingConfig]
)?;
// 2. Create account
system_program::create_account(CpiContext::new(...), lamports, mint_size as u64, &token_program.key())?;
// 3. Initialize EXTENSION first
interest_bearing_mint_initialize(CpiContext::new(...), Some(rate_authority.key()), rate_bps)?;
// 4. Initialize base mint last
anchor_spl::token_interface::initialize_mint2(CpiContext::new(...), decimals, &authority.key(), None)?;
```

### Incompatible Extension Pairs
- `NonTransferable + TransferHook`
- `ConfidentialTransfer + TransferHook/TransferFeeConfig/PermanentDelegate`

### NonTransferable Mint
```rust
non_transferable_mint_initialize(CpiContext::new(...))?;
// Note: NonTransferable automatically enforces ImmutableOwner on all token accounts
```

### Interest-Bearing Token
```rust
// Rate stored as basis points (500 = 5% annual)
// Balances NEVER change on-chain — interest is virtual (display-only)
// Continuous compounding: A = P × e^(rt)
// SECONDS_PER_YEAR = 31,556,736
// Time-weighted avg on rate change: new_avg = [(r1×t1) + (r2×t2)] / (t1+t2)

// rate_authority = all zeros → rate immutable
```

---

## 16. Instruction Introspection

### Reading Other Instructions in the Same Transaction
```rust
use anchor_lang::solana_program::sysvar::instructions::{
    load_current_index_checked, load_instruction_at_checked
};

pub fn claim(ctx: Context<Claim>, amount: u64) -> Result<()> {
    let sysvar = &ctx.accounts.instruction_sysvar;
    let current = load_current_index_checked(sysvar)?;

    // CRITICAL: use relative index, not absolute index 0 (see security below)
    let transfer_ix = load_instruction_at_checked((current - 1) as usize, sysvar)?;

    require_keys_eq!(transfer_ix.program_id, system_program::ID, MyError::InvalidPrecedingIx);

    let system_ix: SystemInstruction = bincode::deserialize(&transfer_ix.data)?;
    match system_ix {
        SystemInstruction::Transfer { lamports } => {
            require!(lamports >= amount, MyError::InsufficientTransfer);
        }
        _ => return err!(MyError::WrongInstructionType),
    }
    Ok(())
}

// Cargo.toml: bincode = "1.3.3"
```

**Security — Relative vs Absolute Index**:
- **Absolute index 0**: vulnerable — attacker places one valid tx at index 0 then makes N withdrawals
- **Relative `current_ix_index - 1`**: safe — always checks the immediately preceding instruction

---

## 17. Ed25519 Signature Verification

### Pattern: Two Instructions per Transaction
```
Instruction 0: Ed25519Program.createInstructionWithPublicKey (stateless verify)
Instruction 1: Your program's claim/process instruction
```

### Client-Side Signing
```typescript
import nacl from "tweetnacl";
import { Ed25519Program } from "@solana/web3.js";

const message = Buffer.alloc(40);
recipientPubkey.toBuffer().copy(message, 0);    // 32 bytes
message.writeBigUInt64LE(BigInt(amount), 32);   // 8 bytes

const signature = nacl.sign.detached(message, signerKeypair.secretKey);
const ed25519Ix = Ed25519Program.createInstructionWithPublicKey({
    publicKey: signerKeypair.publicKey.toBytes(),
    message,
    signature,
});

const tx = new Transaction().add(ed25519Ix).add(yourProgramIx);
```

### On-Chain Verification (via Instruction Sysvar)
```rust
// Validate the Ed25519 instruction preceding this one
let ix = load_instruction_at_checked((current - 1) as usize, sysvar)?;
require_keys_eq!(ix.program_id, ed25519_program::ID, MyError::InvalidSigProgram);
require!(ix.accounts.is_empty(), MyError::InvalidSigIx);
// All instruction indices in data header must equal u16::MAX (sentinel value)
```

**Rules**:
- Ed25519 instruction must immediately precede the claim instruction
- Each claim must have its own Ed25519 instruction (cannot share/reuse)
- Verify: signature count == 1, program_id matches, accounts.is_empty()

---

## 18. Native Solana (No Anchor)

### Program Structure
```toml
[lib]
crate-type = ["cdylib", "lib"]

[dependencies]
solana-program = "3.0.0"
borsh = { version = "0.10.3", features = ["derive"] }
```

```rust
use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint,
    entrypoint::ProgramResult,
    msg,
    pubkey::Pubkey,
};

entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    let accounts_iter = &mut accounts.iter();
    let storage_account = next_account_info(accounts_iter)?;
    // ...
    Ok(())
}
```

### Reading Account Data (Native)
```rust
let data = storage_account.try_borrow_data()?;
let counter_data = CounterData::try_from_slice(&data)?;
msg!("Count: {}", counter_data.count);

// Preview first 8 bytes
let preview = std::cmp::min(8, data.len());
msg!("Bytes: {:?}", &data[..preview]);
```

### Borsh Serialization (Native)
```rust
#[derive(BorshSerialize, BorshDeserialize)]
pub struct CounterData {
    pub count: u64,
}

// Serialize
let serialized = counter_data.try_to_vec()?;
account_data.copy_from_slice(&serialized);

// Deserialize
let counter = CounterData::try_from_slice(&data)?;
```

**Borsh wire format**:
- Fixed: `bool`=1B, `u8`=1B, `u16`=2B LE, `u32`=4B LE, `u64`=8B LE, `Pubkey`=32B
- Variable: 4-byte u32 length prefix + data
- String "hi" → `[0x02, 0x00, 0x00, 0x00, 0x68, 0x69]`
- Field order = declaration order; always little-endian

### Creating Accounts (Native — Keypair)
```rust
use solana_program::{program::invoke, system_instruction};

let space = std::mem::size_of::<CounterData>();
let lamports = Rent::default().minimum_balance(space);

invoke(
    &system_instruction::create_account(
        signer.key, storage_account.key, lamports, space as u64, program_id,
    ),
    &[signer.clone(), storage_account.clone(), system_program.clone()],
)?;

let mut account_data = storage_account.try_borrow_mut_data()?;
account_data.copy_from_slice(&serialized_data);
```

### Creating Accounts (Native — PDA)
```rust
use solana_program::program::invoke_signed;

let (expected_pda, bump) = Pubkey::find_program_address(&[b"seed"], program_id);
// Validate client-provided address
require!(*storage_account.key == expected_pda, ProgramError::InvalidAccountData);

let signer_seeds: &[&[u8]] = &[b"seed", &[bump]];
invoke_signed(
    &system_instruction::create_account(...),
    accounts,
    &[signer_seeds],
)?;
```

### Function Dispatching (Native)
```rust
const INITIALIZE: u8 = 0;
const INCREMENT: u8 = 1;
const RESET: u8 = 2;

match instruction_data[0] {
    INITIALIZE => process_initialize(program_id, accounts, &instruction_data[1..]),
    INCREMENT => process_increment(program_id, accounts, &instruction_data[1..]),
    RESET => process_reset(program_id, accounts),
    _ => {
        msg!("Unknown instruction");
        Err(ProgramError::InvalidInstructionData)
    }
}
// Anchor discriminator (Sha256 of "global:function_name")[0..8]
```

### Security Checks (Native — Critical)
```rust
// 1. Verify account ownership
if config.owner != program_id {
    return Err(ProgramError::IncorrectProgramId);
}

// 2. Verify known program IDs (the $320M Wormhole exploit skipped this)
if *system_program.key != system_program::ID {
    return Err(ProgramError::InvalidAccountData);
}

// 3. Require signer (check BOTH key AND is_signer)
if !admin.is_signer || *admin.key != cfg.admin {
    return Err(ProgramError::MissingRequiredSignature);
}

// 4. Require writable
if !user_account.is_writable {
    return Err(ProgramError::InvalidAccountData);
}

// 5. Reload after CPI (TOCTOU prevention)
let data = user_account.try_borrow_data()?;  // re-read after CPI

// 6. Burn dust before closing (attacker-injected tokens)
// Burn ALL tokens first, then close_account

// 7. Handle frontrun account creation
// If PDA already has lamports: use transfer + allocate + assign separately
// NOT create_account (which would fail if the account already exists)
```

---

## 19. Reading External Program Accounts

### On-Chain (Anchor)
```rust
#[derive(Accounts)]
pub struct ReadExternal<'info> {
    /// CHECK: We verify ownership manually before trusting data
    other_data: UncheckedAccount<'info>,
}

pub fn read_external(ctx: Context<ReadExternal>) -> Result<()> {
    // Verify ownership
    require!(ctx.accounts.other_data.owner == &other_program::ID, MyError::WrongOwner);

    let mut data: &[u8] = &ctx.accounts.other_data.data.borrow();
    let parsed: ExternalStorage = AccountDeserialize::try_deserialize(&mut data)?;
    msg!("Value: {}", parsed.x);
    Ok(())
}

// Local struct name MUST match the originating program's struct name exactly
// (discriminator = sha256("account:StructName")[0..8])
#[account]
pub struct ExternalStorage { x: u64 }
```

### Deserialization Warnings
- ✅ Anchor checks: discriminator match (struct name), sufficient bytes
- ❌ Anchor does NOT check: field names (positional), individual types, data integrity
- Type mismatch (e.g., reading u64 as u32): produces WRONG values silently, no error
- Field name mismatch: fails silently — only discriminator is verified
- Programs are upgradeable — external data formats can change unexpectedly

### Off-Chain (TypeScript)
```typescript
// Read any Anchor program's account using its IDL
const otherProgram = new anchor.Program(otherIdl, otherProgramId, provider);
const [pda] = anchor.web3.PublicKey.findProgramAddressSync(seeds, otherProgramId);
const data = await otherProgram.account.myStorage.fetch(pda);
console.log(data.x.toString());
```

---

## 20. Testing Patterns

### Basic Test Structure
```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { MyProgram } from "../target/types/my_program";

describe("my_program", () => {
    anchor.setProvider(anchor.AnchorProvider.env());
    const program = anchor.workspace.MyProgram as Program<MyProgram>;

    it("initializes storage", async () => {
        const [pda] = anchor.web3.PublicKey.findProgramAddressSync([], program.programId);
        await program.methods.initialize().accounts({ myStorage: pda }).rpc();
        const data = await program.account.myStorage.fetch(pda);
        assert.equal(data.x.toNumber(), 0);
    });
});
```

### Key TypeScript Patterns
```typescript
// BN (BigNumber) for u64/u128; NOT needed for u32 and below
new anchor.BN(1_000_000)

// Fetch possibly-closed accounts (returns null)
const acct = await program.account.myPDA.fetchNullable(address);

// Airdrop utility
async function airdropSol(address: PublicKey, amount: number) {
    const sig = await connection.requestAirdrop(address, amount);
    await connection.confirmTransaction(sig);
}

// Read raw balance
const lamports = await connection.getBalance(pubkey);

// PDA derivation (consistent with on-chain seeds)
const [pda, bump] = anchor.web3.PublicKey.findProgramAddressSync(
    [Buffer.from("my-seed"), userPubkey.toBuffer()],
    program.programId
);

// u64 little-endian seed
const key = new anchor.BN(42);
const seeds = [key.toArrayLike(Buffer, "le", 8)];  // 8 bytes LE
```

### Time Travel with LiteSVM
```typescript
import { LiteSVM } from "litesvm";
const svm = await LiteSVM.fromWorkspace("./")
    .withSplPrograms().withBuiltins().withSysvars();

// Advance clock
const c = svm.getClock();
svm.setClock(new Clock(c.slot, c.epochStartTimestamp, c.epoch, c.leaderScheduleEpoch, BigInt(newTimestamp)));
// Equivalent to Foundry's vm.warp()

// Every manual tx needs fresh blockhash
mintTx.recentBlockhash = svm.latestBlockhash();
```

### Testing Rules
- Always `--reset` the local validator between test runs for deterministic state
- Use `.transaction()` not `.rpc()` when batching instructions
- Use `.fetchNullable()` for possibly-closed accounts
- Provider wallet auto-signs — only add extra signers in `.signers([])`

---

## 21. IDL and TypeScript Interop

### IDL Structure
```json
{
    "version": "...",
    "name": "program_name",
    "instructions": [
        {
            "name": "myInstruction",
            "accounts": [{ "name": "myStorage", "isMut": true, "isSigner": false }],
            "args": [{ "name": "param", "type": "u64" }]
        }
    ],
    "accounts": [...],
    "errors": [...]
}
```

### Naming Convention
- Rust `snake_case` → TypeScript `camelCase` (pre-Anchor 0.30 auto-conversion)
- Struct names: unchanged
- Program name: dashes → underscores in IDL

### Test Against Deployed Program
```typescript
const idl = JSON.parse(fs.readFileSync('target/idl/my_program.json', 'utf-8'));
const programId = "6p29sM4hEK8ZFT5AvsGJQG5nKUtHBKs13iVL6juo5Uqj";
const program = new Program(idl, programId, provider);
await program.methods.initialize().rpc();
```

---

## 22. Anchor Account Constraints Reference

```rust
// Initialization
#[account(init, payer = signer, space = 8 + N)]
#[account(init, payer = signer, space = 8 + N, seeds = [...], bump)]
#[account(init_if_needed, payer = signer, space = 8 + N, seeds = [...], bump)]

// Mutation
#[account(mut)]                          // writable
#[account(mut, seeds = [...], bump)]     // mutable PDA (re-derive to verify)

// Authorization
#[account(has_one = authority)]
#[account(has_one = authority @ MyError::NotAuth)]
#[account(constraint = x.val > 0 @ MyError::InvalidVal)]
#[account(address = KNOWN_PUBKEY)]

// Resizing
#[account(mut, realloc = NEW_SIZE, realloc::payer = signer, realloc::zero = false)]

// Closing
#[account(mut, close = recipient)]

// Token-specific
#[account(mint::decimals = 6, mint::authority = authority)]
#[account(token::mint = mint, token::authority = authority)]
```

---

## 23. Solana sBPF Internals (Reference)

### Virtual Machine Architecture
- **Register-based** (not stack-based like EVM)
- 11 registers: R0–R10, each 64-bit
  - R0: return value / exit code
  - R1–R5: function arguments (caller-saved, dirty after calls)
  - R6–R9: callee-saved (must preserve)
  - R10: read-only frame pointer

### Memory Regions (each 4 GiB)
| Region | Base Address | Purpose |
|---|---|---|
| `MM_RODATA_START` | `0x000000000` | Constants / static strings |
| `MM_BYTECODE_START` | `0x100000000` | Program bytecode |
| `MM_STACK_START` | `0x200000000` | Execution stack |
| `MM_HEAP_START` | `0x300000000` | Heap (32KB, bump allocator) |
| `MM_INPUT_START` | `0x400000000` | Serialized instruction inputs |

### Key Facts
- Each sBPF instruction = exactly 1 CU
- Heap: 32KB max, bump allocator (never frees), resets per execution
- Stack: 4KB per frame, grows downward from R10
- R1 = `0x400000000` on entry (points to serialized account data)
- Three compilation stages: Rust → LLVM IR → SBF bytecode → JIT to native (at runtime)

### Compute Cost Breakdown
```
Total CU = (sBPF instruction count) + (syscall charges)
Each sBPF instruction: 1 CU
sol_log_ / sol_log_data: variable
sol_log_64_ / sol_log_pubkey / sol_log_compute_units_: 100 CU each
Anchor auto-log on entry: ~100 CU
```

---

## 24. Switchboard Oracle (Pull-Based Feeds)

```toml
# Cargo.toml
switchboard-on-demand = "0.5.3"
```

```rust
use switchboard_on_demand::PullFeedAccountData;

pub fn check_price(ctx: Context<CheckPrice>) -> Result<()> {
    let data_slice = ctx.accounts.feed.data.borrow();
    let feed = PullFeedAccountData::parse(data_slice).unwrap();
    let price: Decimal = feed.get_value(
        &Clock::get()?,
        100,    // max_stale_slots
        3,      // min_samples
        true,   // only_positive
    ).unwrap();
    msg!("Price: {:?}", price);
    Ok(())
}
```

**Rules**:
- Feed is empty until off-chain updater script runs (`runfeeds.ts`)
- Stale data (current_slot - last_update > max_stale_slots) → tx fails
- Two-script setup: `initializeFeeds.ts` (once) + `runfeeds.ts` (continuous)

---

## 25. Quick Reference — Common Pitfalls

| Pitfall | Fix |
|---|---|
| Missing `+ 8` in space | Always `size_of::<T>() + 8` for discriminator |
| `UncheckedAccount` without `/// CHECK:` | Add the mandatory doc comment |
| Forgetting `#[account(mut)]` | Mark any account whose data/lamports change |
| Provider wallet in `.signers([])` | It auto-signs; only add extra signers |
| `init_if_needed` + `sub_lamports` | Reinitialization vulnerability — add guards |
| Reading external data without ownership check | Always verify `.owner` before trusting |
| Seeds mismatch between init and lookup | Keep seeds identical; use `#[instruction(...)]` for args |
| Absolute index in instruction introspection | Use `current_ix_index - 1` (relative) |
| Float operations in hot paths | Use integer math; floats cost ~5× more CU |
| `create_account` on existing PDA | Use `transfer + allocate + assign` instead |
| Forgetting `realloc::zero` flag | Explicit; `false` = preserve data, `true` = erase |
| Token extension before base mint init | Initialize extension first, base mint second |
| Hardcoded authority without on-chain storage | Store in `AdminConfig` account for upgradeability |
| `msg!` before failed `require!` appears in logs | Unlike Solidity, Solana logs don't roll back |

---

## 26. Deployment Checklist

```bash
# 1. Sync program ID
anchor keys sync

# 2. Build
anchor build

# 3. Test locally
anchor test

# 4. Deploy to devnet
anchor deploy --provider.cluster devnet

# 5. Verify deployment
solana program show <PROGRAM_ID>

# Post-deploy:
# - Save program ID in README / config
# - Test against deployed program: anchor test --skip-local-validator --skip-deploy
# - To make immutable: solana program set-upgrade-authority <ID> --final
```

### Programs Are Upgradeable by Default
- No proxy patterns needed (direct bytecode replacement)
- Program ID stays the same across upgrades
- No constructors — initialize via explicit first-call function
- No immutable variables — aligns with upgradeable design

---

## 27. Solana vs EVM — Complete Comparison

| Feature | EVM/Solidity | Solana/Anchor |
|---|---|---|
| Storage model | Slot-keyed in contract | Separate owned accounts |
| msg.sender | Global | `Signer<'info>` explicit field |
| Constructor | `constructor()` | Explicit `initialize()` function |
| Tx fees | Based on gas used | Based on signature count only |
| Events (query) | Log index queryable | Real-time only; use tx history |
| Reentrancy guard | Manual mutex / CEI | Borrow checker prevents data races |
| Upgradeable | Proxy + delegatecall | Direct replacement; same address |
| Token standard | ERC-20 (approval: N parties) | SPL Token (approval: 1 delegate) |
| Integer default | uint256 | u64 |
| Overflow | SafeMath / builtin 0.8+ | `checked_*` / `overflow-checks` |
| Null | `address(0)` | `Option<T>` |
| Error handling | try/catch + revert | `Result<()>` + `?` |
| Batch calls | External multicall contract | Native atomic batching |
| SLOAD/SSTORE | Per-slot reads/writes | Account borrows (safe: one writer) |
| View functions | `view` modifier | No — structural (not in Context = not accessible) |
| msg.value | Implicit in `payable` | Must CPI to System Program |
