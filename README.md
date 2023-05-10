# reduced-transactions
source code converting EIP12UnsignedTx to Base64 encoded reduced Transaction

```ts
const unsignedTransaction = new TransactionBuilder(height)
      .from(inputs)
      .to(outputs)
      .sendChangeTo(changeAddress)
      .payMinFee()
      .build()
      .toEIP12Object();
const { inputs, dataInputs } = unsignedTransaction;
const [txId, ergoPayTx] = await getTxReducedB64Safe(unsignedTransaction,inputs,dataInputs);
const ergoPayUrl = `ergopay:${ergoPayTx}`; // turn this into a qr-code or open in browser to redirect to ergopay wallet
```

```ts
import JSONBigInt from 'json-bigint';

const ergolib = import('ergo-lib-wasm-browser');
const DEFAULT_EXPLORER_API_ADDRESS = "https://api.ergoplatform.com/";
const explorerApiV1 = DEFAULT_EXPLORER_API_ADDRESS + 'api/v1';


export async function getTxReducedB64Safe(json, inputs, dataInputs) {
    /*
    * Name: getTxReducedB64Safe
    * Type: async function
    * Description:  creates a ReducedTransaction object from a json transaction and encodes it with Base64
    * Parameters:
    * json: object
    * inputs: object
    * dataInputs: object (default [])
    * Returns:
    * Promise<any>
    * Dependencies:
    * getTxReduced: function
    * byteArrayToBase64: function
    * ErgoLib: import
    * Comments:
    * - returns an array of the transaction id and a ReducedTransaction object encoded with Base64
    * - used to create a ReducedTransaction object
    * - used to create a QR code
    * - used to create a payment link
    */
    
    let txId: string|null = null ;
    let reducedTx;

    try {
        [txId, reducedTx] = await getTxReduced(json, inputs, dataInputs);
    } catch (e) {
        console.log("error", e);
    }
    
    // Reduced transaction is encoded with Base64
    const txReducedBase64 = byteArrayToBase64(reducedTx.sigma_serialize_bytes());

    const ergoPayTx = txReducedBase64.replace(/\//g, '_').replace(/\+/g, '-');
    //console.log("getTxReducedB64Safe3", txId, ergoPayTx);
    // split by chunk of 1000 char to generates the QR codes

    return [txId, ergoPayTx];
}

export async function getUnspentBoxesByAddress(address: string, limit = 50) {
    /*
    * Name: getUnspentBoxesByAddress
    * Type: async function
    * Description:  gets unspent boxes by address
    * Parameters:
    *   address: string
    *   limit: number (default 50)
    * Returns: 
    *   Promise<any>
    * Comments: 
    *   - address must be a valid ergo address
    *   - limit is the maximum number of boxes to return
    */

    //todo: add error handling
    const data = await getRequestV1(`/boxes/unspent/byAddress/${address}?limit=${limit}`);
    return data.items;
}

export async function currentHeight() {
    /*
    * Name: currentHeight
    * Type: async function
    * Description: gets the current height of the blockchain
    * Parameters:
    *  none
    * Returns:
    *  Promise<number>
    * Todo:
    * - add error handling
    * - add caching
    * - use node instead of explorer
    */

    const url = '/blocks?limit=1';
    const res = await get(explorerApiV1 + url);
    const data = await res.json();
    const height = data.items[0].height;
    return height;
}

function byteArrayToBase64( byteArray ) {
    /*
    * Name: byteArrayToBase64
    * Type: function
    * Description:  converts a byte array to a base64 string
    * Parameters:
    * byteArray: object
    * Returns:
    * string
    * Dependencies:
    * none
    */
  
    var binary = '';
    var len = byteArray.byteLength;
    for (var i = 0; i < len; i++) {
        binary += String.fromCharCode( byteArray[ i ] );
    }
    return window.btoa( binary );
}

async function getTxReduced(json, inputs, dataInputs): Promise<[string, any]>{
    /*
    * Name: getTxReduced
    * Type: async function
    * Description:  creates a ReducedTransaction object from a json transaction
    * Parameters:
    *  json: object
    * inputs: object
    * dataInputs: object (default [])
    * Returns:
    * Promise<any>
    * Dependencies:
    * getErgoStateContext: function
    * ErgoLib: import
    * Comments:
    * - returns an array of the transaction id and a ReducedTransaction object
    * - used to create a ReducedTransaction object
    */
    
    const unsignedTx =  (await ergolib).UnsignedTransaction.from_json(JSONBigInt.stringify(json));
    console.log("unsignedTx", unsignedTx);
    const inputBoxes = (await ergolib).ErgoBoxes.from_boxes_json(inputs);
    console.log("inputBoxes", inputBoxes);
    const inputDataBoxes = (await ergolib).ErgoBoxes.from_boxes_json(dataInputs);
    console.log("inputDataBoxes xxxx", inputDataBoxes);
    let ctx:any;

    try {
        ctx = await getErgoStateContext();
    } catch (e) {
        console.log("error", e);    
    }
    const id = unsignedTx.id().to_str();
    const reducedTx = (await ergolib).ReducedTransaction.from_unsigned_tx(unsignedTx, inputBoxes, inputDataBoxes, ctx);
    return [id, reducedTx];
}

async function getErgoStateContext(): Promise<any> {
    /*
    * Name: getErgoStateContext
    * Type: async function
    * Description:  gets the current state context of the ergo blockchain from the explorer
    * Parameters:
    *  none
    * Returns:
    * Promise<any>
    * Dependencies:
    * getExplorerBlockHeaders: function
    * ErgoLib: import
    * Comments:
    * - returns an ErgoStateContext object  
    * - used to create a ReducedTransaction object
    */

    let explorerHeaders: any = [];
    try {
        explorerHeaders = await getExplorerBlockHeaders();
    } catch (e) {
        console.log("error", e);
    }
    console.log("explorerHeaders", explorerHeaders);
    const block_headers = (await ergolib).BlockHeaders.from_json(explorerHeaders);
    const pre_header = (await ergolib).PreHeader.from_block_header(block_headers.get(0));
    const ctx = new (await ergolib).ErgoStateContext(pre_header, block_headers);
    return ctx;
}

async function getExplorerBlockHeaders() {

    /*
    * Name: getExplorerBlockHeaders
    * Type: async function
    * Description: gets block headers from the explorer
    * Parameters:
    *   none
    * Returns: 
    *   Promise<any>
    * Comments: 
    *  - returns an array of 10 most recent block headers
    */

    let res: any = [];
    let headers = [];
    try {
        res = await getRequestV1(`/blocks/headers`);
        headers = res.items.slice(0, 10);
    } catch (e) {
        console.log("error", e);
    }
    return headers;
}

async function getRequestV1(url: string) {
    /*
    * Name: getRequestV1
    * Type: function
    * Description: makes a GET request to the explorer api
    * Parameters:
    *   url: string
    * Returns: 
    *   Promise<any>
    */
    const res = await get(explorerApiV1 + url);
    const data = await res.json();
    console.log("data:", data);
    return data;
}

async function get(url: string, apiKey = '') {
    return await fetch(url, {
        headers: {
            Accept: 'application/json',
            'Content-Type': 'application/json',
            api_key: apiKey,
        },
    });
}

async function post(url: string, body = {}, apiKey = '') {
    return await fetch(url, {
        method: 'POST',
        headers: {
            Accept: 'application/json',
            'Content-Type': 'application/json',
            api_key: apiKey,
        },
        body: JSON.stringify(body),
    });
}
```
