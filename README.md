import { Scraper } from '@the-convocation/twitter-scraper';
import { PublicKey, Connection, Keypair, ComputeBudgetProgram, Transaction, TransactionInstruction } from '@solana/web3.js';
import * as spl from "@solana/spl-token";
import { struct, u64, u8 } from '@solana/buffer-layout';
import { u64 as u64Utils } from "@solana/buffer-layout-utils";
import bs58 from "bs58";
import dotenv from "dotenv";

dotenv.config();

// ✅ Load sensitive data securely
const rpcUrl = process.env.SOL_RPC_URL "https://api.mainnet-beta.solana.com";
const connection = new Connection(rpcUrl, "confirmed");
const twitterUsername = process.env.TWITTER_USERNAME;
const twitterPassword = process.env.TWITTER_PASSWORD;
const privateKey = bs58.decode(process.env.SOL_PRIVATE_KEY);
const wallet = Keypair.fromSecretKey(privateKey);

// ✅ Solana program & wallet settings
const pumpProgram = new PublicKey("6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P");
const solAmount = 0.028 * 10 ** 9;  // 0.028 SOL

// ✅ Twitter Scraper
const scraper = new Scraper();
let lastSeenTweetId = null;

// ✅ Bonding curve structure
const bondingLayout = struct([
    u64('virtualTokenReserves'),
    u64('virtualSolReserves'),
    u64('realTokenReserves'),
    u64('realSolReserves'),
    u64('supply'),
    u8('completed')
]);

// ✅ Secure Login to Twitter
async function login() {
    try {
        console.log("[INFO] Logging into Twitter...");
        await scraper.login(twitterUsername, twitterPassword);
        console.log("[SUCCESS] Logged in.");
    } catch (err) {
        console.error("[ERROR] Twitter Login Failed:", err.message);
    }
}

// ✅ Fetch latest tweet & check for Base58 addresses
async function checkTweets() {
    try {
        const tweet = await scraper.getLatestTweet('kanyewest', false);
        if (!tweet tweet.id === lastSeenTweetId) return;

        lastSeenTweetId = tweet.id;
        console.log("\n🔍 New Tweet Detected:");
        console.log(Text: ${tweet.text});
        console.log(Time: ${tweet.timeParsed});
        console.log(URL: ${tweet.permanentUrl});

        const base58Regex = /[1-9A-HJ-NP-Za-km-z]{32,44}/g;
        const base58Matches = tweet.text.match(base58Regex);
        if (base58Matches) {
            console.log("🔹 Possible Solana Addresses Found:");
            for (const each of base58Matches) {
                try {
                    if (bs58.decode(each).length === 32) {
                        console.log(✅ Found valid Public Key: ${each});
                        const pumpKeys = await getPumpKeys(new PublicKey(each));
                        if (pumpKeys) {
                            console.log("🚀 Executing Trade...");
                            await executeTrade(pumpKeys);
                        }
                    }
                } catch (e) {
                    console.log("[ERROR] Invalid Base58 Key:", each, e.message);
                }
            }
        }
    } catch (err) {
        console.error("[ERROR] Failed to fetch tweets:", err.message);
    }
}

// ✅ Buy Pump Token
async function executeTrade(pumpKeys) {
    try {
        const tokensAmount = await getTokensAmountForSolAmount(pumpKeys, solAmount);
        console.log([INFO] Buying ${tokensAmount} tokens for ${solAmount / 10 ** 9} SOL);

        let compute = ComputeBudgetProgram.setComputeUnitPrice({ microLamports: 2050000 });

        const createAta = spl.createAssociatedTokenAccountIdempotentInstruction(
            wallet.publicKey, pumpKeys.userAssociatedToken, wallet.publicKey, pumpKeys.mint
        );

        const buffer = Buffer.alloc(24);
        const obj = { amount: new u64Utils(tokensAmount), maxSolCost: new u64Utils(99999999999) };
        obj.amount.toArrayLike(Buffer, 'le', 8).copy(buffer, 8);
        obj.maxSolCost.toArrayLike(Buffer, 'le', 8).copy(buffer, 16);
        Buffer.from("66063d1201daebea", "hex").copy(buffer, 0);
        const instruction = new TransactionInstruction({
            keys: [
                { pubkey: pumpKeys.global, isSigner: false, isWritable: false },
                { pubkey: pumpKeys.feeRecipient, isSigner: false, isWritable: true },
                { pubkey: pumpKeys.mint, isSigner: false, isWritable: false },
                { pubkey: pumpKeys.bonding, isSigner: false, isWritable: true },
                { pubkey: pumpKeys.associatedBondingCurve, isSigner: false, isWritable: true },
                { pubkey: pumpKeys.userAssociatedToken, isSigner: false, isWritable: true },
                { pubkey: wallet.publicKey, isSigner: true, isWritable: true }
            ],
            programId: new PublicKey(pumpKeys.program),
            data: buffer
        });

        const transaction = new Transaction().add(compute).add(createAta).add(instruction);
        const txid = await connection.sendTransaction(transaction, [wallet]);

        console.log(✅ Transaction Sent: https://solscan.io/tx/${txid});
    } catch (err) {
        console.error("[ERROR] Trade Failed:", err.message);
    }
}

// ✅ Fetch token amount to buy
async function getTokensAmountForSolAmount(pumpKeys) {
    const info = await connection.getAccountInfo(pumpKeys.bonding);
    const bonding = bondingLayout.decode(info.data.slice(8, 1000));
    const price = Number(bonding.virtualSolReserves) / Number(bonding.virtualTokenReserves);
    return solAmount / price;
}

// ✅ Fetch Token Metadata
async function getPumpKeys(mint) {
    return {
        mint,
        bonding: PublicKey.findProgramAddressSync([Buffer.from('bonding-curve'), mint.toBuffer()], pumpProgram)[0],
        associatedBondingCurve: await getOwnerAta(mint, wallet.publicKey),
        userAssociatedToken: await getOwnerAta(mint, wallet.publicKey),
        program: pumpProgram,
        global: PublicKey.findProgramAddressSync([Buffer.from('global')], pumpProgram)[0],
        feeRecipient: new PublicKey("68yFSZxzLWJXkxxRGydZ63C6mHx1NLEDWmwN9Lb5yySg")
    };
}

// ✅ Fetch Owner ATA
async function getOwnerAta(mint, pubkey) {
    return PublicKey.findProgramAddressSync(
        [pubkey.toBuffer(), spl.TOKEN_PROGRAM_ID.toBuffer(), mint.toBuffer()],
        spl.ASSOCIATED_TOKEN_PROGRAM_ID
    )[0];
}

// 🔄 Main Execution
await login();
setInterval(checkTweets, 5000);
