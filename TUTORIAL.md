# Shielded Chips Tutorial

This tutorial walks you through writing the two Compact smart contracts at the
heart of this example:

1. `chips.compact` — a shielded token that the house mints and hands out to
   players. The chip's value lives inside a zswap UTXO, so how many chips
   someone holds is private.
2. `roulette.compact` — a RED/BLACK betting game that takes chip coins as
   bets and pays winners 2×.

Everything else in the repo is already provided. The tutorial focuses on
the shielded-token operations (minting, sending, receiving, and custodying
coins) and stays deliberately brief on the parts that aren't token-specific
(enum plumbing, the roulette-wheel color table, identity hashing).

---

## Prerequisites

Before you begin this tutorial, ensure you have:

- [Installed the toolchain](https://docs.midnight.network/getting-started/installation)
- A Compact compiler that supports `pragma language_version 0.23` — run
  `compact check` to see what you have, and `compact update` to get the latest
- Node.js v22+
- Docker, with the engine running — `compose.yml` provides the local devnet
- Completed a beginner tutorial (optional)

You do **not** need to start a proof server yourself. `yarn env:up` brings up
the node, the indexer, and a proof server together, and `src/config.ts` already
points at them.

---

## 0. What you start with, and what you'll build

Clone the starter repository:

```bash
git clone git@github.com:nstanford5/shielded-chips.git
```

After cloning, you already have the full TypeScript scaffolding:

```
src/
  config.ts        # network endpoints
  providers.ts     # wires wallet + proof server + indexer into a provider set
  wallet.ts        # MidnightWalletProvider, wallet sync helper
  test/roulette.test.ts   # the end-to-end game, driven through the SDK
contract/
  index.ts         # exports the compiled contracts to TS (CompiledContract.make)
  witnesses.ts     # the localSecretKey witness, shared by both contracts
compose.yml        # local devnet (node + indexer + proof server)
package.json       # `yarn compile`, `yarn validate`, etc.
```

The two things you write in this tutorial are:

- `contract/chips.compact`
- `contract/roulette.compact`

and then you run `yarn compile`, which generates the missing
`contract/managed/` directory with compiled circuits, proving/verifying keys, and
the TypeScript type definitions the scaffolding imports. 

The mental model for the two contracts:

```
  chips.compact                          roulette.compact
  ─────────────                          ────────────────
  house mints a shielded token   ──────► players bet chip COINS
  color derived from            tokenColor      (custodied by the contract)
  (contract addr + domain sep)   ──────► winners get 2× back via sendShielded
```

The roulette contract never mints chips. It only ever receives, holds,
and sends chip coins that the chips contract created — which is exactly what
makes the shielded-token plumbing the interesting part.

---

## Part 1 — `chips.compact`: minting a shielded token

Create the chips contract from the repo root:
```bash
touch contract/chips.compact
```

### 1.1 Header and ledger state

```compact
pragma language_version 0.23;
import CompactStandardLibrary;

// theHouse is the dapp pubkey of whoever deploys this contract.
export sealed ledger theHouse: Bytes<32>;
export ledger tokenColor: Bytes<32>;
export ledger totalMinted: Counter;
export ledger nonceCounter: Counter;
export ledger nonceSeed: Bytes<32>;

witness localSecretKey(): Bytes<32>;
```

Only two of these fields are about tokens, and they matter a lot later:

- `tokenColor` — the 32-byte color (a.k.a. token type) of the chip. Every
  shielded coin carries a `color` field, and a coin's color is derived
  deterministically from the minting contract's address + a domain
  separator. We record it here on the first mint so other contracts (the
  roulette game) can read it off the public ledger and bind to it.
- `nonceSeed`/ `nonceCounter` — the machinery for producing a fresh,
  unpredictable nonce for every coin we mint. More on why in §1.3.

`theHouse` is a `sealed` ledger field (set once, in the constructor, never
writable again) holding the deployer's identity. `localSecretKey` is a witness
— a private input supplied off-chain by the caller's wallet — used to derive
that identity.

### 1.2 Constructor: house identity + nonce seed

```compact
constructor() {
    const _sk = localSecretKey();
    const houseKey = getDappPublicKey(_sk);
    theHouse = disclose(houseKey);
    nonceSeed = disclose(
        persistentHash<Vector<2, Bytes<32>>>([pad(32, "roulette:chip-nonce:"), disclose(houseKey)])
    );
}
```

Identity derivation (`getDappPublicKey`, defined at the bottom of the file) just
hashes the caller's secret with a domain-separated prefix so the same person has
a stable, unlinkable pseudonym. The only token-relevant line is the `nonceSeed`:
we seed the coin-nonce stream with a value bound to this deployment, so two
chip contracts run by the same operator can't ever produce colliding coin
nonces.

### 1.3 `mint` — the core shielded-token circuit

This is the whole point of the chips contract. Read it once in full, then we
dissect it:

```compact
export circuit mint(recipient: ZswapCoinPublicKey, amount: Uint<64>): ShieldedSendResult {
    const _sk = localSecretKey();
    const caller = getDappPublicKey(_sk);
    assert(caller == theHouse, "Only the house can mint chips");
    assert(disclose(amount) > 0, "Mint amount must be positive");

    // Evolve a fresh nonce for this mint.
    nonceCounter.increment(1);
    const idx = nonceCounter.read() as Uint<128>;
    const mintNonce = evolveNonce(idx, nonceSeed);
    nonceSeed = disclose(mintNonce);

    // Mint to the contract itself, then forward to the recipient.
    const domain = pad(32, "roulette:chip:");
    const coin = mintShieldedToken(
        disclose(domain),
        disclose(amount),
        disclose(mintNonce),
        right<ZswapCoinPublicKey, ContractAddress>(kernel.self())
    );

    // Remember the color on first mint.
    if (tokenColor == default<Bytes<32>>) {
        tokenColor = disclose(coin.color);
    }

    const qualified = QualifiedShieldedCoinInfo {
        nonce: coin.nonce,
        color: coin.color,
        value: coin.value,
        mt_index: 0 as Uint<64>
    };
    const result = sendShielded(
        qualified,
        left<ZswapCoinPublicKey, ContractAddress>(disclose(recipient)), @TODO -- should this be ownPublicKey()
        disclose(amount) as Uint<128>
    );

    totalMinted.increment(disclose(amount) as Uint<16>);
    return result;
}
```

Now the shielded-token mechanics, step by step.

#### (a) The nonce: why a coin needs one, and `evolveNonce`

```compact
nonceCounter.increment(1);
const idx = nonceCounter.read() as Uint<128>;
const mintNonce = evolveNonce(idx, nonceSeed);
nonceSeed = disclose(mintNonce);
```

Every shielded coin is identified by a nonce — a unique 32-byte value that
makes the coin's on-chain commitment unforgeable and un-linkable. If you ever
minted two coins with the same nonce, they'd collide. So you need a stream of
distinct nonces.

`evolveNonce(index: Uint<128>, nonce: Bytes<32>) -> Bytes<32>` is the standard
library's nonce ratchet: give it the previous seed and a monotonically
increasing index, and it deterministically produces the next nonce. Here we bump
`nonceCounter`, feed the count in as the index, and roll `nonceSeed` forward so
the next mint continues the chain. The result, `mintNonce`, is this coin's
identity.

#### (b) `mintShieldedToken`

Create the domain separator, feed it into the coin and mint to the contract:

```compact
const domain = pad(32, "roulette:chip:");
const coin = mintShieldedToken(
    disclose(domain),
    disclose(amount),
    disclose(mintNonce),
    right<ZswapCoinPublicKey, ContractAddress>(kernel.self())
);
```

`mintShieldedToken` has the signature:

```
mintShieldedToken(
    domainSep: Bytes<32>,
    value:     Uint<64>,
    nonce:     Bytes<32>,
    recipient: Either<ZswapCoinPublicKey, ContractAddress>
) -> ShieldedCoinInfo
```

Four things to internalise here:

1. **The domain separator determines the color.** A shielded coin's `color` is
   derived from `(domainSep, this contract's address)`. Because both halves are
   fixed for a given deployment, every chip this contract mints — regardless
   of amount or recipient — shares the same color. That's what makes "chips" a
   single fungible token type. Change the domain string and you'd be minting a
   different token.

2. **`Uint<64>` for the mint value.** Note the width: `mintShieldedToken` takes a
   `Uint<64>` value (not `Uint<128>`). `sendShielded` below uses `Uint<128>`.
   Getting these widths wrong is a common compile error.

3. **We mint to the contract itself, not to the recipient.** The recipient
   argument is `right<ZswapCoinPublicKey, ContractAddress>(kernel.self())`.
   - `kernel.self()` is this contract's own address. This only resolves reliably
     after the constructor executes, during constructor execution `kernel.self()`
     resolves differently. It is usually only useful to call `kernel.self()` after
     the constructor executes.
   - Recipients are an `Either<ZswapCoinPublicKey, ContractAddress>`: the
     `left` arm is a user's wallet key; the `right` arm is a contract
     address. So `right(kernel.self())` means "mint this coin to the contract."

   Why mint to self instead of straight to the player? This is the "mint to
   self, then send" pattern. `mintShieldedToken` returns a brand-new
   `ShieldedCoinInfo` owned by the contract; we then hand it off with
   `sendShielded`, which is the operation that produces a proper spend + change
   and pays the player. Minting directly to the player is possible, but routing
   through the contract keeps the value flow explicit and gives us the
   `ShieldedSendResult` to return.

4. **The value is private.** `amount` is disclosed to the mint operation (the
   circuit needs it to build the coin), but it is not written to any public
   ledger field. It lives inside the coin's zswap UTXO. On-chain observers see
   that a mint happened and the coin's color, but not how many chips were
   minted to whom.

The returned `coin: ShieldedCoinInfo` has exactly three fields:

```
ShieldedCoinInfo = { nonce: Bytes<32>, color: Bytes<32>, value: Uint<128> }
```

#### (c) Recording the color on first mint

@TODO -- default returns to all zeroes, which is the color of the NIGHT token. Fix this.
```compact
if (tokenColor == default<Bytes<32>>) {
    tokenColor = disclose(coin.color);
}
```

The first time we mint, `tokenColor` is still its zero default, so we stash
`coin.color`. This publishes the chip color to the ledger so the roulette
contract can look it up and enforce "bets must be paid in chips." The color is
not secret — publishing it is intentional and safe.

#### (d) `sendShielded`

Pay out the coin to the minter:
```compact
const qualified = QualifiedShieldedCoinInfo {
    nonce: coin.nonce,
    color: coin.color,
    value: coin.value,
    mt_index: 0 as Uint<64>
};
const result = sendShielded(
    qualified,
    left<ZswapCoinPublicKey, ContractAddress>(disclose(recipient)),
    disclose(amount) as Uint<128>
);
```

`sendShielded` spends a coin the contract owns and pays some value to a
recipient:

```
sendShielded(
    input:     QualifiedShieldedCoinInfo,
    recipient: Either<ZswapCoinPublicKey, ContractAddress>,
    value:     Uint<128>
) -> ShieldedSendResult
```

Two shielded-specific details:

- **`ShieldedCoinInfo` vs `QualifiedShieldedCoinInfo`.** `sendShielded` can only
  spend a *qualified* coin — one that also carries an `mt_index` (its position in
  the zswap commitment Merkle tree). The coin we just minted is a plain
  `ShieldedCoinInfo`, so we wrap it, copying `nonce`/`color`/`value` straight
  across and setting `mt_index: 0`. `0` is correct here because the coin was
  created in this same transaction — it has no prior committed position in the
  tree yet.

- **The `left` arm this time.** Now the recipient is a real player, so we use
  `left(...)` (a `ZswapCoinPublicKey`), the opposite arm from the mint.

`sendShielded` returns a `ShieldedSendResult`:

```
ShieldedSendResult = { change: Maybe<ShieldedCoinInfo>, sent: ShieldedCoinInfo }
```

`sent` is the coin that went to the recipient; `change` is any left-over value
returned to the sender (present only if you send less than the input coin's
value). Because we send the full `amount`, there is no change — but we still
return the whole `result` to the caller so the SDK/wallet can pick up the
outputs.

#### (e) A word on `disclose(...)` everywhere

You'll notice almost every argument to a token function is wrapped in
`disclose(...)`: `disclose(domain)`, `disclose(amount)`, `disclose(mintNonce)`,
`disclose(recipient)`. Compact treats witness-derived and otherwise-private
values as non-disclosed by default; passing them into an operation that has a
**public effect** (minting or moving coins on the ledger) requires you to state
explicitly that you accept revealing them at that boundary. Forgetting a
`disclose` here is the classic "potential witness-value disclosure must be
declared" compile error.

### 1.4 Identity helper

```compact
circuit getDappPublicKey(_secret: Bytes<32>): Bytes<32> {
    return persistentHash<Vector<2, Bytes<32>>>([pad(32, "roulette:pk:"), _secret]);
}
```

Not token-specific: it hashes the caller's secret behind a `"roulette:pk:"`
domain prefix to produce a stable, cross-dapp-unlinkable pseudonym. The roulette
contract uses the identical derivation so a player has the same identity in
both contracts.

That's the entire chips contract.

---

## Part 2 — `roulette.compact`: a game that custodies chip coins

Create `roulette.compact`:  

```bash
touch `contract/roulette.compact`
```

The roulette contract is longer, but most of
its length is game logic (assertions, the wheel, commit/reveal of the winning
number). The tutorial moves quickly through those and slow down every time a chip coin
moves.

### 2.1 Header, enums, ledger state

```compact
pragma language_version 0.23;
import CompactStandardLibrary;

export enum BetState { CLOSED, OPEN }
export enum Color { GREEN, RED, BLACK }   // GREEN = the zero pocket only

export sealed ledger theHouse: Bytes<32>;
export sealed ledger chipColor: Bytes<32>;
export sealed ledger winningNumHash: Bytes<32>;
export ledger betState: BetState;
export ledger color: Color;
export ledger bets: Map<Bytes<32>, Color>;
export ledger betCoins: Map<Bytes<32>, QualifiedShieldedCoinInfo>;
export ledger winnerList: Set<Bytes<32>>;
export ledger matchedWinners: Set<Bytes<32>>;
export ledger houseCoins: Map<Bytes<32>, QualifiedShieldedCoinInfo>;

witness localSecretKey(): Bytes<32>;
```

The two ledger fields that matter for tokens are the coin maps:

- `betCoins: Map<Bytes<32>, QualifiedShieldedCoinInfo>` — each player's bet
  chip, held in escrow by the contract, keyed by the player's pseudonym.
- `houseCoins: Map<Bytes<32>, QualifiedShieldedCoinInfo>` — the house's
  pre-deposited "match" coins used to pay the 2× winnings, keyed by each coin's
  nonce.

Storing `QualifiedShieldedCoinInfo` in ledger state is how a contract keeps
custody of a coin between transactions: it received the coin earlier, and it
remembers the qualified handle so it can `sendShielded` that exact coin later.

`chipColor` is `sealed` and bound at deploy time — this is the color the game
will accept, read straight from the chips contract's `tokenColor`.

### 2.2 Constructor

Bind the chip color and commit to the winning number:

```compact
constructor(_winningNum: Uint<8>, allowedChipColor: Bytes<32>) {
    assert(_winningNum >= 0 && _winningNum <= 36, "Cheat Detected: theHouse: Please keep the number on the table");
    const _sk = localSecretKey();
    theHouse = disclose(getDappPublicKey(_sk));
    chipColor = disclose(allowedChipColor);
    winningNumHash = commitWithSk(_winningNum as Bytes<32>, _sk);
    betState = BetState.OPEN;
}
```

The one token-relevant line is `chipColor = disclose(allowedChipColor)`: the deployer
passes in the chips contract's published color, and from now on the game only
accepts coins of that color. The winning number is stored as a *commitment*
(`commitWithSk` hashes it with the house secret) and revealed later — standard
commit/reveal, not token-specific.

### 2.3 `houseDeposit` — a contract receiving and custodying a coin

Before any bets pay out, the house parks matching coins in the contract:

```compact
export circuit houseDeposit(coin: ShieldedCoinInfo): [] {
    const _sk = localSecretKey();
    assert(getDappPublicKey(_sk) == theHouse, "Only the house can deposit match funds");
    assert(coin.color == chipColor, "Deposit must be made with roulette chips");

    receiveShielded(disclose(coin));
    houseCoins.insertCoin(
        disclose(coin.nonce),
        disclose(coin),
        right<ZswapCoinPublicKey, ContractAddress>(kernel.self())
    );
}
```

This is the receive side of shielded tokens, and it's a two-step handshake:

1. `receiveShielded(coin)` accepts an incoming coin into the contract's
   custody. A shielded coin sent to a contract address is not automatically
   spendable by the contract — the receiving contract must explicitly call
   `receiveShielded` to take it into its zswap state. Without this call the coin
   would be effectively stranded.

   The coin comes in as a plain `ShieldedCoinInfo` (the caller's wallet built it
   as an input to this transaction). Note the guard `coin.color == chipColor`:
   the contract refuses anything that isn't a roulette chip.

2. `houseCoins.insertCoin(key, coin, owner)` records the coin in ledger map
   state so the contract can find and spend it in a later transaction.
   `insertCoin` is a ledger-`Map` method specific to coin storage: it takes the
   received `ShieldedCoinInfo` and stores it as a `QualifiedShieldedCoinInfo`
   (the qualified form carries the `mt_index` needed to spend it later),
   attributing ownership to `right(kernel.self())` — the contract itself. We key
   it by the coin's own `nonce`, which is guaranteed unique, so no separate
   deposit counter is needed.

Together: `receiveShielded` takes the coin into custody now; `insertCoin`
persists a spendable handle for later. Every coin the roulette contract holds
goes through this exact pair.

> The author left a `@TODO` on the `disclose(coin)` here, musing that disclosing
> the coin "feels" like exposing it. It doesn't expose the *value* on the public
> ledger — the value stays in the UTXO — but the `disclose` is still required
> because the coin flows into a public ledger effect (see §1.3(e)).

### 2.4 `betColor`

A player sends a coin into the escrow of the contract and places a bet:

```compact
export circuit betColor(coin: ShieldedCoinInfo, colorBet: Color): [] {
    assert(betState == BetState.OPEN, "Not ready to accept bets yet");
    assert(colorBet == Color.RED || colorBet == Color.BLACK, "Only RED or BLACK bets are allowed");

    const _sk = localSecretKey();
    const player = getDappPublicKey(_sk);
    assert(player != theHouse, "theHouse cannot make bets");

    const pubPlayer = disclose(player);
    assert(!betCoins.member(pubPlayer), "Already placed a bet this round");
    assert(disclose(coin.color) == chipColor, "Bet must be made with roulette chips");

    receiveShielded(disclose(coin));
    betCoins.insertCoin(
        pubPlayer,
        disclose(coin),
        right<ZswapCoinPublicKey, ContractAddress>(kernel.self())
    );
    bets.insert(pubPlayer, disclose(colorBet));
}
```

The coin handling is exactly the same receive+insert pair as `houseDeposit`
— that's the reusable shape for a contract that takes custody of a shielded coin. The only
differences are the guards and bookkeeping:

- keyed by the player's pseudonym `pubPlayer` instead of the coin nonce, so a
  player can bet at most once per round;
- the same `coin.color == chipColor` check;
- the chosen color is recorded publicly in `bets`.

Privacy note: the player's wallet identity stays private — only the pseudonym `pubPlayer` and the bet
color become public. The bet value, however, is `betCoins[pubPlayer].value`
and is publicly readable. This example is deliberately "identity-private,
behaviour-public."

### 2.5 `revealWinningNumber`

Bets are in and `theHouse` can now spin the wheel and reveal the winningNumber:

```compact
export circuit revealWinningNumber(winningNum: Uint<8>): [] {
    const _sk = localSecretKey();
    assert(getDappPublicKey(_sk) == theHouse, "Only the House can reveal the number");
    const hash = commitWithSk(winningNum as Bytes<32>, _sk);
    assert(hash == winningNumHash, "Cheat Detected: theHouse: tried to change the winning number");
    betState = BetState.CLOSED;
    color = getColor(disclose(winningNum));
}
```

No tokens move. The house re-hashes the number and checks it against the
commitment made at deploy time so it can't change the outcome after seeing the
bets, closes betting, and publishes the winning color.

### 2.6 Paying a winner: two `sendShielded` calls, and why it's split

A winner gets 2× their own stake back, because of a matching coin from `theHouse`.
Crucially this is done in two separate circuits, not one:

```compact
// Phase 1: return the player's own bet coin.
export circuit claimMyBet(): [] {
    assert(betState == BetState.CLOSED, "Winning number has not been revealed");
    const _sk = localSecretKey();
    const player = disclose(getDappPublicKey(_sk));
    assert(bets.member(player), "You did not place a bet");
    assert(bets.lookup(player) == color, "You did not place a winning bet");
    assert(!winnerList.member(player), "Already claimed bet");

    const coin = betCoins.lookup(player);
    sendShielded(
        coin,
        left<ZswapCoinPublicKey, ContractAddress>(ownPublicKey()),
        coin.value as Uint<128>
    );
    winnerList.insert(player);
}
```

`ownPublicKey` is safe to use here, because the caller of this circuit has already been authenticated. @TODO -- add a link here to the security section.

Send one fo the house's match coins of equal value:

```compact
export circuit claimMatch(matchKey: Bytes<32>): [] {
    const _sk = localSecretKey();
    const player = disclose(getDappPublicKey(_sk));
    assert(winnerList.member(player), "Must claim your bet first");
    assert(!matchedWinners.member(player), "Already claimed match");
    assert(houseCoins.member(disclose(matchKey)), "Match coin not available");

    const matchCoin = houseCoins.lookup(disclose(matchKey));
    const betCoin = betCoins.lookup(player);
    assert(matchCoin.value == betCoin.value, "Match coin must equal your bet value");

    sendShielded(
        matchCoin,
        left<ZswapCoinPublicKey, ContractAddress>(ownPublicKey()),
        matchCoin.value as Uint<128>
    );
    houseCoins.remove(disclose(matchKey));
    betCoins.remove(player);
    matchedWinners.insert(player);
}
```

Here's what's shielded-token-important:

- **Spending a custodied coin.** Each `sendShielded` reads a
  `QualifiedShieldedCoinInfo` straight out of ledger map state
  (`betCoins.lookup(player)`, `houseCoins.lookup(matchKey)`) and spends it.
  Because those coins were stored qualified (via `insertCoin`, with a real
  `mt_index`), they're immediately spendable — no wrapping needed, unlike the
  freshly-minted coin in §1.3(d).

- **`ownPublicKey()` as the recipient.** The payout goes to
  `left(ownPublicKey())` — `ownPublicKey()` returns the transaction author's
  `ZswapCoinPublicKey`, i.e. the real wallet calling the circuit as a witness function. This is a
  subtle privacy point: the player's on-chain pseudonym is unlinkable to their
  wallet, but a payout must land in an actual wallet key, so the coin does go to
  a concrete `ZswapCoinPublicKey`. `ownPublicKey()` is safe to use here, because
  the caller has already been authenticated.

- **The value equality is enforced in-circuit.** Phase 2 asserts
  `matchCoin.value == betCoin.value` by reading both values from public ledger
  state. That's what mechanically guarantees the 2× is exactly 2× — no
  off-chain trust between the house and the player. This is also why
  `claimMyBet` deliberately does not remove `betCoins[player]`: `claimMatch`
  still needs the bet coin's value for this check, and only removes it
  afterward.

- **Why two circuits instead of one?** A single circuit that calls
  `sendShielded` is limited by the node's per-circuit "effects" budget (roughly
  3 coin operations; exceeding it triggers `EffectsCheckFailure 186`). Returning
  the stake and a match coin in one circuit blows that budget, so the 2×
  payout is split into two transactions. This is a real, practical constraint
  worth remembering whenever you move more than a couple of coins per circuit.

