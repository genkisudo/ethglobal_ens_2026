# ethglobal_ens_2026

# Across + ENS — Financial Identity and Compliance-Gated Payments for AI Agents

## 1. Project summary

**Working name:** AgentPay / AgentDNS / ENS Agent Pay

**One-line pitch**

> ENS gives agents a financial identity. Across makes payments chain-agnostic. A programmable compliance gate controls what reaches the agent.

The project turns an ENS name into a payment endpoint for an AI agent.

Example:

```text
research-agent.eth
```

The ENS profile defines:

* the agent wallet
* preferred settlement chain
* preferred settlement token
* optional service metadata

A user or another agent can then pay:

```text
20 USDC to research-agent.eth
```

without needing to know where the agent wallet lives or which chain/token it prefers.

Example:

```text
Sender
ETH on Arbitrum
      ↓
Across Swap API
      ↓
20 USDC on Base
      ↓
research-agent.eth
```

---

# 2. Core thesis

AI agents increasingly need persistent identities and wallets, but wallet addresses are poor interfaces for agent-to-agent commerce.

ENS can provide a stable, human-readable financial identity.

Across can translate the sender's current assets into the agent's preferred settlement state.

Conceptually:

```text
research-agent.eth
      ↓ ENS
agent wallet = 0xAgent
preferredChain = Base
preferredToken = USDC
      ↓ Across
sender state → agent settlement state
```

The sender specifies:

```text
who + amount
```

The agent specifies:

```text
how it wants to receive funds
```

Across handles execution.

A second payment mode adds a **compliance-gated deposit endpoint**:

```text
originator
    ↓
Compliance Deposit Gateway
    ↓ screen originator
approved ─────────→ Across ─────────→ agent
flagged  ─────────→ refund on origin
```

This gate is application infrastructure, not a native Across sanctions/compliance feature.

Across persistent deposit addresses are permanent, amount-agnostic addresses that automatically sweep supported deposits to their configured destination. Because that sweep is automatic, a compliance decision must happen **before** funds are forwarded to an Across deposit address.

---

# 3. MVP goal

Build one complete payment flow:

1. Agent owner connects a wallet.
2. Owner selects an ENS name for the agent.
3. Owner configures:

   * destination chain
   * destination token
   * optional agent metadata
4. Configuration is stored in ENS text records.
5. A sender enters the agent's ENS name.
6. App resolves the agent wallet and settlement preference.
7. Sender enters an amount.
8. Sender chooses an origin chain/token.
9. Across returns an `exactOutput` quote.
10. Sender signs.
11. Agent receives the exact requested amount on its preferred chain.

Add one compliance-gated deposit flow:

12. Agent owner creates a programmable deposit endpoint.
13. A sender deposits supported funds to the gateway.
14. The gateway records the sender, token, amount, and intended agent.
15. A screening service classifies the originator as `approved` or `flagged`.
16. Approved deposits are forwarded into an Across transfer.
17. Flagged deposits are refunded on the origin chain and never forwarded to the agent.

For the hackathon, support one origin chain and one token for this second flow. The screening adapter can be mocked or backed by a real compliance API without changing the core architecture.

---

# 4. Demo scenario

### Agent

ENS:

```text
research-agent.eth
```

Settlement preference:

```text
Token: USDC
Chain: Base
```

Wallet:

```text
0xAgent
```

Optional metadata:

```text
Service: research
Type: AI agent
```

### Sender

Sender holds:

```text
ETH on Arbitrum
```

The user enters:

```text
research-agent.eth
```

The app resolves:

```text
research-agent.eth

AI research agent
Receives USDC on Base
0xAge...123
```

The user enters:

```text
20 USDC
```

Then selects:

```text
Pay with ETH on Arbitrum
```

Across computes:

```text
ETH / Arbitrum
        ↓
USDC / Base
```

with:

```text
recipient = 0xAgent
tradeType = exactOutput
outputAmount = 20 USDC
```

The UI shows:

```text
Agent receives

20.00 USDC
Base

You pay

≈ 0.005X ETH
Arbitrum

[ Pay research-agent.eth ]
```

The sender signs and the agent receives exactly 20 USDC.

---

# 5. ENS data model

Do not deploy a custom registry contract for the MVP.

