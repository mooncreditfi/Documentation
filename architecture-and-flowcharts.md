# MoonCreditFi Blockchain Architecture & Flowcharts

**Trust Before Capital**

Technical architecture and workflow documentation for MoonCreditFi's Solana blockchain infrastructure.

| Property | Details |
|---|---|
| Project | MoonCreditFi |
| Blockchain | Solana |
| Smart Contract | Rust + Anchor |
| Token Standard | SPL Token |
| Settlement Asset | USDC |
| Network | Solana Devnet |
| State Management | Program Derived Addresses (PDAs) |
| Primary Workflow | Mudarabah Business Financing |
| Documentation Type | Blockchain Architecture & Flowcharts |

---

# 1. Architecture Overview

MoonCreditFi uses a hybrid blockchain architecture that separates on-chain financial execution from off-chain application services.

The Solana program manages financial operations and enforces the rules governing investor contributions, funding pools, capital disbursement, settlements, and distributions.

The frontend provides the interface through which entrepreneurs and investors interact with the protocol.

The backend manages application data, user profiles, business information, verification processes, and other off-chain services.

## 1.1 High-Level Architecture

```mermaid
flowchart TB
    U["Platform Users"]

    U --> E["Entrepreneur"]
    U --> I["Investor"]
    U --> A["Platform Administrator"]

    E --> FE["MoonCreditFi Frontend"]
    I --> FE
    A --> FE

    FE --> W["Solana Wallet Adapter"]
    FE --> BE["Backend Services"]
    FE --> RPC["Solana RPC"]

    BE --> DB["Supabase / Application Database"]
    BE --> V["Business Verification"]
    BE --> R["Reputation Services"]

    RPC --> P["MoonCreditFi Anchor Program"]

    P --> CFG["PlatformConfig PDA"]
    P --> EP["EntrepreneurProfile PDA"]
    P --> BP["BusinessProposal PDA"]
    P --> MP["MudarabahPool PDA"]
    P --> IP["InvestorPosition PDA"]
    P --> SR["SettlementRecord PDA"]

    MP --> VAULT["Pool USDC Vault"]
    VAULT --> TOKEN["SPL Token Program"]

    P --> EVENTS["Program Events"]
    EVENTS --> RPC
```

## 1.2 Architecture Responsibilities

| Layer | Responsibility |
|---|---|
| Frontend | User interaction, wallet connection, transaction construction |
| Wallet | Transaction and message signing |
| Backend | User profiles, business information, verification and application services |
| Supabase | Off-chain application data |
| Solana RPC | Blockchain communication and account retrieval |
| Anchor Program | Financial rules and instruction execution |
| PDAs | Program-controlled state and deterministic account addresses |
| SPL Token Program | USDC token transfers |
| USDC Vault | Holds pool contributions and settlement funds |

**Important:** The frontend and backend do not independently authorize blockchain transactions. The smart contract validates the supplied accounts and required signatures when instructions execute.

---

# 2. On-chain Account Architecture

MoonCreditFi uses program-owned accounts to represent the platform's financial and operational state.

## 2.1 Account Relationship Diagram

```mermaid
flowchart TD
    PROGRAM["MoonCreditFi Program"]

    PROGRAM --> CONFIG["PlatformConfig"]
    PROGRAM --> ENTRE["EntrepreneurProfile"]
    PROGRAM --> PROPOSAL["BusinessProposal"]
    PROGRAM --> POOL["MudarabahPool"]
    PROGRAM --> POSITION["InvestorPosition"]
    PROGRAM --> SETTLEMENT["SettlementRecord"]

    CONFIG --> AUTH["Platform Authority"]
    CONFIG --> MINT["Approved USDC Mint"]

    ENTRE --> OWNER["Entrepreneur Wallet"]
    ENTRE --> PROPOSAL

    PROPOSAL --> POOL

    POOL --> VAULT["USDC Vault"]
    POOL --> POSITION
    POOL --> SETTLEMENT

    POSITION --> INVESTOR["Investor Wallet"]
    VAULT --> TOKEN["SPL Token Program"]
```

## 2.2 Account Descriptions