### 2.7 House sweeps losers

Losers are not allowed to claim, so the house reclaims the stranded coins:

```compact
export circuit houseClaimBet(loserPseudonym: Bytes<32>): [] {
    assert(betState == BetState.CLOSED, "Winning number has not been revealed");
    const _sk = localSecretKey();
    assert(getDappPublicKey(_sk) == theHouse, "Only the house can sweep");
    assert(bets.member(disclose(loserPseudonym)), "That player did not place a bet");
    assert(bets.lookup(disclose(loserPseudonym)) != color, "That player won");
    assert(betCoins.member(disclose(loserPseudonym)), "No bet coin to sweep");

    const coin = betCoins.lookup(disclose(loserPseudonym));
    sendShielded(coin, left<ZswapCoinPublicKey, ContractAddress>(ownPublicKey()), coin.value as Uint<128>);
    betCoins.remove(disclose(loserPseudonym));
}

export circuit houseClaimMatch(matchKey: Bytes<32>): [] {
    const _sk = localSecretKey();
    assert(getDappPublicKey(_sk) == theHouse, "Only the house can sweep");
    assert(houseCoins.member(disclose(matchKey)), "Match coin not available");
    const coin = houseCoins.lookup(disclose(matchKey));
    sendShielded(coin, left<ZswapCoinPublicKey, ContractAddress>(ownPublicKey()), coin.value as Uint<128>);
    houseCoins.remove(disclose(matchKey));
}
```