Use ENS text records.

Recommended record:

```text
com.agentpay.profile
```

Example:

```json
{
  "version": 1,
  "type": "agent",
  "service": "research",
  "chainId": 8453,
  "token": "0xUSDC_ADDRESS"
}
```

The ENS address record resolves the recipient wallet.

The text record defines the preferred settlement configuration.

For v1, support only one destination chain/token.

---

# 6. Recipient resolution

The app resolves two things from ENS:

```text
1. agent wallet address
2. payment preference
```

Flow:

```text
resolve research-agent.eth
        ↓
address → 0xAgent
        ↓
com.agentpay.profile
        ↓
USDC / Base
```

If a chain-specific address exists, use it.

Otherwise, use the resolved EVM address.

---

# 7. Agent setup page

Route:

```text
/setup
```

UI:

```text
Configure research-agent.eth

Agent type
[ AI agent ]

Service
[ Research ]

Preferred token
[ USDC ▼ ]

Preferred network
[ Base ▼ ]

Wallet
0xAgent...

[ Save to ENS ]
```

Saving updates the ENS text record.

The owner signs the ENS transaction.

No project-owned identity database is required.

---

# 8. Payment flow

Main page:

```text
Pay an agent

Recipient
[ research-agent.eth ]

Amount
[ 20 ] [ USDC ]

[ Continue ]
```

The app resolves:

```text
research-agent.eth
```

and displays:

```text
AI research agent

Receives:
USDC on Base

Wallet:
0xAge...123
```

The sender then chooses:

```text
Pay from

Network
[ Arbitrum ▼ ]

Asset
[ ETH ▼ ]
```

Do not build automatic wallet-wide route discovery for the MVP.

---

# 9. Across integration

Use the Across Swap API.

The core trade mode is:

```text
tradeType=exactOutput
```

This is important because agent payments should be destination-defined.

Example:

```text
Pay the agent exactly 20 USDC.
```

rather than:

```text
Spend 0.005 ETH and deliver whatever remains.
```

Conceptual request:

```text
tradeType=exactOutput

amount=20000000

originChainId=42161
inputToken=<ETH>

destinationChainId=8453
outputToken=<USDC>

depositor=0xSender
recipient=0xAgent
```

Use:

```text
strictTradeType=true
```

when exact settlement semantics are required.

---

# 10. Quote UI

Example:

```text
Pay research-agent.eth

Agent receives

20.00 USDC
Base

You pay

0.0052 ETH
Arbitrum

Estimated fees
$0.17

Recipient
0xAge...123

[ Confirm payment ]
```

The destination amount should be visually dominant.

The interface should emphasize:

```text
Agent receives 20 USDC
```

not the bridge mechanics.

---

# 11. Transaction execution

Conceptual flow:

```text
Across quote
    ↓
ERC-20 approval if required
    ↓
Across transaction
    ↓
wallet confirmation
```

For native ETH, send the Across transaction directly.

For ERC-20 input tokens, approve first if needed.

---

# 12. Success state

After execution:

```text
Payment sent ✓

20.00 USDC
to research-agent.eth

Arbitrum
   ↓
Across
   ↓
Base

Recipient
0xAgent
```

Optionally poll until:

```text
Delivered ✓
```

Final state:

```text
research-agent.eth received
20 USDC on Base
```

---

# 13. Compliance-gated Across deposit endpoint

## Goal

Give an agent a reusable payment address that anyone can fund, while ensuring only approved originators are forwarded through Across.

Example:

```text
research-agent.eth
Compliance deposit endpoint:
0xGateway...

Accepted:
USDC on Arbitrum
```

A sender transfers USDC to `0xGateway...`.

The system evaluates the **actual funding address** before initiating an Across transfer.

### Approved deposit

```text
0xSender
   ↓ USDC
ComplianceDepositGateway
   ↓ screening = approved
Across
   ↓
research-agent.eth
```

### Flagged deposit

```text
0xFlagged
   ↓ USDC
ComplianceDepositGateway
   ↓ screening = flagged
refund
   ↓
0xFlagged
```

The flagged funds never reach the final agent destination.

## Why the gate must sit before Across

Across persistent deposit addresses are permanent and automatically sweep supported deposits. A sender can transfer supported USDC to the address without requesting a new quote.

