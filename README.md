# txFinalized()
An async method to detect if a Solana transaction is finalized or dropped.

Assumes you have solanaWeb3 in global scope.

Takes into account all three commitment statuses for Solana transactions as outlined in Solana docs as well as initial null responses and null responses between status changes. 

[Solana Commitment Statuses](https://docs.solanalabs.com/consensus/commitments)

# Usage
1. RPC Endpoint URL (string)
2. Transaction Signature (string)
3. Max Attempts (integer)
4. Pause Between Attempts (integer)
```javascript

// wait for a response from Solana for this signature, check 40 times max, and pause 4 seconds between checks.
let status = await txFinalized("YOUR_RPC_URL", "SOLANA_TX_SIGNATURE", 40, 4);

// if a status other than finalized is returned, handle the error and exit.
if(status != "finalized"){
  console.log("txFinalized Error: "+status);
  return;
}

// otherwise once finalized, do more stuff.
console.log("Doing more stuff!");

```

```javascript

// example with 3 retry attempts

let cluster = "YOUR_RPC_URL";
let response = {};
try {
  mintToCollectionTx.sign([SystemFeeSigner]);
  let txId = await connection.sendTransaction(mintToCollectionTx);
  let final = await txFinalized(cluster,txId,40,4);
  if(final != "finalized"){
    await new Promise(_ => setTimeout(_, 3000));
    let txId = await connection.sendTransaction(mintToCollectionTx);
    let final = await txFinalized(cluster,txId,40,4);
    if(final != "finalized"){
      await new Promise(_ => setTimeout(_, 3000));
      let txId = await connection.sendTransaction(mintToCollectionTx);
      let final = await txFinalized(cluster,txId,40,4);
      if(final != "finalized"){
        response.status = "error";
        response.signature = txId;
        response.data = "Bad Status: "+final;
        response.notes = "Failed after 3 retries!";
        console.log(response);
      }
    }
  }
  response.status = "final";
  response.signature = txId;
  response.data = "Complete";
  console.log(response);
}
catch(error) {
  response.status = "error";
  response.signature = false;
  response.data = "Failed to Send!";
  response.log = error;
  console.log(response);
}

```

# Method

```javascript
async function txFinalized(cluster, sig, max = 40, seconds = 4){
  return await new Promise(resolve => {
    let start = 1;
    seconds = (seconds * 1000);
    let connection = new solanaWeb3.Connection(cluster, "confirmed");
    let intervalID = setInterval(async()=>{
      console.log(start+": "+sig);
      let tx_status = await connection.getSignatureStatuses([sig], {searchTransactionHistory: true,});
      console.log(tx_status.value[0]);
      if(start > 20 && tx_status.value[0] == null){
        clearInterval(intervalID);
        console.log('Oh No! Something Happened!');
        resolve('Oh No! Something Happened!');
      }
      if (typeof tx_status == "undefined" || 
      typeof tx_status.value == "undefined" || 
      tx_status.value == null || 
      tx_status.value[0] == null || 
      typeof tx_status.value[0] == "undefined" || 
      typeof tx_status.value[0].confirmationStatus == "undefined"){} 
      else if (tx_status.value[0].confirmationStatus == "processed"){
        console.log('Transaction Processed!');
        start = 1;
      }
      else if(tx_status.value[0].confirmationStatus == "confirmed"){
        console.log('Transaction Confirmed!');
        start = 1;
      }
      else if (tx_status.value[0].confirmationStatus == "finalized"){
        console.log('Transaction Complete!');
        clearInterval(intervalID);
        resolve('finalized');
      }
      start++;
      if(start==max){
        clearInterval(intervalID);
        resolve(max+' max retrys reached');
      }
    }, seconds);
  });  
}
```

# Details

• This js method will attempt to get an initial Solana transaction status of a signature up to 20 times with a Promise.

• If any status other than null is returned from Solana it will continue to wait for a "finalized" status up to the max.

• If/when a "finalized" status is return from Solana, the method returns "finalized" as a string.

• Any other status returned should be gracefully handled as an error.

• In some cases Solana returns "processed" status. When this happens the counter is reset to 1 and it will continue checking for a "finalized" status up to the max attempts.
