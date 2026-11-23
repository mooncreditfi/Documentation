# MoonCreditFi Blockchain Architecture & Flowcharts

**Trust Before Capital**

Technical architecture documentation for the MoonCreditFi Solana blockchain infrastructure.

| Property | Details |
|---|---|
| Project | MoonCreditFi |
| Blockchain | Solana |
| Smart Contract Language | Rust |
| Framework | Anchor |
| Token | USDC |
| Network | Solana Devnet |
| Architecture | Program-based financial infrastructure |
| Status | Development |

---

# 1. Overview

MoonCreditFi uses Solana smart contracts to manage the on-chain components of a Mudarabah-based business financing system.

The blockchain architecture connects four major participants and components:

1. Entrepreneurs — Submit business financing proposals.
2. Investors — Contribute USDC to financing pools.
3. MoonCreditFi Smart Contracts — Enforce financial rules and manage on-chain state.
4. Backend Infrastructure — Handles business information, verification, private documents, and off-chain services.

The architecture separates business information and verification from financial operations executed on-chain.

The Solana program controls the financial rules and token movements that are implemented in the contracts.

# 2. High-Level System Architecture

The MoonCreditFi system consists of three major infrastructure layers:

- Frontend application.
- Backend services.
- Solana blockchain.

## 2.1 Overall architecture diagram

```mermaid
flowchart TB
    subgraph USERS["Platform Participants"]
        E[Entrepreneur]
        I[Investor]
        R[Authorized Reviewer]
    end

    subgraph FRONTEND["Frontend Layer"]
        UI[MoonCreditFi Web Application]
        AUTH[SIWS Authentication]
        WALLET[Solana Wallet Integration]
        CLIENT[Anchor Program Client]
    end

    subgraph BACKEND["Backend Layer"]
        API[Backend Services]
        DB[(Supabase PostgreSQL)]
        STORAGE[Private Document Storage]
        VERIFY[Business Verification]
        CRED[CredLayer Reputation]
    end

    subgraph BLOCKCHAIN["Solana Blockchain"]
        PROGRAM[MoonCreditFi Anchor Program]
        CONFIG[Platform Configuration]
        PROFILE[Entrepreneur Profile]
        PROPOSAL[Business Proposal]
        POOL[Mudarabah Pool]
        POSITION[Investor Position]
        SETTLEMENT[Settlement Record]
        VAULT[USDC Pool Vault]
    end

    E --> UI
    I --> UI
    R --> UI

    UI --> AUTH
    UI --> WALLET
    UI --> API
    UI --> CLIENT

    API --> DB
    API --> STORAGE
    API --> VERIFY
    API --> CRED

    CLIENT --> PROGRAM
    WALLET --> PROGRAM

    PROGRAM --> CONFIG
    PROGRAM --> PROFILE
    PROGRAM --> PROPOSAL
    PROGRAM --> POOL
    PROGRAM --> POSITION
    PROGRAM --> SETTLEMENT
    PROGRAM --> VAULT
```

## 2.2 Architecture responsibilities

| Layer | Responsibility |
|---|---|
| Frontend | User interaction, dashboards, transaction initiation |
| Backend | Profiles, business data, verification and private records |
| Solana | Financial state and program-enforced operations |
| Wallet | Authentication signatures and transaction authorization |
| CredLayer | Reputation and wallet risk signals |

The backend must not be treated as the authority for actual on-chain balances.

Likewise, the blockchain cannot independently establish whether submitted business information is truthful.

# 3. Smart Contract Architecture

The MoonCreditFi Anchor program is the core on-chain financial engine.

It manages the relationship between platform configuration, entrepreneurs, financing proposals, pools, investors, and settlements.

## 3.1 Program account architecture