Therefore, screening must occur **before** funds are forwarded to Across:

```text
sender
  ↓
programmable gateway
  ↓ screening
Across
  ↓
agent
```

Do not rely on:

```text
sender
  ↓
Across deposit address
  ↓ screening
agent
```

because the application does not control the automatic sweep once funds reach the Across deposit address.

## Gateway design

For the hackathon, deploy one minimal origin-chain contract:

```text
ComplianceDepositGateway
```

Responsibilities:

```text
deposit(agentId, token, amount)
record depositor
escrow funds
emit DepositReceived
allow approved forwarding
allow flagged refund
```

Conceptual state:

```solidity
enum Status {
    Pending,
    Approved,
    Forwarded,
    Flagged,
    Refunded
}
```

Each deposit stores:

```text
depositId
depositor
token
amount
agent ENS name or resolved recipient
timestamp
status
```

The gateway does **not** determine whether an address is sanctioned or risky. A separate screening adapter makes that decision.

## Screening flow

```text
DepositReceived
      ↓
backend watcher
      ↓
ComplianceProvider.screen(depositor)
      ↓
approved / flagged
```

Use a small provider interface:

```ts
interface ComplianceProvider {
  screen(address: `0x${string}`): Promise<{
    status: "approved" | "flagged"
    reason?: string
  }>
}
```

This keeps the architecture independent of a specific vendor.

For the hackathon, use either:

* a mocked denylist for the demo, or
* a real compliance API if available.

Do not claim that Across itself performs sanctions screening.

## Approved forwarding

After approval, the backend obtains an Across route and authorizes the gateway to forward the funds.

```text
Gateway
   ↓
Across
   ↓
agent's preferred chain/token
```

The MVP should support one known-good route rather than arbitrary tokens and chains.

## Persistent deposit address option

Across also exposes an early-access Persistent Deposit Address API.

A persistent Across deposit address:

* is reusable
* has no amount embedded in it
* automatically sweeps supported deposits
* currently supports USDC destinations on HyperCore and HyperEVM
* returns supported origin inputs and route limits

For a compatible route, the compliance gateway can forward an **approved** deposit to the Across persistent deposit address:

```text
sender
   ↓
ComplianceDepositGateway
   ↓ approved
Across persistent deposit address
   ↓ automatic sweep
agent destination
```

The gateway remains necessary because a direct transfer to the Across address would bypass application-level screening.

## Refund behavior

Flagged deposits should be refunded directly from the gateway on the **origin chain**:

```text
flagged
   ↓
gateway refund
   ↓
original depositor
```

Do not intentionally send flagged deposits into Across and depend on an Across refund.

Across supports `refundAddress` and `refundOnOrigin` for transfers that have already entered its lifecycle, but those parameters are intended for failed or expired transfers rather than pre-transfer compliance rejection.

## ENS integration

Extend the agent profile with an optional compliance configuration:

```json
{
  "version": 2,
  "type": "agent",
  "service": "research",
  "chainId": 8453,
  "token": "0xUSDC_ADDRESS",
  "compliance": {
    "required": true,
    "gatewayChainId": 42161,
    "gateway": "0xGATEWAY_ADDRESS"
  }
}
```

Resolution becomes:

```text
research-agent.eth
        ↓ ENS
agent wallet
settlement preference
compliance policy
        ↓
if compliance.required
        ↓
show gated payment route
```

For the hackathon UI, expose:

```text
Direct payment
Compliance-gated deposit
```

If an agent sets `compliance.required = true`, default to the gated path.

---

# 14. Architecture

Keep the system small.

```text
┌─────────────────────────────┐
│          Next.js            │
│                             │
│ Agent setup                 │
│ ENS resolution              │
│ Payment UI                  │
│ Compliance status           │
│ Transaction status          │
└─────────────┬───────────────┘
              │
       ┌──────┴──────┐
       │             │
       ▼             ▼
      ENS         Next.js API
                      │
              ┌───────┴────────┐
              ▼                ▼
      Compliance adapter   Across APIs
              │
              ▼
   ComplianceDepositGateway
```

Recommended stack:

```text
Frontend
Next.js
TypeScript
React

Ethereum
viem
wagmi

ENS
viem ENS resolution

Crosschain
Across Swap API

Backend
Next.js API routes
```