| Account | Purpose |
|---|---|
| PlatformConfig | Stores global platform configuration and authority |
| EntrepreneurProfile | Represents an entrepreneur's on-chain registration |
| BusinessProposal | Stores financing proposal information |
| MudarabahPool | Stores pool funding and settlement state |
| InvestorPosition | Tracks an investor's contribution and claim status |
| SettlementRecord | Stores submitted and approved settlement information |
| USDC Vault | Holds the USDC associated with a financing pool |

These relationships describe the documented architecture. Exact account fields and constraints must be checked against the deployed Rust source and IDL.

---

# 3. PDA Architecture

Program Derived Addresses provide deterministic account locations and allow the program to authorize specific operations without maintaining a conventional private key for each PDA.

## 3.1 PDA Derivation Flow

```mermaid
flowchart TD
    A["MoonCreditFi Program ID"]

    A --> B["PDA Seeds"]
    B --> C["Address Derivation"]
    C --> D["Canonical Bump"]
    D --> E["Derived Account Address"]

    E --> F["Anchor Account Validation"]
    F --> G{"Expected PDA?"}

    G -->|Yes| H["Continue Instruction"]
    G -->|No| I["Reject Transaction"]
```

## 3.2 Documented PDA Design

| Account | Suggested Seeds |
|---|---|
| PlatformConfig | `["platform_config"]` |
| EntrepreneurProfile | `["entrepreneur", entrepreneur_pubkey]` |
| BusinessProposal | `["proposal", entrepreneur_pubkey, proposal_id]` |
| MudarabahPool | `["pool", proposal_pubkey]` |
| InvestorPosition | `["position", pool_pubkey, investor_pubkey]` |
| SettlementRecord | `["settlement", pool_pubkey]` |
| Pool vault authority | `["vault_authority", pool_pubkey]` |

These are the proposed seed conventions in the architecture draft. The actual deployed program's seed definitions are authoritative.

## 3.3 PDA Security Flow

```mermaid
flowchart TD
    A["Instruction Received"]
    B["Read Supplied Accounts"]
    C["Derive Expected PDA"]
    D{"Address Matches?"}
    E["Validate Account Ownership"]
    F["Validate Account Data"]
    G["Execute Instruction"]
    H["Return Error"]

    A --> B --> C --> D
    D -->|Yes| E --> F --> G
    D -->|No| H
```

---

# 4. USDC Vault Architecture

Each financing pool is intended to have a dedicated USDC vault controlled through the program's PDA authority.

Investors contribute USDC to the vault. When the relevant conditions are satisfied, the program can authorize transfers through the SPL Token Program.

## 4.1 Vault Relationship

```mermaid
flowchart TB
    INVESTOR["Investor Wallet"]

    INVESTOR --> SOURCE["Investor USDC Token Account"]

    SOURCE -->|"Contribution"| TOKEN["SPL Token Program"]

    TOKEN --> VAULT["Pool USDC Vault"]

    AUTH["Vault Authority PDA"] --> PROGRAM["MoonCreditFi Program"]

    PROGRAM -->|"Authorized CPI"| TOKEN

    VAULT -->|"Approved Distribution"| TOKEN

    TOKEN --> INVESTOR2["Investor Token Account"]

    TOKEN --> ENTRE["Entrepreneur Token Account"]
```

## 4.2 Contribution Transaction Flow

```mermaid
sequenceDiagram
    actor Investor
    participant Frontend
    participant Wallet
    participant RPC
    participant Program as MoonCreditFi Program
    participant Token as SPL Token Program

    Investor->>Frontend: Select funding pool
    Frontend->>RPC: Fetch pool and account state
    RPC-->>Frontend: Return pool information

    Investor->>Frontend: Enter contribution amount
    Frontend->>Wallet: Request transaction signature
    Wallet-->>Frontend: Signed transaction

    Frontend->>RPC: Submit transaction
    RPC->>Program: Execute contribute instruction

    Program->>Program: Validate signer and pool
    Program->>Program: Validate mint and vault
    Program->>Program: Validate contribution limit

    Program->>Token: TransferChecked via CPI
    Token-->>Program: Transfer result

    Program->>Program: Update InvestorPosition
    Program->>Program: Update total contributions

    Program-->>RPC: Emit contribution event
    RPC-->>Frontend: Transaction result
    Frontend-->>Investor: Display confirmation
```