```mermaid
flowchart TD
    PROGRAM["MoonCreditFi Anchor Program"]

    PROGRAM --> CONFIG["PlatformConfig PDA"]

    CONFIG --> ENTRE["EntrepreneurProfile PDA"]

    ENTRE --> PROPOSAL["BusinessProposal PDA"]

    PROPOSAL --> POOL["MudarabahPool PDA"]

    POOL --> POSITION["InvestorPosition PDA"]

    POOL --> SETTLEMENT["SettlementRecord PDA"]

    POOL --> VAULTAUTH["Vault Authority PDA"]

    VAULTAUTH --> VAULT["USDC Token Vault"]
```

## 3.2 Account responsibilities

| Account | Responsibility |
|---|---|
| PlatformConfig | Stores global platform configuration |
| EntrepreneurProfile | Represents an entrepreneur on-chain |
| BusinessProposal | Represents a financing proposal |
| MudarabahPool | Maintains pool financial state |
| InvestorPosition | Tracks an investor's contribution and claim |
| SettlementRecord | Records settlement submission and approval |
| Vault Authority | Controls the pool vault through PDA signing |
| USDC Vault | Holds pool USDC tokens |

The vault authority is a PDA used to authorize token transfers. The USDC vault itself is a token account.

## 3.3 Account relationships

```mermaid
erDiagram
    PLATFORM_CONFIG ||--o{ ENTREPRENEUR_PROFILE : manages
    ENTREPRENEUR_PROFILE ||--o{ BUSINESS_PROPOSAL : creates
    BUSINESS_PROPOSAL ||--o| MUDARABAH_POOL : funds
    MUDARABAH_POOL ||--o{ INVESTOR_POSITION : contains
    MUDARABAH_POOL ||--o| SETTLEMENT_RECORD : settles
    MUDARABAH_POOL ||--|| USDC_VAULT : controls

    PLATFORM_CONFIG {
        pubkey authority
        pubkey usdc_mint
        bool paused
        uint8 bump
    }

    ENTREPRENEUR_PROFILE {
        pubkey entrepreneur
        uint64 profile_id
        uint8 bump
    }

    BUSINESS_PROPOSAL {
        pubkey entrepreneur
        uint64 proposal_id
        uint64 requested_amount
        bool approved
    }

    MUDARABAH_POOL {
        pubkey proposal
        pubkey entrepreneur
        uint64 funding_target
        uint64 total_contributed
        uint64 total_returned
        uint16 investor_profit_bps
        uint16 entrepreneur_profit_bps
    }

    INVESTOR_POSITION {
        pubkey pool
        pubkey investor
        uint64 contribution
        uint64 claimed_amount
    }

    SETTLEMENT_RECORD {
        pubkey pool
        pubkey entrepreneur
        uint64 total_returned
        bool approved
    }
```

This diagram describes the intended logical relationships. The actual account fields and cardinalities must be reconciled with the Anchor implementation.

# 4. PDA Architecture

Program Derived Addresses provide deterministic account addresses controlled by the MoonCreditFi program.

## 4.1 PDA derivation flow

```mermaid
flowchart TD
    A["MoonCreditFi Program ID"]

    A --> B["Platform Config Seeds"]
    A --> C["Entrepreneur Seeds"]
    A --> D["Proposal Seeds"]
    A --> E["Pool Seeds"]
    A --> F["Investor Position Seeds"]
    A --> G["Settlement Seeds"]

    E --> H["Vault Authority Seeds"]
    H --> I["USDC Token Vault"]
```

## 4.2 Proposed PDA seeds

| Account | Seeds |
|---|---|
| PlatformConfig | `platform_config` |
| EntrepreneurProfile | `entrepreneur + entrepreneur_pubkey` |
| BusinessProposal | `proposal + entrepreneur_pubkey + proposal_id` |
| MudarabahPool | `pool + proposal_pubkey` |
| InvestorPosition | `position + pool_pubkey + investor_pubkey` |
| SettlementRecord | `settlement + pool_pubkey` |
| Vault Authority | `vault_authority + pool_pubkey` |