No database required.

One minimal `ComplianceDepositGateway` contract is required for the gated-deposit feature.

No indexer required.

---

# 15. Backend API

One endpoint is enough.

### POST `/api/quote`

Input:

```json
{
  "originChainId": 42161,
  "inputToken": "0x...",
  "destinationChainId": 8453,
  "outputToken": "0x...",
  "outputAmount": "20000000",
  "depositor": "0xSender",
  "recipient": "0xAgent"
}
```

Return:

```json
{
  "inputAmount": "...",
  "outputAmount": "20000000",
  "fees": "...",
  "approvalTxns": [],
  "swapTx": {},
  "estimatedFillTime": "..."
}
```

Keep the Across API key server-side.

Additional endpoints for the gated flow:

```text
POST /api/compliance/screen
GET  /api/deposits/:id
POST /api/deposits/:id/forward
POST /api/deposits/:id/refund
```

For production, forwarding and refund authorization require a hardened operator/security model. The hackathon MVP can use a single trusted operator key.

---

# 16. Security checks

Before allowing payment:

### Validate ENS configuration

Verify:

* supported chain
* supported token
* Across route exists
* profile JSON is valid

### Show identity and wallet

Always display:

```text
research-agent.eth
0xAge...123
```

before signature.

### Show exact destination

Display:

```text
20 USDC
Base
```

clearly.

### Resolve fresh data

Do not permanently cache ENS payment preferences.

Future payments should use the current ENS configuration.

---

# 17. Important limitation

ENS does not prove that an address is autonomously controlled by an AI agent.

For the MVP, ENS provides:

* persistent naming
* wallet resolution
* payment preferences
* optional agent metadata

It should be presented as an **agent financial identity layer**, not an autonomous-agent verification system.

---

# 18. Failure states

### ENS does not resolve

```text
Couldn't resolve research-agent.eth.
```

### No agent payment profile

```text
research-agent.eth has no payment profile.
```

### Unsupported route

```text
This agent's settlement preference is not currently supported by Across.
```

### Exact output unavailable

```text
Unable to guarantee 20 USDC at the moment.
Try again.
```

---

# 19. What not to build

Do not add:

```text
❌ agent marketplace
❌ agent discovery
❌ reputation system
❌ subscriptions
❌ escrow
❌ automatic spending
❌ agent-to-agent messaging
❌ custom ENS resolver
❌ additional protocol contracts beyond the minimal compliance gateway
❌ transaction history database
❌ autonomous treasury management
❌ multi-agent orchestration
```

These distract from the core primitive.

---

# 20. Pages

Only three pages are necessary.

### `/`

```text
Pay an agent

research-agent.eth

20 USDC

[ Continue ]
```

### `/setup`

```text
Configure research-agent.eth

AI research agent
USDC
Base

[ Save to ENS ]
```

### `/tx/:id`

```text
20 USDC → research-agent.eth

Arbitrum → Base

Delivered ✓
```

---

# 21. MVP acceptance criteria

The MVP is complete when:

* [ ] Connect wallet.
* [ ] Resolve an ENS name.
* [ ] Resolve the agent wallet.
* [ ] Read the agent payment profile from ENS.
* [ ] Display agent metadata.
* [ ] Display destination chain/token.
* [ ] Sender selects origin chain/token.
* [ ] Sender enters payment amount.
* [ ] Request an Across `exactOutput` quote.
* [ ] Display required input amount.
* [ ] Execute the Across transaction.
* [ ] Agent receives the requested asset.
* [ ] Display successful delivery.
* [ ] Accept a USDC deposit into `ComplianceDepositGateway`.
* [ ] Capture the actual originator address from the onchain deposit.
* [ ] Screen the originator through the compliance adapter.
* [ ] Forward an approved deposit through Across.
* [ ] Refund a flagged deposit on the origin chain.
* [ ] Demonstrate that a flagged deposit never reaches the agent destination.

Everything else is optional.

---

# 22. Demo script

### 0–10 sec

Show:

```text
research-agent.eth

AI research agent

Preferred payment
USDC · Base
```

Explain:

> This agent has a persistent financial identity in ENS. It defines its wallet and how it prefers to receive payments.

### 10–20 sec

