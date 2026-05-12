# Bitcoin Pre-Release Source Code — Architecture Overview

> **Source:** Pre-release preview by Satoshi Nakamoto, copyright 2008.  
> **Files covered:** `main.h`, `main.cpp`, `node.h`, `node.cpp`  
> **Note:** This is an early prototype. Variable names, constants, and comments reflect a work in progress (e.g. the blockchain is still called the *"timechain"*).

---

## 1. High-Level Structure

The codebase is split into two clear concerns:

| Module | Files | Responsibility |
|--------|-------|---------------|
| **Core** | `main.h` / `main.cpp` | Transactions, blocks, the chain, mining, wallet, coin selection |
| **Network** | `node.h` / `node.cpp` | Peer connections, socket I/O, message framing, message routing |

Both modules share a thin set of global state variables declared in the headers and defined in the `.cpp` files.

---

## 2. Core Module (`main.h` / `main.cpp`)

### 2.1 Constants and Units

```cpp
static const int64 COIN = 1000000;     // 1 bitcoin in base units
static const int64 CENT = 10000;       // 0.01 bitcoin
static const int64 TRANSACTIONFEE = 1 * CENT;
static const unsigned int MINPROOFOFWORK = 20;  // marked "ridiculously easy for testing"
```

Bitcoin amounts are represented as 64-bit integers in the smallest unit (later called satoshis). The mining difficulty at this stage is intentionally set very low for development.

---

### 2.2 Transaction Data Model

Transactions are built from a hierarchy of classes:

```
COutPoint       — reference to a specific output of a prior transaction (hash + index)
CInPoint        — in-memory pointer to a transaction + input index
CTxIn           — one transaction input: a COutPoint + a scriptSig (unlocking script)
CTxOut          — one transaction output: an amount (nValue) + a scriptPubKey (locking script)
CTransaction    — a complete transaction: vector<CTxIn> + vector<CTxOut> + nLockTime
```

**`CDiskTxPos`** is a lightweight on-disk locator storing `(nFile, nBlockPos, nTxPos)` — essentially a file offset triple used to quickly find a transaction on disk without loading the whole block.

#### Key transaction methods

- `CheckTransaction()` — basic sanity checks (no negative outputs, coinbase size limit, all inputs non-null for non-coinbase).
- `AcceptTransaction()` — validates against in-memory pool and previous outputs; inserts into `mapTransactions`.
- `ConnectInputs()` — marks referenced outputs as spent, verifies signatures, accumulates fees.
- `DisconnectInputs()` — reverses ConnectInputs; used during chain reorganization.
- `IsCoinBase()` — true when the transaction has exactly one null-prevout input (the block reward).

---

### 2.3 Wallet Transaction Hierarchy

```
CTransaction
  └── CMerkleTx       — adds a Merkle branch linking the tx to a block (hashBlock, vMerkleBranch)
        └── CWalletTx — adds wallet metadata: prior supporting txes, order info, fFromMe, fSpent
```

`CWalletTx::AddSupportingTransactions()` walks back up to 3 hops through the input chain and bundles those ancestor transactions together, so a lightweight client can verify a payment without holding the full chain.

---

### 2.4 Block Structure

```
CBlock
  ├── Header: hashPrevBlock, hashMerkleRoot, nTime, nBits, nNonce
  └── Body:   vector<CTransaction> vtx
```

The **Merkle tree** is built over transaction hashes (`BuildMerkleTree()`). The block hash is computed over only the 80-byte header. Proof-of-work requires this hash to be numerically less than `(~uint256(0) >> nBits)`.

**`CBlockIndex`** is the in-memory index node for one block, forming a doubly-linked list (`pprev` / `pnext`) that represents the main chain. Multiple `pprev` pointers can point to the same block (forks), but `pnext` only follows the longest branch.

**`CBlockLocator`** is used during peer sync. It encodes a sparse list of recent block hashes (exponentially spaced further back in time) so two nodes can quickly find their common ancestor even if they diverged long ago.

---

### 2.5 Global State

```cpp
map<uint256, CBlockIndex*> mapBlockIndex;      // all known blocks
map<uint256, CTransaction> mapTransactions;    // unconfirmed tx mempool
map<uint256, CWalletTx>    mapWallet;          // user's own transactions
map<uint256, CBlock*>      mapOrphanBlocks;    // blocks whose parent is unknown
map<vector<unsigned char>, CPrivKey> mapKeys;  // private keys
map<uint160, vector<unsigned char>> mapPubKeys;// public keys indexed by hash
```