## 4.3 Contribution Validation

The contract should verify:

- Investor authorization.
- Correct pool address.
- Correct USDC mint.
- Correct source token account.
- Correct pool vault.
- Correct vault authority.
- Positive contribution amount.
- Pool status is Open.
- Contribution does not exceed the remaining funding target.

A successful frontend submission is not sufficient proof of a successful contribution. The transaction must be confirmed and the resulting on-chain state checked.

---

# 5. Entrepreneur Financing Workflow

The entrepreneur financing workflow starts with registration and ends with the creation of an approved financing pool.

## 5.1 Entrepreneur Proposal Flowchart

```mermaid
flowchart TD
    START(["Start"])

    START --> CONNECT["Connect Solana Wallet"]
    CONNECT --> REGISTER["Register Entrepreneur"]
    REGISTER --> PROFILE["Create Entrepreneur Profile"]

    PROFILE --> CREATE["Submit Business Proposal"]
    CREATE --> REVIEW["Platform Reviews Proposal"]

    REVIEW --> DECISION{"Proposal Approved?"}

    DECISION -->|No| REJECT["Proposal Rejected or Returned"]
    REJECT --> END1(["End"])

    DECISION -->|Yes| APPROVE["approve_proposal"]
    APPROVE --> CREATEPOOL["create_pool"]

    CREATEPOOL --> VAULT["Initialize Pool Vault"]
    VAULT --> OPEN["Pool Status: Open"]

    OPEN --> END2(["Funding Available"])
```

## 5.2 Workflow Instructions

| Step | Instruction | Description |
|---|---|---|
| 1 | `register_entrepreneur` | Registers the entrepreneur |
| 2 | `create_proposal` | Creates a business financing proposal |
| 3 | `approve_proposal` | Authorizes the proposal |
| 4 | `create_pool` | Creates the financing pool and vault |

Proposal review is an administrative process. A proposal must not be treated as approved simply because it exists on-chain.

---

# 6. Investor Funding Workflow

After a pool is created and opened, investors can contribute USDC.

## 6.1 Funding Flowchart

```mermaid
flowchart TD
    A["Open Funding Pool"]
    B["Investor Reviews Terms"]
    C["Investor Connects Wallet"]
    D["Enter USDC Contribution"]
    E["Sign Transaction"]
    F["Execute contribute"]
    G{"Contract Validation"}

    H["Transfer USDC to Vault"]
    I["Create or Update InvestorPosition"]
    J["Update Pool Total"]
    K{"Funding Target Reached?"}

    L["Continue Accepting Contributions"]
    M["Pool Status: Funded"]
    N["Stop Further Contributions"]

    A --> B --> C --> D --> E --> F --> G

    G -->|Invalid| X["Reject Transaction"]
    G -->|Valid| H --> I --> J --> K

    K -->|No| L
    L --> B

    K -->|Yes| M --> N
```

## 6.2 Funding Rules

- Contributions must be greater than zero.
- Contributions must not exceed the funding target.
- Only eligible pools can receive contributions.
- Investor positions must be associated with the correct pool and wallet.
- Contributions must be recorded using integer token base units.

---

# 7. Pool Activation and Capital Disbursement

Once the pool reaches its funding target, the activation and capital disbursement process can begin.

The documented contract flow requires the entrepreneur to activate the pool. Capital disbursement requires the entrepreneur and platform administrator signatures.

## 7.1 Activation Flowchart

```mermaid
flowchart TD
    A["Pool Status: Open"]
    B["Check Total Contributions"]
    C{"Target Reached?"}

    D["Remain Open"]
    E["Entrepreneur Activates Pool"]
    F["activate_pool"]
    G["Pool Status: Funded"]

    H["Entrepreneur + Admin Authorize"]
    I["disburse_capital"]
    J["Validate Vault and Destination"]
    K["Transfer USDC to Entrepreneur"]
    L["Pool Status: Active"]

    A --> B --> C
    C -->|No| D
    C -->|Yes| E --> F --> G
    G --> H --> I --> J --> K --> L
```

## 7.2 Capital Disbursement Sequence

