# Pot 21: start here

Multiplayer blackjack on Solana. Players sign in with X (through Privy), get a Solana wallet automatically, stake into a shared pot and the best hand wins it. The house keeps 2%.

## Just want to see it?

Open **Play-preview.html** in any web browser. It runs the whole game inside the page with practice bots and demo tokens. No setup needed.

## Run the real site on your computer

1. Install Node.js 20 or newer: https://nodejs.org
2. Open a terminal in this folder and run:
   ```
   npm install
   npm run build
   npm start
   ```
3. Go to http://localhost:8787 in two browser windows to play against yourself, or fill seats with bots.

Without Privy keys it runs in demo mode with a stand-in X sign-in and demo tokens. To switch on real X sign-in, put your Privy keys in a file called `.env` (or `.env.txt`) in this folder. See `.env.example` for the format.

## Put it online for everyone

Follow **"Putting it online"** in README.md: upload this folder to GitHub, create the site on Render from `render.yaml`, paste your Privy keys into Render, and share the link.

## What's in this folder

| Folder | What it is |
|---|---|
| `client/` | The website players see |
| `server/` | The game server: tables, dealing, sign-in, wallets |
| `shared/` | Game rules, fee maths and fair-shuffle code used by both |
| `programs/pot21_escrow/` | The Solana escrow contract that holds stakes and pays winners |
| `README.md` | Full documentation and setup steps |
| `.env.example` | Settings template: copy to `.env` and fill in |
| `Dockerfile`, `render.yaml` | Hosting setup for putting the site online |

## Status

- Working and tested: the game, multiplayer tables, fees, refunds, fair shuffle, sign-in checks, deposits and withdrawals (with demo tokens).
- Written but not yet run for real: the Solana escrow contract and real-token transactions, and the real Privy SDK. These need the setup in README.md.