These maps are protected by matching `CCriticalSection` mutex objects (`cs_mapWallet`, `cs_mapKeys`, etc.).

---

### 2.6 Block Acceptance and Chain Reorganization

When a new block arrives (`ProcessBlock`):

1. Preliminary `CheckBlock()` — size limits, timestamp, proof-of-work, Merkle root.
2. If the parent block is unknown, the block is shelved in `mapOrphanBlocks` and the peer is asked for the missing chain segment.
3. `AcceptBlock()` — verifies proof-of-work difficulty matches `GetNextWorkRequired()`, validates all transaction inputs, writes to disk, calls `AddToBlockIndex()`.
4. If the new block extends the main chain, it simply connects. If it creates a longer fork, `Reorganize()` is called: it disconnects blocks back to the common ancestor, then connects the new branch, validating both directions in a test run first before committing to disk.

**Difficulty retargeting** (`GetNextWorkRequired`) looks back over a 30-day window (at a 15-minute target block time) and adjusts `nBits` by ±1 if the actual elapsed time is more than 2× or less than ½ of the target.

**Block reward** (`GetBlockValue`) starts at 10 000 CENT and halves every 100 000 blocks.

---

### 2.7 Message Handling

`ProcessMessage()` dispatches incoming peer messages by command string:

| Command | Action |
|---------|--------|
| `version` | Exchange protocol version, decide if peer is a full node or client |
| `addr` | Store new peer addresses, relay to other nodes |
| `inv` | Receive inventory announcements (tx/block hashes); request unknowns |
| `getdata` | Serve a block or relayed object from memory |
| `getblocks` | Serve a range of blocks starting from a locator |
| `tx` | Receive and relay a single unconfirmed transaction |
| `block` | Receive and process a new block |
| `getaddr` | Return recently seen peer addresses |
| `checkorder` / `submitorder` | Prototype payment order flow (pre-standard) |
| `reply` | Callback dispatch for pending request trackers |

---

### 2.8 Bitcoin Miner

`BitcoinMiner()` runs in a dedicated thread and:

1. Creates a **coinbase transaction** paying the block reward to `keyUser`.
2. Collects valid, final, non-coinbase transactions from `mapTransactions` into the block, ordering them so dependencies come first.
3. Sets `hashMerkleRoot`, `nBits` (from `GetNextWorkRequired`), and `nTime`.
4. Pre-formats the 80-byte header into a SHA-256 padding buffer for efficiency.
5. Iterates `nNonce` from 1, computing `SHA256(SHA256(header))` via `BlockSHA256()` using Crypto++ at each step.
6. When `hash <= hashTarget`, the block is found. It calls `ProcessBlock(NULL, pblock)` — treating the self-mined block exactly like one received from a peer.

---

### 2.9 Wallet Actions

- `CountMoney()` — sums all unspent, final wallet outputs.
- `SelectCoins()` — stochastic subset-sum solver (1 000 random trials) to find a minimal coin set covering the target amount.
- `CreateTransaction()` — builds a new transaction, signs inputs with `SignSignature()`, adds a change output back to the sender if needed.
- `SendMoney()` — calls `CreateTransaction`, then `AcceptTransaction` and `RelayWalletTransaction` to broadcast.

---

## 3. Network Module (`node.h` / `node.cpp`)

### 3.1 Message Wire Format

Every message on the wire follows this layout:

```
[4 bytes]  pchMessageStart  = { 0xf9, 0xbe, 0xb4, 0xd9 }
[12 bytes] command string   (null-padded)
[4 bytes]  payload size
[N bytes]  payload
```

The magic bytes are chosen to be invalid UTF-8 and uncommon in normal data, making framing errors easy to detect. `CMessageHeader` encapsulates parsing and validation of this header.

---

### 3.2 Address and Inventory Types

**`CAddress`** stores an IPv4-mapped address (12-byte reserved prefix + 4-byte IP + 2-byte port) plus a `nServices` bitfield and a timestamp. Default port is `2222`.

**`CInv`** (inventory vector) identifies a network object by type and hash. Defined types:

```cpp
MSG_TX      = 1   // unconfirmed transaction
MSG_BLOCK   = 2   // block
MSG_REVIEW  = 3   // (prototype marketplace feature)
MSG_PRODUCT = 4   // (prototype marketplace feature)
MSG_TABLE   = 5   // (prototype marketplace feature)
```

---

### 3.3 The `CNode` Class

`CNode` represents one open TCP connection. Key members:

