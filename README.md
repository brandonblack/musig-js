# MuSig in TypeScript

Zero dependency implementation of the [MuSig Spec](https://github.com/ElementsProject/secp256k1-zkp/blob/master/doc/musig-spec.mediawiki).

Similar to [bitcoinjs/BIP32](https://github.com/bitcoinjs/bip32), requires a
user injected secp256k1 implementation. Two examples provided in
`test/utils.ts`.

Works with ECMAScript 2019 if you provide a complete implementation of the
Crypto interface. If you want to use `base_crypto` as `test/utils.ts` does,
then requires ECMAScript 2020 BigInt.

# Usage

```shell
npm i
npm run build
npm run test
npm run bench
```

This example uses `Buffer` for convenience, but it is not required, any
`Uint8Array` will do.

```typescript
import { MuSigFactory } from '.';
import { tinyCrypto } from './test/utils';
import * as tiny from 'tiny-secp256k1';

const musig = MuSigFactory(tinyCrypto);

const fromHex = (hex: string): Uint8Array => Buffer.from(hex, 'hex');
const toHex = (bytes: Uint8Array): string => Buffer.from(bytes).toString('hex');
const secretKey1 = fromHex('be01d8dcf3879a0fec05130ca95d35bf7823833e3cdf91e310408606717055d9');
const pubKey1 = fromHex('648c0c80c8520875c22a1cf31cd718b72a50e381731bc7f8efec9944074cb21b');
const secretKey2 = fromHex('644b07a5cb70f68316cd9a51cabdc61c4d0b1f38b189d0c92370a3844fd0241f');
const pubKey2 = fromHex('8c9444613a1bb8b442b81cc7fbf81a186a34d7c4e596362543e17dde3efdc4b3');
const tweak = fromHex('8df63a82e5e71884bb16e2896e12ba2b7fe0e670d466be03b578fc435d5c9876');

const { publicKey: aggregatePublicKey, keyAggSession } = musig.keyAgg(
  [pubKey1, pubKey2],
  { tweaks: [tweak], tweaksXOnly: [true] }
);

const msg = fromHex('f1d1d6ef2d97319149aaed92c69ebb21d6c54c0fc4e908f4f4ee42a1e5b8b854');

// Signing round 1 - generate nonces, share public nonces

const nonce1 = musig.nonceGen({
  sessionId: fromHex('0000000000000000000000000000000000000000000000000000000000000001'),
  secretKey: secretKey1,
  msg,
  aggregatePublicKey
});

const nonce2 = musig.nonceGen({
  sessionId: fromHex('0000000000000000000000000000000000000000000000000000000000000001'),
  secretKey: secretKey2,
  msg,
  aggregatePublicKey
});

const sharedPublicNonces = [nonce1.publicNonce, nonce2.publicNonce];

const aggNonce = musig.nonceAgg(sharedPublicNonces);

// Signing round 2 - generate and share partial sigs, verify and aggregate

const { sig: sig1, signingSession: signingSession1 } = musig.partialSign({
  msg,
  secretKey: secretKey1,
  nonce: nonce1,
  aggNonce,
  keyAggSession
});

const { sig: sig2, signingSession: signingSession2 } = musig.partialSign({
  msg,
  secretKey: secretKey2,
  nonce: nonce2,
  aggNonce,
  keyAggSession
});

const check2By1 = musig.partialVerify({
  sig: sig2,
  msg,
  publicKey: pubKey2,
  publicNonce: sharedPublicNonces[1],
  aggNonce,
  keyAggSession,
  signingSession: signingSession1
});

const check1By2 = musig.partialVerify({
  sig: sig1,
  msg,
  publicKey: pubKey1,
  publicNonce: sharedPublicNonces[0],
  aggNonce,
  keyAggSession,
  signingSession: signingSession2
});

console.log(`check2By1: ${!!check2By1}, check1By2: ${!!check1By2}`);

const sigBy1 = musig.signAgg([sig1, sig2], signingSession1);

const sigBy2 = musig.signAgg([sig2, sig1], signingSession2);

console.log(toHex(sigBy1));
// 13a8d88bb5727fe945293f81f0f1000eecb8ded5ca950bcfb74d6536d456372b9ae00ccb9cbacc00a3bca07129920b88d4df4f5c24ece1f7159ff94c1dde1bba
console.log(toHex(sigBy1) === toHex(sigBy2));

const valid = tiny.verifySchnorr(msg, aggregatePublicKey, sigBy1);
console.log(`final sig valid: ${valid}`);
```
