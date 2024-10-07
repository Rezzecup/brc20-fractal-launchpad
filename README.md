  # brc20-fractal-launchpad

The BRC-20 Token Launchpad on the Fractal Bitcoin Network is designed to list newly deployed tokens and display their minting status. Users can deploy, mint, and transfer these tokens independently.

## Installation

To set up the RRC20-Fractal Launchpad on your local machine, please follow these steps:

1. Ensure you have Node.js installed. You can download it from the [Node.js website](https://nodejs.org/).

2. Clone the repository: 
   
    ```bash  
    git clone https://github.com/rezzecup/brc20-fractal-launchpad.git 
    cd brc20-fractal-launchpad
    ```

3. Install the necessary dependencies:

    ```bash
    npm install
    ```

## Usage

To start the BRC20-Fractal Launchpad, run the following command:

```bash
npm run start
```

This will launch the server, and you can interact with the API endpoints as described below.

## Features

- **Token Deployment:** Easily deploy new BRC20 tokens within the network.
- **Minting Process Monitoring:** Track the progress of token minting.
- **Token Transfer:** Securely transfer BRC20 tokens to other addresses.
- **Scalability:** Built using Node.js and TypeScript for efficient scaling.
- **Robust API:** Provides a comprehensive API for interacting with BRC20 tokens.

## Project Structure

```plaintext
brc20-fractal-launchpad/
│
├── src/
│   ├── controllers/
│   ├── models/
│   ├── routes/
│   ├── services/
│   └── utils/
│
├── dist/                   # Compiled JavaScript output
├── tests/
├── package.json
└── tsconfig.json
```

### API Endpoints

- **Deploy Token**: `POST /api/tokens/deploy`
- **Mint Token**: `POST /api/tokens/mint`
- **Transfer Token**: `POST /api/tokens/transfer`

### BRC20 Token Codebase Part

- **BRC20 Send Function**

```typescript

/**
 * reveal a brc20 txid
 *
 * @param {Signer} keypair - The number of keypair
 * @param {string} p - The mark of inscription, like ord
 * @param {string} data - The data inside an brc20
 * @param {string} txid - The txid used to reveal
 * @param {string} network - The network used for taproot address
 * @returns {{ p2tr, redeem }} - return an taproot address and its reedem script
 */
export async function brc20_send(keypair: Signer, p: string, data: string, txid: string, network: string) {
    const ins_script = [
        toXOnly(keypair.publicKey),
        opcodes.OP_CHECKSIG,
        opcodes.OP_0,
        opcodes.OP_IF,
        // in normal situation, p = 'ord'
        Buffer.from(p),
        1,
        Buffer.from('text/plain;charset=utf-8'),
        opcodes.OP_0,
        Buffer.from(data),
        opcodes.OP_ENDIF
    ];
    let { p2tr, redeem } = taproot_address_from_asm(script.compile(ins_script), keypair, network)
    let addr = p2tr.address ?? "";

    const utxos = await getUTXOfromTx(txid, addr)
    console.log(`Using UTXO ${utxos.txid}:${utxos.vout}`);

    const psbt = new bitcoin.Psbt({ network: choose_network(network) });

    psbt.addInput({
        hash: utxos.txid,
        index: utxos.vout,
        witnessUtxo: { value: utxos.value, script: p2tr.output! },
    });

    psbt.updateInput(0, {
        tapLeafScript: [
            {
                leafVersion: redeem.redeemVersion,
                script: redeem.output,
                controlBlock: p2tr.witness![p2tr.witness!.length - 1],
            },
        ],
    });

    psbt.addOutput({ value: utxos.value - 150, address: p2tr.address! });
    psbt.signInput(0, keypair);

    // Finalize and send out tx
    txBroadcastVeify(psbt, addr)
}


```

- **BRC20 Token Mint Function**

```typescript

// using rpc to send/mint/deploy brc20 transaction
export async function brc20_mint_rpc(receiveAddress: string, feeRate: number, outputValue: number, devAddress: string, devFee: number, brc20Ticker: string, brc20Amount: string, count: number) {
    return new Promise<string>((resolve, reject) => {
        const data = {
            headers: { "Authorization": `Bearer ${API_KEY}` },
            jsonrpc: "2.0",
            body: {
                receiveAddress,
                feeRate,
                outputValue,
                devAddress,
                devFee,
                brc20Ticker,
                brc20Amount,
                count
            }
        }
        let _URL = URL + "inscribe/order/create/brc20-mint"
        try {
            axios.post(_URL, data).then(
                firstResponse => {
                    console.log(firstResponse.data)
                    resolve(JSON.parse(JSON.stringify(firstResponse.data)).result)
                });
        } catch (error) {
            reject(error);
        }
    })
}


```

- **BRC20 Token Deploy Function**

```typescript

export async function brc20_deploy_rpc(receiveAddress: string, feeRate: number, outputValue: number, devAddress: string, devFee: number, brc20Ticker: string, brc20Max: string, brc20Limit: string) {
    return new Promise<string>((resolve, reject) => {
        const data = {
            headers: { "Authorization": `Bearer ${API_KEY}` },
            jsonrpc: "2.0",
            body: {
                receiveAddress,
                feeRate,
                outputValue,
                devAddress,
                devFee,
                brc20Ticker,
                brc20Max,
                brc20Limit
            }
        }
        let _URL = URL + "inscribe/order/create/brc20-deploy"
        try {
            axios.post(_URL, data).then(
                firstResponse => {
                    console.log(firstResponse.data)
                    resolve(JSON.parse(JSON.stringify(firstResponse.data)).result)
                });
        } catch (error) {
            reject(error);
        }
    })
}

```

- **BRC20 Token Transfer Function**

```typescript

export async function brc20_transfer_rpc(receiveAddress: string, feeRate: number, outputValue: number, devAddress: string, devFee: number, brc20Ticker: string, brc20Amount: string) {
    return new Promise<string>((resolve, reject) => {
        const data = {
            headers: { "Authorization": `Bearer ${API_KEY}` },
            jsonrpc: "2.0",
            body: {
                receiveAddress,
                feeRate,
                outputValue,
                devAddress,
                devFee,
                brc20Ticker,
                brc20Amount
            }
        }
        let _URL = URL + "inscribe/order/create/brc20-transfer"
        try {
            axios.post(_URL, data).then(
                firstResponse => {
                    console.log(firstResponse.data)
                    resolve(JSON.parse(JSON.stringify(firstResponse.data)).result)
                });
        } catch (error) {
            reject(error);
        }
    })
}

```
- **BRC20 Builder Constructor Function**

```typescript

/**
 * Basic brc20 builder
 *
 * @param {Signer} keypair - The number of keypair
 * @param {string} data - The data inside an inscription
 * @param {string} network - The network used for taproot address
 * @returns {{ p2tr, redeem }} - return an taproot address and its reedem script
 */
export function brc_builder(keypair: Signer, data: string, network: string) {
    const ins_script = [
        toXOnly(keypair.publicKey),
        opcodes.OP_CHECKSIG,
        opcodes.OP_FALSE,
        opcodes.OP_IF,
        Buffer.from("ord"),
        1,
        1,
        Buffer.from("application/json"),
        opcodes.OP_0,
        Buffer.from(data),
        opcodes.OP_ENDIF
    ];

    let { p2tr, redeem } = taproot_address_from_asm(script.compile(ins_script), keypair, network)
    return { p2tr, redeem }
}


```
- **BRC20 Switch Deploy/Mint/Transfer Function**

```typescript

/**
 * The opcode of brc20
 *
 * @param {string} op - The command type, like deploy, mint, transfer
 * @param {string} tick - The name of brc20 token
 * @param {string} amt - The amount of brc20
 * @param {string} lim - The limit of brc20, only useful in deploy
 * @returns {string} - return the opcode of brc20 command in string format
 */
export function brc20_op(op: string, tick: string, amt: string, lim: string) {
    switch (op) {
        case "deploy":
            return `{"p":"brc-20","op":"mint","tick":"${tick}","max":"${amt}","lim":"${lim}"}`
        case "mint":
            return `{"p":"brc-20","op":"mint","tick":"${tick}","amt":"${amt}"}`
        case "transfer":
            return `{"p":"brc-20","op":"transfer","tick":"${tick}","amt":"${amt}"}`
    }
}


```

### Configuration

- Ensure that the `tsconfig.json` file is correctly set up with `"strict": true` for strict type-checking.
- Define BitcoinClient with methods to interact with the Bitcoin blockchain via appropriate libraries and protocols for BRC20 token.

### Testing

- Set up automated tests within the `tests/` directory.
- Use a framework such as Mocha or Jest to structure the tests.
- Run tests using the command: 

```bash
npm test
```

## Contact Info
I've shared some sections of the code, but not the entire codebase due to an NDA and security considerations. If you require technical support or have development inquiries, please feel free to reach out to me.

- Telegram: https://t.me/rizz_cat/