| Member | Purpose |
|--------|---------|
| `hSocket` | Raw OS socket handle |
| `vSend` / `vRecv` | Outgoing / incoming byte stream buffers (`CDataStream`) |
| `nVersion` | Negotiated protocol version |
| `fClient` | True if the peer does not serve the full block history |
| `fInbound` | True if the connection was accepted (not initiated) |
| `setInventoryKnown` | Inventory items known to this peer (to avoid re-sending) |
| `vInventoryToSend` | Inventory to announce on next send pass |
| `mapAskFor` | Priority queue of objects to request (rate-limited to avoid hammering) |
| `mapRequests` | Pending request callbacks keyed by random nonce |
| `vfSubscribe` | 256-channel subscription bitmap (prototype pub/sub) |

`PushMessage()` is a variadic template that serializes arguments directly into `vSend` within a critical section, framing them with `BeginMessage` / `EndMessage`.

---

### 3.4 Threading Model

`StartNode()` launches three background threads via `_beginthread` (Win32):

**Thread 0 — `ThreadSocketHandler`**  
Runs a `select()` loop over all open sockets.  
- Accepts new inbound connections on the listening socket.  
- Reads available data into each node's `vRecv` buffer.  
- Drains each node's `vSend` buffer onto the wire.  
- Detects disconnections and cleans up `CNode` objects.

**Thread 1 — `ThreadOpenConnections`**  
Maintains outbound connectivity (target: 5 peers).  
- Picks a random class-C subnet from `mapAddresses`, then a random IP within it — a deliberate strategy to resist address-flooding attacks where an attacker floods many IPs in the same subnet.  
- Calls `ConnectNode()`, advertises the local address, and requests more addresses (`getaddr`).

**Thread 2 — `ThreadMessageHandler`**  
Processes application-layer messages.  
- For every connected node, calls `ProcessMessages()` (dispatches to `ProcessMessage`) and then `SendMessages()` (flushes inventory announcements and pending data requests).  
- Sleeps 200 ms between passes to allow messages to batch up.

**Thread 3 — `ThreadBitcoinMiner`** (optional, controlled by `fGenerateBitcoins`)  
Wraps `BitcoinMiner()` from `main.cpp`.

---

### 3.5 Relay System

When a transaction or block is received and accepted, it is inserted into `mapRelay` (keyed by `CInv`) and its hash is added to every connected peer's `vInventoryToSend`. On the next send pass, peers receive an `inv` announcement; they request any items they don't already have with `getdata`; the sender then serves from `mapRelay`. Relay entries expire after 10 minutes.

---

## 4. Data Flow Summary

```
[Miner thread]
    BitcoinMiner()
        → builds CBlock
        → solves SHA256 proof-of-work
        → ProcessBlock(NULL, block)
              → CheckBlock()
              → AcceptBlock()
                    → ConnectInputs() for each tx
                    → AddToBlockIndex()
                    → RelayInventory()   ──────────────────────────────┐
                                                                       │
[Network thread]                                                       │
    ThreadSocketHandler                                                │
        recv() → vRecv                                                 │
                                                                       │
    ThreadMessageHandler                                               │
        ProcessMessage("tx")   → AcceptTransaction() → mapTransactions │
        ProcessMessage("block")→ ProcessBlock()      → AddToBlockIndex()
                                                                       │
        SendMessages()                                                 │
            → flush "inv" to peers  ←─────────────────────────────────┘
            → flush "getdata" requests
            → flush "addr" peer lists
```

---

## 5. Notable Pre-Release Details

- The blockchain is called the **"timechain"** throughout the code (e.g. `hashTimeChainBest`, `PrintTimechain()`).
- The genesis block is hard-coded with timestamp `1221069728` (Unix epoch for 11 Sep 2008) and nonce `141755`.
- `MINPROOFOFWORK = 20` is explicitly commented as *"ridiculously easy for testing"*; production was planned at `40`.
- `TRANSACTIONFEE` is noted with a `/// change this to a user options setting, optional fee can be zero` comment — fees were not yet mandatory.
- The code has vestiges of a **marketplace prototype** (`MSG_PRODUCT`, `MSG_TABLE`, `CProduct`, `CTable`, `checkorder`/`submitorder`) that never made it into the public release.
- Block files are stored as `blk0001.dat`, `blk0002.dat`, … capped at ~2 GB each (FAT32 file size constraint is explicitly noted in a comment).
- The Windows-specific WinSock2 API is used directly, confirming this was written primarily for Windows.