Same "lookup a custodied `QualifiedShieldedCoinInfo` → `sendShielded` to
`ownPublicKey()` → remove from the map" shape you've now seen four times. Only
the authorisation and win/loss guards differ.

### 2.8 Wheel + identity helpers

```compact
circuit getOdd(num: Uint<8>): Boolean { /* returns true for the RED numbers */ }
circuit getColor(num: Uint<8>): Color {
    if (num == 0) { return Color.GREEN; }
    if (getOdd(num) == true) { return Color.RED; } else { return Color.BLACK; }
}
circuit getDappPublicKey(_secret: Bytes<32>): Bytes<32> {
    return persistentHash<Vector<2, Bytes<32>>>([pad(32, "roulette:pk:"), _secret]);
}
circuit commitWithSk(_winningNum: Bytes<32>, _sk: Bytes<32>): Bytes<32> {
    return disclose(persistentHash<Vector<2, Bytes<32>>>([_winningNum, _sk]));
}
```

Pure helper circuits: the standard single-zero roulette color table (single zero is back!), the shared
pseudonym derivation (identical to the chips contract), and the winning-number
commitment. Copy `getOdd` in full from the reference source — it's just the list
of RED pockets `1,3,5,…,35`.

---

## Part 3 — Compile

With both `.compact` files written, generate the compiled artifacts:

