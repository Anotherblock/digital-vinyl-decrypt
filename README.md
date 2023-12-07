# decrypt music

this is a guide to play your music that is encrypted onchain.

## requirements

- bun runtime
- anotherblock music nft
- wallet private key

## steps

1. start by creating a new project (`bun init` etc)
2. install the following packages:

```sh
bun add viem siwe @lit-protocol/lit-node-client
```

3. create a `.env` and put your private key in the `PRIVATE_KEY`
4. find the url to the content metadata. you'll find this in the nft metadata in the `content` property. e.g ich r u - boys noize: [`https://arweave.net/vO3M5ALVROim8M-yRlQgt7jb7b7rc1c5mHYkigrg-dw`](https://arweave.net/vO3M5ALVROim8M-yRlQgt7jb7b7rc1c5mHYkigrg-dw)
5. download this file to `content-metadata.json` in the project directory.

```sh
curl -o content-metadata.json https://arweave.net/vO3M5ALVROim8M-yRlQgt7jb7b7rc1c5mHYkigrg-dw -L
```

6. now, create a new file called `decrypt.ts` with the following contents:

```ts
import { decryptFile, LitNodeClient } from '@lit-protocol/lit-node-client';
import { SiweMessage } from 'siwe';
import { createWalletClient, Hex, http } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { base } from 'viem/chains';

import contentMetadata from './content-metadata.json';

const litClient = new LitNodeClient({});

const account = privateKeyToAccount(process.env.PRIVATE_KEY as Hex);

const walletClient = createWalletClient({
  account,
  transport: http(base.rpcUrls.default.http[0])
});

const { chainId, domain, statement, version } = contentMetadata.config;

const siweMessage = new SiweMessage({
  domain,
  address: account.address,
  statement,
  uri: `https://${domain}`,
  chainId,
  version
});

const message = siweMessage.prepareMessage();

const signature = await walletClient.signMessage({ message });

const authSig = {
  sig: signature,
  derivedVia: 'web3.eth.personal.sign',
  signedMessage: message,
  address: account.address
};

await litClient.connect();

const encryptionKey = await litClient.getEncryptionKey({
  toDecrypt: contentMetadata.encryptedSymmetricKey,
  authSig,
  chain: contentMetadata.config.chain,
  accessControlConditions: contentMetadata.acc
});

const fileResponse = await fetch(contentMetadata.fileUrl);

const file = new Blob([await fileResponse.arrayBuffer()]);

const decryptedFile = await decryptFile({
  file,
  symmetricKey: encryptionKey
});

Bun.write(`music.mp3`, Buffer.from(decryptedFile));
```

7. run the file with `bun run decrypt.ts`
8. enjoy the music in `music.mp3`