```mermaid
sequenceDiagram
    actor Entrepreneur
    actor Admin as Platform Admin
    participant Frontend
    participant Program as MoonCreditFi Program
    participant Vault as USDC Vault
    participant Token as SPL Token Program

    Entrepreneur->>Frontend: Request pool activation
    Frontend->>Program: activate_pool
    Program->>Program: Validate funding target
    Program->>Program: Validate pool status
    Program-->>Frontend: Pool funded

    Entrepreneur->>Frontend: Request capital disbursement
    Frontend->>Program: disburse_capital
    Admin->>Frontend: Provide required authorization

    Program->>Program: Validate required signers
    Program->>Program: Validate pool and vault

    Program->>Token: Transfer USDC using PDA authority
    Token->>Vault: Debit pool vault
    Token-->>Program: Transfer result

    Program->>Program: Update pool status to Active
    Program-->>Frontend: Transaction result
```

**Financial risk:** Once capital has been transferred to the entrepreneur, the contract cannot guarantee repayment. Capital recovery depends on the business outcome, settlement process, and applicable agreements.

---

# 8. Settlement and Distribution Workflow

After the business activity, the entrepreneur submits settlement information.

The platform administrator reviews the settlement and approves the amount that will be used for distribution.

## 8.1 Settlement Flowchart

```mermaid
flowchart TD
    A["Pool Status: Active"]
    B["Business Activity"]
    C["Entrepreneur Prepares Settlement"]
    D["submit_settlement"]
    E["Pool Status: SettlementPending"]
    F["Platform Reviews Settlement"]
    G{"Settlement Approved?"}

    H["Request Further Evidence or Reject"]
    I["approve_settlement"]
    J["Record Approved Return Amount"]
    K["Pool Status: Settled"]

    A --> B --> C --> D --> E --> F --> G

    G -->|No| H
    G -->|Yes| I --> J --> K
```

## 8.2 Settlement Sequence

```mermaid
sequenceDiagram
    actor Entrepreneur
    actor Admin as Platform Admin
    participant Frontend
    participant Program as MoonCreditFi Program
    participant Vault as USDC Vault
    participant Token as SPL Token Program

    Entrepreneur->>Frontend: Submit business settlement
    Frontend->>Program: submit_settlement

    Program->>Program: Validate entrepreneur
    Program->>Program: Validate pool status
    Program->>Program: Record settlement
    Program-->>Frontend: Settlement pending

    Entrepreneur->>Frontend: Return USDC to pool vault
    Frontend->>Program: Submit return transaction
    Program->>Token: TransferChecked
    Token->>Vault: Deposit returned USDC

    Admin->>Frontend: Review settlement
    Frontend->>Program: approve_settlement

    Program->>Program: Validate admin authority
    Program->>Program: Record approved return amount
    Program->>Program: Set pool to Settled

    Program-->>Frontend: Settlement approved
```

The returned funds must be deposited into the pool vault before they can be distributed. The settlement approval process does not itself prove the accuracy of the business's reported results.

---

# 9. Profit and Loss Distribution

MoonCreditFi uses a Mudarabah-inspired profit-sharing model.

The investor and entrepreneur agree on profit-sharing percentages. Investors provide capital, while the entrepreneur manages the business.

## 9.1 Profit Distribution Flowchart

```mermaid
flowchart TD
    A["Approved Settlement"]
    B["Calculate Total Returned"]
    C["Calculate Realized Profit"]
    D{"Profit Greater Than Zero?"}

    E["Calculate Investor Profit Share"]
    F["Calculate Entrepreneur Profit Share"]
    G["Investors Claim Distribution"]
    H["Track Investor Claims"]
    I{"All Investor Claims Accounted For?"}
    J["Entrepreneur Claims Profit"]

    K["Calculate Capital Loss"]
    L["Allocate Returned Capital Proportionally"]
    M["Investors Claim Available Capital"]

    N["Complete Distribution"]

    A --> B --> C --> D

    D -->|Yes| E --> F --> G --> H --> I
    I -->|No| H
    I -->|Yes| J --> N

    D -->|No| K --> L --> M --> N
```

## 9.2 Example: Profit Scenario

Assume:

| Item | Amount |
|---|---:|
| Total investor capital | 10,000 USDC |
| Total returned | 12,000 USDC |
| Realized profit | 2,000 USDC |
| Investor profit share | 70% |
| Entrepreneur profit share | 30% |