```bash
yarn compile
```

Successful output:

```
Compiling 7 circuits:
  circuit "betColor" (k=15, rows=16583)  
  circuit "claimMatch" (k=15, rows=16762)  
  circuit "claimMyBet" (k=15, rows=16481)  
  circuit "houseClaimBet" (k=15, rows=16770)  
  circuit "houseClaimMatch" (k=15, rows=16756)  
  circuit "houseDeposit" (k=15, rows=16532)  
  circuit "revealWinningNumber" (k=14, rows=8542)  
Overall progress [====================] 7/7                                                                                       
Compiling 1 circuits:
  circuit "mint" (k=15, rows=26944)  
```

The compile script will compile both the chips and the roulette contracts.

---

## Part 4 — Run the game end to end

The end-to-end test in `src/test/roulette.test.ts` plays a full round against a
local devnet: Alice (the house) deploys chips, mints to Alice/Bob/Claire,
deploys roulette bound to the chip color, deposits two match coins, Bob bets
RED (wins) and Claire bets BLACK (loses), Alice reveals RED, Bob claims 2× in
two phases, and Alice sweeps the leftovers.

Be sure that the Docker engine is running and start the local devnet:

```bash
yarn env:up
```

Run the test script:

```bash
yarn test:local
```

Watch the log lines like "Bob chip balance after claimMatch: 200" — that `100→ 200` 
is the private chip value moving through the `sendShielded` calls you
wrote in Part 2, and never appearing on the public ledger.

