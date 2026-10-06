# Pot 21

Multiplayer blackjack where every seat stakes the same amount, the best hand takes the pot, and the house keeps 2%.

## Run the demo (no blockchain needed)

```bash
npm install
npm run build        # bundles the browser client into client/dist
npm start            # http://localhost:8787
```

Press **Sign in with X**. Without a Privy app ID configured, demo mode asks you to type any handle instead, and new accounts start with 1,000 demo CHIP. Connecting Phantom in the wallet panel credits it with demo CHIP to try deposits and withdrawals.

To play other people, open the site in two browser windows (or one normal and one private) and sit at the same table. To play alone, fill the empty seats with bots.

`node client/preview/build.mjs` builds a single-file preview that runs the server inside the page, with a stand-in Phantom wallet and bots.

`npm test` runs the engine, fee, fairness and protocol tests.

## Finding a game

The lobby has two parts:

- **Play now:** pick a stake and press one button. If someone is already waiting at that stake, you join their table. Otherwise a 2-player table opens and appears for others. If your game wallet is short, the button becomes **Deposit to play**.
- **Players waiting:** every table that has people seated and free seats, closest to starting first, each with a **Join** button.

Seat count (2–6) and demo bots are under **More options**.

## How a hand works

1. **Sit down.** A table has 2–6 seats and a fixed stake. When every seat is filled, the hand opens.
2. **Stake.** Each player has 60 seconds to stake into the round's escrow. If anyone doesn't, every stake is refunded in full (no fee) and the non-stakers lose their seats.
3. **Play.** Cards are dealt two each. Players act in seat order, starting from a button that moves one seat each hand. You get 20 seconds per move; when time runs out you stand.
4. **Settle.** The highest hand of 21 or under wins, and a two-card 21 beats any other 21. Tied winners split the pot. If everyone busts, the pot is refunded evenly, less the fee.
5. **Fee.** `fee = floor(stake × seats × 2%)` goes to the house wallet. The winners get the rest. Any unit left over after an even split goes to the first winner, so no tokens are ever lost.

## Wallets, deposits and withdrawals

- **Sign in with X** (through Privy) is the first thing players see. On their first sign-in, Privy creates a Solana wallet for them. That wallet is their **game wallet**: stakes are paid from it, winnings land in it, and Privy signs its transactions.
- **Phantom** is optional. Players connect it from the wallet panel, by signing a free message, to deposit from it and withdraw to it. Anyone can also send tokens straight to their game wallet address.
- Signing in with Phantom instead is under "Other ways to sign in". That path gives the player a game wallet whose key is kept in the browser, with a backup-key export.
- **Deposit**: Phantom → game wallet. **Withdraw**: game wallet → Phantom.
- **Fees**: 2% on deposits, 2% on withdrawals (`DEPOSIT_FEE_BPS`, `WITHDRAW_FEE_BPS`; 200 = 2%, 0 turns it off) and 2% on every pot. On a deposit the fee comes out of the amount sent; on a withdrawal it comes out of the amount withdrawn.
- **Gas**: a deposit also sends 0.01 SOL to the game wallet when it is running low, so it can pay for stakes and withdrawals.

The pot fee is enforced by the escrow program. The deposit and withdrawal fees are added by the site's transactions, but players hold their own keys, so someone sending tokens by hand can skip them. Enforcing those too would need the game wallet's funds to sit in a program account.

## Setting up Sign in with X (Privy)

