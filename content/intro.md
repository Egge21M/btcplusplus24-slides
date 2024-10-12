## cashu-ts

cashu-ts is a typescript library that helps interacting with a mint

- takes care of the underlying cryptography
- provides a declarative api for the protocol

---

## setting up

to set up the practice project

```sh [1|2|3|4]
git clone https://github.com/Egge21M/btcplusplus24-code
cd btcplusplus24-code
git checkout exercise
npm install
```

---

to install cashu-ts

```sh
npm i @cashu/cashu-ts
```

---

## basic api

mainly cashu-ts exports two classes and some utilities

```ts [1|4-5]
import { CashuWallet, CashuMint, getEncodedToken } from "@cashu/cashu-ts";

const mintUrl = "https://testnut.cashu.space";
const mint = new CashuMint(minutUrl);
const wallet = new CashuWallet(mint);

const res = await wallet.mintToken(21, "12345");
const encoded = getDecodedToken(someToken);
```

---

## learn: information about a mint

you can obtain information about a mint by calling it's /info endpoint.
it returns things like:

- name
- description
- contact info
- supported NUTs

---

## try: src/commands/info.ts

in cashu-ts there are two methods that do this for you

```ts [2,5 | 3,6]
const mintUrl = "https://testnut.cashu.space";
const mint = new CashuMint(mintUrl);
const wallet = new CashuWallet(mint);

const info = await mint.getInfo();
const info = await wallet.getMintInfo();
```

---

## learn: minting a token

in cashu token creation is called **minting**.
minting is a three step process:

- user requests a token quote, mint responds with a payment request (quote)
- user pays the quote
- user requests the token

---

## try: src/commands/mint.ts

in cashu-ts this is as easy as

```ts [4|7]
const mint = new CashuMint("https://testnut.cashu.space");
const wallet = new CashuWallet(mint);

const quoteRes = await wallet.createMintQuote(21);
// at this point the payment needs to be made
// quoteRes contains the quote ID as well as the payment request
const { proofs } = await wallet.mintTokens(21, quoteRes.quote);
```

---

## extra: checking wether a quote has been paid

```ts [1|2-6]
const quoteRes = await wallet.createMintQuote(21);
// put this logic in a loop
const { state } = await wallet.checkMintQuote(quoteRes.quote);
if (state === "PAID") {
  // token has been paid
}
```

---

## learn: anatomy of a cashu token

a cashu token contains the cryptographic proof of ownership, as well as some metadata

```ts
{
  memo: "This is the token memo",
  token: [
    {
      mint: "https://testnut.cashu.space",
      proofs: [
        {
          amount: 1,
          C: "03ee437e0240b350fa3391cb061f6282b3848b522ca149c5b67a8e5c7add3f9e7b",
          id: "009a1f293253e41e",
          secret:
            "796e55657bfca49346dc735122bcf53a41bd08caec44e63017b7b770406ba221",
        },
      ],
    },
  ],
};
```

---

## learn: cashu token encoding

typically users encounter encoded token data

<br/>
JSON serialised and base64 encoded (v3)

```
cashuAeyJ0b2tlbiI6W3sibWludCI6Imh0dHBzOi8vODMzMy5zcGFjZTozMzM4IiwicHJvb2ZzIjpbeyJhbW91bnQiOjIsImlkIjoiMDA5YTFmMjkzMjUzZTQxZSIsInNlY3JldCI6IjQwNzkxNWJjMjEyYmU2MWE3N2UzZTZkMmFlYjRjNzI3OTgwYmRhNTFjZDA2YTZhZmMyOWUyODYxNzY4YTc4MzciLCJDIjoiMDJiYzkwOTc5OTdkODFhZmIyY2M3MzQ2YjVlNDM0NWE5MzQ2YmQyYTUwNmViNzk1ODU5OGE3MmYwY2Y4NTE2M2VhIn0seyJhbW91bnQiOjgsImlkIjoiMDA5YTFmMjkzMjUzZTQxZSIsInNlY3JldCI6ImZlMTUxMDkzMTRlNjFkNzc1NmIwZjhlZTBmMjNhNjI0YWNhYTNmNGUwNDJmNjE0MzNjNzI4YzcwNTdiOTMxYmUiLCJDIjoiMDI5ZThlNTA1MGI4OTBhN2Q2YzA5NjhkYjE2YmMxZDVkNWZhMDQwZWExZGUyODRmNmVjNjlkNjEyOTlmNjcxMDU5In1dfV0sInVuaXQiOiJzYXQiLCJtZW1vIjoiVGhhbmsgeW91LiJ9
```