The actual program seed definitions are authoritative.

# 5. Entrepreneur Financing Flow

This flow explains how an entrepreneur moves from account authentication to an approved financing opportunity.

## 5.1 Entrepreneur journey

```mermaid
flowchart TD
    A["Entrepreneur Opens MoonCreditFi"]

    A --> B["Connect Solana Wallet"]
    B --> C["SIWS Authentication"]

    C --> D["Create Entrepreneur Profile"]

    D --> E["Create Business Profile"]

    E --> F["Submit Business Information"]
    F --> G["Upload Supporting Documents"]

    G --> H["Business Verification"]

    H --> I{"Business Verified?"}

    I -->|No| J["Request Corrections or Reject"]
    J --> E

    I -->|Yes| K["Create Financing Proposal"]

    K --> L["Submit Proposal"]

    L --> M["Authorized Review"]

    M --> N{"Proposal Approved?"}

    N -->|No| O["Proposal Rejected"]
    N -->|Yes| P["Create Mudarabah Pool"]

    P --> Q["Pool Opens for Contributions"]
```

## 5.2 Important separation

Business verification and financing proposal approval are separate operations.

A verified business is not automatically entitled to funding.

A financing proposal must undergo its own review and approval process.

# 6. Investor Funding Flow

This flow explains how an investor discovers a financing opportunity and contributes USDC.

## 6.1 Investor journey

```mermaid
flowchart TD
    A["Investor Opens MoonCreditFi"]

    A --> B["Connect Wallet"]
    B --> C["SIWS Authentication"]

    C --> D["Open Investor Dashboard"]

    D --> E["Explore Funding Opportunities"]

    E --> F["Select Mudarabah Pool"]

    F --> G["Review Business Information"]

    G --> H["Review Financing Terms"]

    H --> I{"Accept Terms?"}

    I -->|No| E

    I -->|Yes| J["Check USDC Balance"]

    J --> K{"Sufficient Balance?"}

    K -->|No| L["Display Insufficient Balance"]

    K -->|Yes| M["Prepare Contribution Transaction"]

    M --> N["Request Wallet Signature"]

    N --> O["Submit Transaction to Solana"]

    O --> P["Anchor Program Validates Transaction"]

    P --> Q["Transfer USDC to Pool Vault"]

    Q --> R["Update Investor Position"]

    R --> S["Update Pool Contribution Total"]

    S --> T["Display Confirmed Investment"]
```

## 6.2 Contribution architecture

```mermaid
flowchart LR
    A["Investor Wallet"] --> B["Investor USDC Account"]

    B -->|USDC Transfer| C["Pool USDC Vault"]

    D["MoonCreditFi Program"] --> E["Validate Contribution"]

    E --> C

    D --> F["Investor Position PDA"]

    D --> G["Mudarabah Pool PDA"]
```

Every contribution must be validated by the program. Frontend validation is only for user experience and is not a substitute for on-chain validation.

# 7. Mudarabah Pool Lifecycle

The Mudarabah pool manages the lifecycle of business financing.

## 7.1 State transition diagram

```mermaid
stateDiagram-v2
    [*] --> Open

    Open --> Funded: Funding target reached

    Open --> Cancelled: Authorized cancellation

    Funded --> Active: Capital disbursed

    Funded --> Cancelled: Permitted cancellation

    Active --> SettlementPending: Settlement submitted

    SettlementPending --> Settled: Settlement approved

    Settled --> [*]

    Cancelled --> [*]
```

## 7.2 Lifecycle explanation

| Status | Description |
|---|---|
| Open | Pool accepts investor contributions |
| Funded | Funding target reached |
| Active | Capital has been disbursed |
| SettlementPending | Settlement awaits approval |
| Settled | Settlement approved and distribution can proceed |
| Cancelled | Pool cancelled under permitted conditions |

The program must enforce valid transitions and prevent unauthorized status changes.

# 8. Capital Disbursement Flow