Successful output looks like this:

```
[11:25:30.951] INFO (36852): Wallet sync [27]: shielded=true, unshielded=true, dust=true
[11:25:30.951] INFO (36852): Wallet sync complete after 27 emissions
[11:25:30.954] INFO (36852): All providers initialized.
[11:25:30.955] INFO (36852): Deploying chips contract...
[11:25:50.895] INFO (36852): Chips contract deployed at bf4b22efc6063e01a96869cfc7267ffb2ab48af6f848eb8cf7043cb96d88abed
[11:25:50.986] INFO (36852): Alice minting 100 chips to herself (coin #1)...
[11:26:14.938] INFO (36852): Alice minting 100 chips to herself (coin #2)...
[11:26:39.011] INFO (36852): Alice minting 100 chips to Bob...
[11:27:03.102] INFO (36852): Alice minting 100 chips to Claire...
[11:27:25.863] INFO (36852): Chip token color: d0fc7faf7f9efa7453f0487c63ee20a34fdd9360ce06cf4a050bc176a8993891
[11:27:25.863] INFO (36852): Syncing wallet...
[11:27:25.865] INFO (36852): Wallet sync [1]: shielded=true, unshielded=true, dust=true
[11:27:25.865] INFO (36852): Wallet sync complete after 1 emissions
[11:27:25.866] INFO (36852): Syncing wallet...
[11:27:25.868] INFO (36852): Wallet sync [1]: shielded=true, unshielded=true, dust=true
[11:27:25.868] INFO (36852): Wallet sync complete after 1 emissions
[11:27:25.868] INFO (36852): Syncing wallet...
[11:27:25.870] INFO (36852): Wallet sync [1]: shielded=true, unshielded=true, dust=true
[11:27:25.870] INFO (36852): Wallet sync complete after 1 emissions
[11:27:25.879] INFO (36852): Deploying roulette contract bound to chip color d0fc7faf7f9efa7453f0487c63ee20a34fdd9360ce06cf4a050bc176a8993891...
[11:27:44.648] INFO (36852): Roulette deployed at d1b84317e3fbee14ba3bb6b65341dbada924b18c46af2f0ef4dd8c077c588ffe
[11:27:44.657] INFO (36852): Alice depositing match coin #1 (value=100)...
[11:28:08.674] INFO (36852): Syncing wallet...
[11:28:08.675] INFO (36852): Wallet sync [1]: shielded=true, unshielded=true, dust=true
[11:28:08.675] INFO (36852): Wallet sync complete after 1 emissions
[11:28:08.679] INFO (36852): Alice depositing match coin #2 (value=100)...
[11:28:32.791] INFO (36852): Syncing wallet...
[11:28:32.793] INFO (36852): Wallet sync [1]: shielded=true, unshielded=true, dust=true
[11:28:32.793] INFO (36852): Wallet sync complete after 1 emissions
[11:28:32.845] INFO (36852): Bob is betting a chip coin (value=100) on RED
[11:28:56.934] INFO (36852): Claire is betting a chip coin (value=100) on BLACK
[11:29:20.973] INFO (36852): Alice revealing the winning number...
[11:30:02.441] INFO (36852): Syncing wallet...
[11:30:02.442] INFO (36852): Wallet sync [1]: shielded=true, unshielded=true, dust=true
[11:30:02.442] INFO (36852): Wallet sync complete after 1 emissions
[11:30:02.445] INFO (36852): Bob chip balance after claimMyBet: 100
[11:30:02.459] INFO (36852): Bob claiming match with key=69b601c310697831359676448057e8855e3b5d7d1a39a30bda25f6c7d4711e00
[11:30:26.526] INFO (36852): Syncing wallet...
[11:30:26.527] INFO (36852): Wallet sync [1]: shielded=true, unshielded=true, dust=true
[11:30:26.527] INFO (36852): Wallet sync complete after 1 emissions
[11:30:26.530] INFO (36852): Bob chip balance after claimMatch: 200
[11:30:26.547] INFO (36852): Alice sweeping loser=7bd4a4c861d992a0144b772fb5e3e130828495df061b4b3dd287b1154f273619
[11:30:50.628] INFO (36852): Syncing wallet...
[11:30:50.630] INFO (36852): Wallet sync [1]: shielded=true, unshielded=true, dust=true
[11:30:50.630] INFO (36852): Wallet sync complete after 1 emissions
[11:30:50.633] INFO (36852): Alice chip balance after sweeping bet: 100
[11:30:50.643] INFO (36852): Alice sweeping match=c0c51cad85e4200323742db107632a78c75262c4c7489c53ab3b3007ca6ede00
 ✓ src/test/roulette.test.ts (12 tests) 345610ms
   ✓ Roulette tutorial (RED/BLACK, 2x payouts, house match) (12)
     ✓ Alice deploys the chips contract  19941ms
     ✓ Alice mints chips to herself (2 for matches), Bob, and Claire  94983ms
     ✓ Alice deploys the roulette contract bound to the chip color  18776ms
     ✓ Alice deposits two match coins so the house can cover both bets  48142ms
     ✓ Bob bets RED with a 100-chip coin  24089ms
     ✓ Claire bets BLACK with a 100-chip coin  24087ms
     ✓ Alice reveals the winning number (RED)  17403ms
     ✓ Bob claims his bet back (phase 1: claimMyBet → 1x)  24078ms
     ✓ Bob claims a match coin (phase 2: claimMatch → total 2x)  24086ms
     ✓ Alice sweeps Claire's bet coin via houseClaimBet  24099ms
     ✓ Alice sweeps the remaining unused match coin via houseClaimMatch  24091ms
     ✓ Claire cannot claim — she bet BLACK but the winning color was RED 64ms
```