1. Create an app at [dashboard.privy.io](https://dashboard.privy.io).
2. **Login methods → Socials:** turn on **X (Twitter)**.
3. **Wallets:** enable Solana embedded wallets. The site also asks Privy to create one for every new player on sign-in.
4. **Allowed origins** and **allowed OAuth redirect URLs:** add your site's address, plus `http://localhost:8787` for local testing.
5. Copy the **App ID**, **App secret** and **Verification key** into `.env`:
   ```
   PRIVY_APP_ID=...
   PRIVY_APP_SECRET=...
   PRIVY_VERIFICATION_KEY="-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----"
   ```
6. Run `npm install`, `npm run build` and `npm start`. The sign-in page now goes through X.

How it works: the browser loads Privy's React SDK in a small hidden React root (`client/src/privy/react-bridge.ts`) and sends the Privy access token to the server. The server checks the token's signature against your verification key (`server/src/privy.ts`). It then asks Privy's API for the player's X username and Privy wallet address. Without `PRIVY_APP_ID`, demo mode shows a stand-in X sign-in where you type a handle.

## Putting it online

The site is set up for one-click hosting on [Render](https://render.com). You can also run the `Dockerfile` on any host that runs containers.

1. **Put the code on GitHub.** Create a private repository and upload the contents of the Pot21 folder. Leave out `.env` and `.env.txt`; `.gitignore` already skips them.
2. **Create the service.** On Render, choose **New → Blueprint** and pick your repository. Render reads `render.yaml` and sets everything up.
3. **Add your Privy keys.** Render asks for `PRIVY_APP_ID`, `PRIVY_APP_SECRET` and `PRIVY_VERIFICATION_KEY`. Paste them there; they're stored as secrets, not in your code.
4. **Allow your new address in Privy.** Once Render shows your site's address (for example `https://pot21.onrender.com`), add it to **Allowed origins** and the **allowed OAuth redirect URLs** in the Privy dashboard.
5. **Share the link.** Players sign in with X and play straight away.

What you get: real X sign-in, real multiplayer, and accounts saved on a disk so a restart or redeploy doesn't sign anyone out. Tokens are still demo tokens (`LEDGER=mock`) until the Solana escrow is deployed (see "Going on-chain"). Demo token balances reset when the server restarts.

The disk for saved accounts needs a paid Render plan. On the free plan, remove the `disk:` section from `render.yaml`; the site still works, but everyone has to sign in again after each restart.

## Fair shuffle

Before anyone stakes, the server publishes `sha256(serverSeed)`. Each player's own seed is mixed into the shuffle. After the hand, the server reveals `serverSeed`, and the **Check this deal** button re-runs the shuffle in your browser and compares every card. The algorithm is in `shared/src/fairness.ts`.

## Project layout

| Path | What it is |
|---|---|
| `shared/` | Rules, fee maths, fair shuffle and the wire protocol, used by both server and browser. `escrow.ts` builds the escrow program's instructions. |
| `server/` | WebSocket game server. `table.ts` runs a table, `hub.ts` handles requests, `ledger/` moves tokens (`mock.ts` for the demo, `solana.ts` for on-chain). |
| `client/` | Browser app (plain TypeScript, bundled with esbuild). |
| `programs/pot21_escrow/` | Anchor (Rust) escrow program on Solana. |

## Going on-chain (devnet)

The escrow program holds every stake. It only ever pays to a seated player's own token account or to the house account. If the server disappears, anyone can trigger a full refund one hour after a round opens.

```bash
# 1. Build and deploy the program (needs the Solana CLI and Anchor 0.31)
anchor keys sync          # replaces the placeholder program id
anchor build && anchor deploy --provider.cluster devnet

# 2. One-time config: 2% fee, house wallet, server key
solana-keygen new -o authority-keypair.json
ADMIN_KEYPAIR=~/.config/solana/id.json PROGRAM_ID=<id> TOKEN_MINT=<mint> \
HOUSE_WALLET=<house owner> AUTHORITY_PUBKEY=$(solana-keygen pubkey authority-keypair.json) \
npx tsx server/scripts/init-config.ts

# 3. Run the server against devnet (fill in .env from .env.example)
LEDGER=solana npm start
```

The token can be your own SPL token or wrapped SOL (mint `So11111111111111111111111111111111111111112`).

In solana mode, players sign in with Phantom (switch Phantom to devnet in its settings) and deposit from it. The game wallet signs its own stakes and withdrawals. The server's authority key needs devnet SOL to pay for opening and settling rounds. Players need devnet SOL for fees and some of the token.

## Not done yet

- **Privy SDK untested here.** The Privy integration was tested end to end against a stand-in with the same hooks, real signed tokens and a fake Privy API. The real `@privy-io/react-auth` package couldn't be installed in the build environment. The first run with your Privy app ID is the real test.
- **Phantom-sign-in game wallets** keep their key in the browser. Clearing site data without the backup loses that wallet. Privy accounts don't have this problem.
- **Untested here:** the Anchor program, `server/src/ledger/solana.ts`, `shared/src/escrow.ts` and `client/src/solana.ts` (on-chain stakes, deposits and withdrawals). They were written to the Anchor 0.31 and `@solana/web3.js` 1.x APIs but have not been compiled or run against a validator. The settlement maths in the program is tested and matches the TypeScript. The next step is an Anchor test suite on a local validator.
- **Trust model:** the server decides who won. The escrow stops it from sending funds anywhere else, and the revealed seed lets players check every deal. It cannot stop a dishonest server from naming the wrong winner. Keep the authority key on a locked-down machine.
- **Saved data:** accounts, linked wallets and sign-ins are saved to `DATA_DIR/accounts.json` (sign-in tokens are stored hashed). Tables in progress and demo token balances are not saved. With real tokens, balances live on chain.