Switch to the sender.

The sender holds:

```text
ETH · Arbitrum
```

Enter:

```text
research-agent.eth
20 USDC
```

The app displays:

```text
Agent receives
20 USDC · Base

You pay
0.005X ETH · Arbitrum
```

### 20–35 sec

Click:

```text
Pay research-agent.eth
```

Sign.

### 35–45 sec

Show:

```text
Delivered ✓

20 USDC → research-agent.eth
Base
```

Close with:

> ENS gives the agent a persistent financial identity. Across converts any supported sender state into the agent's preferred settlement state.

---

# 23. Why ENS is essential

Without ENS, paying an agent requires knowing:

```text
wallet address
destination chain
destination token
```

With ENS:

```text
research-agent.eth
```

is enough.

The agent owner controls the payment configuration directly.

Applications do not need their own identity or settlement-preference database.

---

# 24. Why Across is essential

Across handles:

```text
source asset
source chain
swap
bridge
destination settlement
```

The division of responsibility is clean:

```text
ENS
WHO THE AGENT IS
+
HOW IT WANTS TO BE PAID

Across
HOW TO DELIVER THE PAYMENT
```

---

# 25. Stretch goal — agent payment links

Support:

```text
agentpay.xyz/research-agent.eth
```

or:

```text
agentpay.xyz/research-agent.eth?amount=20
```

An agent can publish this link in:

* its website
* API docs
* GitHub profile
* chat interface
* agent registry

The link contains the identity.

ENS contains the settlement preference.

Across executes the payment.

---

# 26. Stretch goal — service metadata

Extend the ENS profile:

```json
{
  "version": 2,
  "type": "agent",
  "service": "research",
  "description": "Long-form crypto research",
  "chainId": 8453,
  "token": "0xUSDC_ADDRESS"
}
```

This allows payment interfaces to show basic context without introducing a separate database.

Do not turn this into a marketplace in v1.

---

# 27. Long-term direction

The broader idea is to make ENS names behave like financial endpoints for software agents.

Today:

```text
address → wallet location
```

ENS:

```text
name → identity
```

This project:

```text
agent name → identity + desired financial state
```

Across then performs:

```text
sender's current state
        ↓
agent's desired state
```

That creates a useful primitive for future agent-to-agent commerce.

---

# 28. Final project description

**AgentPay turns ENS names into chain-agnostic financial identities for AI agents.**

An agent owner publishes the agent's wallet, preferred settlement network, and token through ENS. A sender only needs the agent's ENS name and the amount to pay.

The application resolves the agent's financial profile and uses the Across Swap API to transform the sender's available asset into the agent's preferred settlement asset on the preferred chain.

For example, `research-agent.eth` can request USDC on Base while the sender pays with ETH on Arbitrum. Across handles the crosschain swap and the agent receives the exact requested amount.

The sender does not manage the route.

The agent declares the desired financial state.

For agents that require screened funding, the project also exposes a programmable compliance deposit gateway. Funds are escrowed on the origin chain, the actual originator is screened, approved deposits are forwarded through Across, and flagged deposits are refunded before they reach the agent.

This compliance layer is implemented by the application; it should not be represented as a native Across sanctions-screening feature.

**ENS gives agents a financial identity. Across makes payments chain-agnostic. The gateway controls what reaches the agent.**

---

## Implementation note

Use **`exactOutput`** for the primary flow so the agent receives an exact amount such as 20 USDC.

Do not use Across Embedded Actions in v1. A standard Swap API payment to the ENS-resolved agent wallet is enough. Embedded Actions become relevant later if an agent wants incoming payments automatically routed into another onchain action.

## Across implementation constraints

As of September 2026, Across persistent deposit addresses are early access and currently target USDC-SPOT on HyperCore or USDC on HyperEVM. The API returns supported origin chains/tokens, limits, fees, and indicative fill times for each generated address.

For the general ENS-agent payment flow, continue using the Swap API. Use the persistent deposit-address API only when its supported destination fits the demo.

Across supports explicit refund configuration for Swap API routes through `refundAddress` and `refundOnOrigin`. Protocol refunds may take hours because expired deposits pass through Across's refund settlement process. Compliance rejections should therefore be refunded from the gateway before an Across transfer is initiated.