---

## Shielded-token cheat-sheet

| Operation | Function | Coin in | Coin out |
|-----------|----------|---------|----------|
| Create a coin | `mintShieldedToken(domainSep, value:Uint<64>, nonce, recipient)` | — | `ShieldedCoinInfo` |
| Fresh nonce | `evolveNonce(index:Uint<128>, seed)` | — | `Bytes<32>` |
| Accept an incoming coin | `receiveShielded(coin)` | `ShieldedCoinInfo` | `[]` |
| Store a coin in a ledger `Map` | `map.insertCoin(key, coin, owner)` | `ShieldedCoinInfo` | (stored `QualifiedShieldedCoinInfo`) |
| Spend a coin | `sendShielded(qualified, recipient, value:Uint<128>)` | `QualifiedShieldedCoinInfo` | `ShieldedSendResult` |
| This contract's address | `kernel.self()` → wrap with `right(...)` | | |
| Caller's wallet key(witness) | `ownPublicKey()` → wrap with `left(...)` | | |

Key type reminders:

- `ShieldedCoinInfo = { nonce, color, value }` — a coin created this
  transaction.
- `QualifiedShieldedCoinInfo = { nonce, color, value, mt_index }` — a coin
  already committed to the ledger, spendable. Use `mt_index: 0` only for a coin
  minted in the current transaction.
- `ShieldedSendResult = { change: Maybe<ShieldedCoinInfo>, sent: ShieldedCoinInfo }`.
- Recipients are `Either<ZswapCoinPublicKey, ContractAddress>`: `left` = wallet
  key, `right` = contract address.
- Coin values are never written to public ledger state — that privacy is the
  whole reason to use shielded tokens.

## Next Steps
@TODO -- fill in next steps here