Once the pool has reached its funding target, the approved capital may be transferred to the entrepreneur.

## 8.1 Disbursement diagram

```mermaid
flowchart TD
    A["Funding Target Reached"]

    A --> B["Validate Pool State"]

    B --> C{"Pool Fully Funded?"}

    C -->|No| D["Reject Disbursement"]

    C -->|Yes| E["Authorize Pool Activation"]

    E --> F["Pool Becomes Active"]

    F --> G["Request Capital Disbursement"]

    G --> H["Validate Entrepreneur"]

    H --> I["Validate Vault Authority"]

    I --> J["Validate USDC Mint"]

    J --> K["Execute Token Transfer"]

    K --> L["Update Pool State"]

    L --> M["Emit Disbursement Event"]

    M --> N["Frontend Refreshes Pool Data"]
```

The contract must prevent a second disbursement for the same pool.

# 9. Settlement Architecture

After the entrepreneur completes the agreed business activity, settlement information is submitted for review.

The contract records the settlement and enforces the authorized approval process.

## 9.1 Settlement flow

```mermaid
flowchart TD
    A["Business Activity Completed"]

    A --> B["Entrepreneur Prepares Settlement"]

    B --> C["Submit Settlement Information"]

    C --> D["Create Settlement Record"]

    D --> E["Authorized Reviewer"]

    E --> F["Examine Supporting Evidence"]

    F --> G{"Settlement Approved?"}

    G -->|No| H["Request More Information"]

    H --> B

    G -->|Yes| I["Approve Settlement On-chain"]

    I --> J["Record Total Returned Amount"]

    J --> K["Calculate Profit or Loss"]

    K --> L["Enable Eligible Distributions"]
```

## 9.2 Off-chain and on-chain separation

| Operation | Location |
|---|---|
| Business financial documents | Private backend storage |
| Business evidence review | Off-chain review process |
| Settlement submission | Backend and blockchain as designed |
| Settlement approval | Authorized on-chain instruction |
| Distribution calculations | Smart contract |
| Token transfers | Solana |
| Transaction records | Solana |

An approved settlement means an authorized reviewer accepted the submitted information. It does not make the underlying business figures independently proven facts.

# 10. Profit and Loss Distribution Flow

The distribution process depends on the settlement outcome.

## 10.1 Profit scenario

```mermaid
flowchart TD
    A["Settlement Approved"]

    A --> B["Calculate Returned Capital"]

    B --> C["Calculate Actual Profit"]

    C --> D["Apply Agreed Profit-Sharing Ratio"]

    D --> E["Calculate Investor Profit Share"]

    D --> F["Calculate Entrepreneur Profit Share"]

    E --> G["Calculate Each Investor's Entitlement"]

    G --> H["Investor Claims Distribution"]

    H --> I["Verify Claim Eligibility"]

    I --> J["Transfer USDC"]

    F --> K["Check Investor Claim Accounting"]

    K --> L["Entrepreneur Claims Eligible Profit"]
```

## 10.2 Loss scenario

```mermaid
flowchart TD
    A["Settlement Approved"]

    A --> B["Calculate Actual Capital Returned"]

    B --> C["Determine Genuine Business Loss"]

    C --> D["Calculate Investor Capital Loss"]

    D --> E["Calculate Remaining Investor Entitlements"]

    E --> F["Distribute Available Funds Proportionally"]

    F --> G["Investors Claim Available Capital"]

    G --> H["Complete Pool Accounting"]
```

In a genuine Mudarabah business loss, investors generally bear the financial capital loss, while the entrepreneur loses their effort and expected profit. Negligence, misconduct, fraud, or breach must be addressed through an appropriate determination.

# 11. Complete End-to-End System Flow

The following diagram brings the entire MoonCreditFi financing lifecycle together.