Calculation:

```text
Realized Profit = 12,000 - 10,000

Investor Profit = 2,000 × 70%
               = 1,400 USDC

Entrepreneur Profit = 2,000 × 30%
                    = 600 USDC
```

Investors collectively receive 11,400 USDC, while the entrepreneur receives 600 USDC.

These amounts are illustrative and are not guaranteed returns.

## 9.3 Example: Loss Scenario

Assume:

| Item | Amount |
|---|---:|
| Total investor capital | 10,000 USDC |
| Total returned | 8,000 USDC |
| Business loss | 2,000 USDC |

The available 8,000 USDC is distributed among investors in proportion to their respective contributions.

The entrepreneur receives no profit distribution in this example.

Liability for negligence, misconduct, fraud, or contractual breach requires an appropriate determination.

## 9.4 Distribution Sequence

```mermaid
sequenceDiagram
    actor Investor
    actor Entrepreneur
    participant Frontend
    participant Program as MoonCreditFi Program
    participant Vault as USDC Vault
    participant Token as SPL Token Program

    Investor->>Frontend: Request distribution
    Frontend->>Program: claim_distribution

    Program->>Program: Validate investor position
    Program->>Program: Calculate entitlement
    Program->>Program: Check claimed status

    Program->>Token: Transfer investor entitlement
    Token->>Vault: Debit vault
    Token-->>Program: Transfer result

    Program->>Program: Mark investor position claimed

    Program-->>Frontend: Claim confirmation

    Entrepreneur->>Frontend: Request entrepreneur profit
    Frontend->>Program: claim_entrepreneur_profit

    Program->>Program: Check investor claim accounting
    Program->>Program: Calculate entrepreneur profit
    Program->>Token: Transfer eligible profit
    Program-->>Frontend: Profit claim result
```

---

# 10. Pool Cancellation and Refund Flow

A pool may be cancelled under the conditions enforced by the contract.

The documented cancellation flow supports cancellation of eligible Open or Funded pools. Investors can then claim eligible refunds.

## 10.1 Cancellation Flowchart

```mermaid
flowchart TD
    A["Pool Exists"]
    B{"Current Pool Status"}

    C["Open"]
    D["Funded"]
    E["Active or Later"]

    F["Admin Calls cancel_pool"]
    G["Validate Cancellation Conditions"]
    H["Pool Status: Cancelled"]
    I["Investor Calls claim_refund"]
    J["Validate Investor Position"]
    K["Transfer Refund"]
    L["Mark Refund Claimed"]
    M["Refund Completed"]

    N["Cancellation Not Permitted"]

    A --> B
    B -->|Open| C --> F
    B -->|Funded| D --> F
    B -->|Active or Later| E --> N

    F --> G --> H --> I --> J --> K --> L --> M
```

## 10.2 Refund Rules

- Only cancelled pools can process refunds.
- The investor must own the corresponding position.
- Refunds must not exceed the investor's refundable contribution.
- Duplicate refunds must be rejected.
- The pool vault must have sufficient funds.

---

# 11. Complete Mudarabah Lifecycle

This diagram combines the principal financing stages into one view.

```mermaid
flowchart TD
    START(["Start"])

    START --> REG["Entrepreneur Registration"]
    REG --> PROP["Business Proposal"]
    PROP --> REVIEW["Platform Review"]

    REVIEW --> APPROVED{"Approved?"}
    APPROVED -->|No| REJECT["Rejected or Returned"]
    REJECT --> END1(["End"])

    APPROVED -->|Yes| POOL["Create Mudarabah Pool"]
    POOL --> OPEN["Open for Contributions"]

    OPEN --> FUND["Investor USDC Contributions"]
    FUND --> TARGET{"Funding Target Reached?"}

    TARGET -->|No| FUND
    TARGET -->|Yes| ACTIVATION["Activate Pool"]

    ACTIVATION --> DISBURSE["Disburse Capital"]
    DISBURSE --> ACTIVE["Business Activity"]

    ACTIVE --> SETTLEMENT["Submit Settlement"]
    SETTLEMENT --> APPROVAL["Settlement Review"]

    APPROVAL --> SETTLED["Approve Settlement"]
    SETTLED --> DISTRIBUTION["Calculate Profit or Loss"]

    DISTRIBUTION --> CLAIMS["Investor Claims"]
    CLAIMS --> ENTREPRENEUR["Entrepreneur Profit Claim"]
    ENTREPRENEUR --> END2(["Lifecycle Completed"])
```