CBOR and base64 encoded (v4)

```
cashuBpGF0gaJhaUgArSaMTR9YJmFwgaNhYQFhc3hAOWE2ZGJiODQ3YmQyMzJiYTc2ZGIwZGYxOTcyMTZiMjlkM2I4Y2MxNDU1M2NkMjc4MjdmYzFjYzk0MmZlZGI0ZWFjWCEDhhhUP_trhpXfStS6vN6So0qWvc2X3O4NfM-Y1HISZ5JhZGlUaGFuayB5b3VhbXVodHRwOi8vbG9jYWxob3N0OjMzMzhhdWNzYXQ=
```

---

## try: src/commands/token.ts

you can de/encode token in cashu-ts using a couple utility functions

```ts [1|3-4]
const decodedToken = getDecodedToken(token);

const v4Token = getEncodedTokenV4(rawTokenData);
const v3Token = getEncodedToken(rawTokenData);
```

---

## extra: return an encoded token from mintHandler

take the proofs received from the mint and encode them into an encoded token

```ts
const { proofs } = await wallet.mintTokens(21, quoteRes);
const tokenData: Token = {
  memo: "demo",
  token: [{ mint: "https://testnut.cashu.space", proofs }],
};
console.log(getEncodedToken(tokenData));
```

---

## learn: sending a cashu token

sending a cashu token is as easy as sending someone the proofs that make up the token

however, as cashu proofs have fixed denominations, we might need to swap our proofs first, in order to get the right amount

---

## example

alice has a single 64 sat proof and wants to pay bob 33 sats.

alice would first contact the mint and swap her 64 sat proof for many smaller ones:

32,16,8,4,2,1,1

---

## try: src/commands/send.ts

in cashu-ts the send method takes care of this automatically

```ts [4]
const mint = new CashuMint(mintUrl);
const wallet = new CashuWallet(mint);

// send contains the proofs that should be sent
const { returnChange, send } = await wallet.send(9, myProofs);
```

---

## learn: receive a cashu token

in order to prevent double spendings we can not simply store a received token.

first we got to invalidate the old token, by swapping it for a new one with the mint

---

## try: receive a cashu token

call the receive method to swap token automatically

```ts [4]
const mint = new CashuMint(mintUrl);
const wallet = new CashuWallet(mint);

const newProofs = await wallet.receive(proofs);
```

---

## learn: melt a cashu token

spending a cashu token to pay a lightning invoice is called melting. Just like minting it is a multi step process.

- send the payment request to the mint to get a quote
- pay the quote

the quote will include the **fee_reserve** that needs to be added to pay for lightning fees

---

## try: src/commands/melt.ts

cashu-ts handles lightning fees for you, but there is some work required

```ts [4 | 5 | 6 | 7 | 9-14]
const mint = new CashuMint(mintUrl);
const wallet = new CashuWallet(mint);

const quote = await wallet.createMeltQuote(invoice);
const totalAmount = quote.fee_reserve + invoiceAmount;
const { returnChange, send } = await wallet.send(totalAmount, send);
const payRes = await wallet.meltTokens(quote, proofsToSpend);

if (payRes.isPaid) {
  // you successfully paid the invoice
}

// a mint might return some fee change
console.log(payRes.change);
```

---

## call to action: get involved

cashu-ts is a group effort and we are always looking for feedback and contributors!

https://github.com/cashubtc/cashu-ts
