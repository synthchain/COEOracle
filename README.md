var properties = require('properties-reader')('./properties');
var cmcAPI = require('./cmcAPI');
const web3 = require('web3');
const Tx = require('ethereumjs-tx');
const web3js = new web3(new web3.providers.HttpProvider(properties.get('rpc.address')));

var latestTransactionHash;
var lastTransactionJSON;
const adminAddress = properties.get('admin.address').replace(/['"]+/g, '');
const privateKey = Buffer.from(properties.get('admin.pk'), 'hex')
const coeAddress = '0x37b21509D0dFdB33A83f70954bC567343C15Bdac';
const updateCMCABI = [{
  "constant": false,
  "inputs": [{
      "name": "_cmc",
      "type": "uint256"
    },
    {
      "name": "_btc",
      "type": "uint256"
    },
    {
      "name": "_eth",
      "type": "uint256"
    }
  ],
  "name": "updateCMC",
  "outputs": [],
  "payable": false,
  "stateMutability": "nonpayable",
  "type": "function"
}];

const contract = new web3js.eth.Contract(updateCMCABI, coeAddress);

function poll() {
  console.log('polling');
  cmcAPI.getGlobalCapData(function(err, data) {
    if (!err) {
      var total_cap, btc_cap, eth_cap;
      try {
        total_cap = Math.round(data.quote.USD.total_market_cap);
        btc_cap = Math.round(total_cap * data.btc_dominance / 100);
        eth_cap = Math.round(total_cap * data.eth_dominance / 100);
      } catch (error) {
        console.error(error);
        console.error("Failed to parse data.", data);
        return;
      }
      console.log(total_cap);
      console.log(btc_cap);
      console.log(eth_cap);

      var validationResponse = validateData(total_cap, btc_cap, eth_cap);

      if (!validationResponse.isValid) {
        console.error("Bad data.");
        console.error(validationResponse.error);
        return;
      }
      web3js.eth.getTransaction(latestTransactionHash, function(err, response) {
        if (!err && response.blockNumber == null) {
          console.log('Attempting to replace pending transacton.');
          // last transaction still pending; let's replace it.
          web3js.eth.getGasPrice().then(function(gasPrice) {
            var newGasPrice = Math.max([lastTransactionJSON * 1.1, gasPrice]) * 1.1;
            lastTransactionJSON.gasPrice = newGasPrice;
            lastTransactionJSON.data = contract.methods.updateCMC(total_cap, btc_cap, eth_cap).encodeABI();
            console.log(lastTransactionJSON);

            var transaction = new Tx(lastTransactionJSON);
            transaction.sign(privateKey);
            web3js.eth.sendSignedTransaction('0x' + transaction.serialize().toString('hex'))
              .on('transactionHash', setLatestTxHash);
          });
        } else {
          web3js.eth.getGasPrice().then(function(gasPrice) {
            web3js.eth.getTransactionCount(adminAddress).then(function(count) {
              //creating raw tranaction
              lastTransactionJSON = {
                "from": adminAddress,
                "gasPrice": web3js.utils.toHex(gasPrice * 1.5),
                "gasLimit": web3js.utils.toHex(100000),
                "to": coeAddress,
                "data": contract.methods.updateCMC(total_cap, btc_cap, eth_cap).encodeABI(),
                "nonce": web3js.utils.toHex(count)
              }
              console.log(lastTransactionJSON);
              //creating tranaction via ethereumjs-tx
              var transaction = new Tx(lastTransactionJSON);
              //signing transaction with private key
              transaction.sign(privateKey);
              //sending transacton via web3js module
              web3js.eth.sendSignedTransaction('0x' + transaction.serialize().toString('hex'))
                .on('transactionHash', setLatestTxHash);
            });
          });
        }
      });
    }
  });
}

function setLatestTxHash(hash) {
  console.log(hash);
  latestTransactionHash = hash;
}

function validateData(cmc, btc, eth) {
  var response = {};
  response.isValid = true;

  if (isNaN(cmc) || isNaN(btc) || isNaN(eth) || cmc <= 0 || btc <= 0 || eth <= 0) {
    response.isValid = false;
    response.error = "Something is NaN";
    console.log('cmc: ' + cmc);
    console.log('btc: ' + btc);
    console.log('eth: ' + eth);
    return response;
  }

  if (cmc <= btc + eth) {
    response.isValid = false;
    response.error = "BTC + ETH is >= total cap.";
    return response;
  }

  // for now we assume no apocalypse (we can always change this code)
  if (cmc < 1000000) {
    response.isValid = false;
    response.error = "No way CMC is that low! Data issue.";
    return response;
  }

  if (eth > btc) {
    console.log("Wow, congrats ETH!");
  }

  return response;
}

poll();
// 1 minute = 60000
setInterval(poll, 60000 * properties.get('COE.intervalMinutes'));