```mermaid
flowchart TD
    START["User Enters MoonCreditFi"]

    START --> AUTH["Wallet Connection and SIWS"]

    AUTH --> ROLE{"User Role"}

    ROLE -->|Entrepreneur| ENT["Entrepreneur Onboarding"]
    ROLE -->|Investor| INV["Investor Onboarding"]

    ENT --> BUSINESS["Create Business Profile"]

    BUSINESS --> VERIFY["Business Verification"]

    VERIFY --> APPROVAL{"Business Verified?"}

    APPROVAL -->|No| CORRECTION["Correct Business Information"]

    CORRECTION --> BUSINESS

    APPROVAL -->|Yes| PROPOSAL["Create Financing Proposal"]

    PROPOSAL --> REVIEW["Proposal Review"]

    REVIEW --> APPROVED{"Proposal Approved?"}

    APPROVED -->|No| REJECT["Proposal Rejected"]

    APPROVED -->|Yes| CREATE["Create Mudarabah Pool"]

    CREATE --> OPEN["Open Pool"]

    INV --> EXPLORE["Explore Opportunities"]

    EXPLORE --> REVIEWPOOL["Review Pool and Terms"]

    OPEN --> REVIEWPOOL

    REVIEWPOOL --> CONTRIBUTE["Investor Contributes USDC"]

    CONTRIBUTE --> VAULT["USDC Enters Pool Vault"]

    VAULT --> FUNDING{"Funding Target Reached?"}

    FUNDING -->|No| OPEN

    FUNDING -->|Yes| ACTIVE["Activate Pool"]

    ACTIVE --> DISBURSE["Disburse Capital"]

    DISBURSE --> BUSINESSACT["Entrepreneur Manages Business"]

    BUSINESSACT --> SETTLEMENT["Submit Settlement"]

    SETTLEMENT --> SETTLE_REVIEW["Settlement Review"]

    SETTLE_REVIEW --> SETTLED{"Approved?"}

    SETTLED -->|No| SETTLEMENT

    SETTLED -->|Yes| CALC["Calculate Profit or Loss"]

    CALC --> DISTRIBUTE["Investor Distributions"]

    DISTRIBUTE --> ENTREPRENEUR["Eligible Entrepreneur Profit Claim"]

    ENTREPRENEUR --> COMPLETE["Pool Accounting Complete"]
```

# 12. Architecture Boundaries and Security

The following responsibilities must remain clearly separated.

| Responsibility | Authoritative layer |
|---|---|
| Wallet signature | Wallet cryptographic verification |
| Authentication session | Backend |
| User profile | Backend database |
| Business documents | Private backend storage |
| Business verification | Authorized review process |
| Reputation signals | CredLayer service |
| Pool financial state | Solana program |
| Investor position | Solana program |
| USDC vault balance | Solana token account |
| Capital disbursement | Solana program |
| Settlement review decision | Authorized reviewer |
| Distribution entitlement | Solana program |
| Transaction confirmation | Solana RPC and on-chain state |

## 12.1 Security principles

- Validate every signer.
- Validate every PDA.
- Validate token mint and token accounts.
- Use program-controlled vault authorities.
- Prevent duplicate claims.
- Prevent unauthorized withdrawals.
- Use checked arithmetic.
- Protect investor funds from arbitrary administrative withdrawal.
- Use multisig governance for privileged production operations.
- Maintain a clear audit trail.
- Do not trust frontend-submitted financial state.

# 13. Architecture Summary

MoonCreditFi separates its blockchain infrastructure into distinct responsibilities:

**Entrepreneur layer:** Business registration and financing proposals.

**Investor layer:** Opportunity discovery and USDC contributions.

**Smart contract layer:** Pool management, fund custody, capital disbursement, settlement authorization, and distributions.

**Backend layer:** Business information, verification, documents, reputation, and other off-chain services.

**Blockchain layer:** Verifiable financial transactions and program-enforced state transitions.

The result is a financing architecture in which off-chain business processes interact with transparent on-chain financial operations.