This is the main financing lifecycle. The cancellation and refund path is handled separately.

---

# 12. Smart Contract Instruction Map

| Instruction | Workflow |
|---|---|
| `initialize_platform` | Platform initialization |
| `register_entrepreneur` | Entrepreneur onboarding |
| `create_proposal` | Financing proposal creation |
| `approve_proposal` | Proposal approval |
| `create_pool` | Funding pool creation |
| `contribute` | Investor funding |
| `activate_pool` | Pool activation |
| `disburse_capital` | Capital transfer |
| `submit_settlement` | Settlement submission |
| `approve_settlement` | Settlement approval |
| `claim_distribution` | Investor distribution |
| `claim_entrepreneur_profit` | Entrepreneur profit claim |
| `cancel_pool` | Pool cancellation |
| `claim_refund` | Investor refund |

---

# 13. Transaction Security Architecture

Every financial instruction must validate its accounts and authorization before executing.

## 13.1 Transaction Validation Flow

```mermaid
flowchart TD
    A["Transaction Submitted"]
    B["Wallet Signature Verification"]
    C["Program Instruction Execution"]
    D["Account Ownership Validation"]
    E["PDA Validation"]
    F["Signer Authorization"]
    G["Token and Mint Validation"]
    H["Pool State Validation"]
    I["Arithmetic and Balance Validation"]
    J{"All Checks Passed?"}

    K["Execute Financial Operation"]
    L["Reject Transaction"]

    A --> B --> C --> D --> E --> F --> G --> H --> I --> J
    J -->|Yes| K
    J -->|No| L
```

## 13.2 Security Requirements

1. Validate every required signer.
2. Validate every PDA using the correct seeds and bump.
3. Validate the token mint and token program.
4. Validate source and destination token accounts.
5. Prevent duplicate claims.
6. Prevent unauthorized capital disbursement.
7. Use checked integer arithmetic.
8. Enforce valid pool status transitions.
9. Restrict administrative operations.
10. Define upgrade authority governance.

A successful transaction signature alone does not establish that the real-world business activity or settlement is truthful.

---

# 14. Important Architecture Limitations

The diagrams describe the intended contract and application architecture. They do not prove that every component has been implemented or exposed through the frontend.

The following items require explicit verification:

- Whether all documented account fields match the Rust structs.
- Whether the proposed PDA seeds match the deployed program.
- Whether the frontend exposes every contract instruction.
- Whether returned settlement funds are deposited before approval.
- Whether all investor distributions are fully accounted for.
- How unclaimed investor funds are handled.
- How rounding residuals are reconciled.
- Whether emergency pause functionality exists.
- Whether administrative control is protected by multisig governance.

**Critical financial consideration:** The smart contract can enforce how a submitted settlement is distributed. It cannot independently establish whether the entrepreneur's reported business profit or loss is genuine.

---

# 15. Documentation References

- [Solana Core Concepts](https://solana.com/docs/core)
- [Solana Program Derived Addresses](https://solana.com/docs/core/pda)
- [Solana Token Program CPI](https://solana.com/docs/tokens/advanced/cpi)
- [Solana Program Development](https://solana.com/developers/docs/programs)

---

# 16. Conclusion

MoonCreditFi's blockchain architecture connects entrepreneurs, investors, application services, Solana wallets, and the Anchor smart contract.

The core on-chain workflow is:

1. Register entrepreneurs.
2. Submit and approve financing proposals.
3. Create Mudarabah funding pools.
4. Accept investor USDC contributions.
5. Activate fully funded pools.
6. Disburse capital to entrepreneurs.
7. Submit and approve settlements.
8. Distribute realized profit or returned capital.
9. Process eligible investor refunds when pools are cancelled.

The architecture aims to make financial operations transparent, programmable, and verifiable while recognizing that business verification, settlement evidence, legal agreements, and governance require additional off-chain processes.

**MoonCreditFi — Trust Before Capital.**
