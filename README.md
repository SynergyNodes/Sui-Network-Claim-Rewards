# Sui Network - Claim Rewards
JS script to claim Validator Rewards

```
import shell_exec from 'shell_exec';
var exec = shell_exec.shell_exec;

// Change the following path to Sui Binary
const sui_binary_path = "/root/sui"; 

// Specify the address where you want to download the rewards
const address = ""; // This is required.

// if you know the gasObj id that you want to use,  you can hard code this. Or else, leave it blank.
var gasObj = "";
var primary_coin = "";
var gas_budget = 20000000;


///////////// Get Staked Sui Objects and Withdraw stake ///////////////
///////////// Get objectId for Gas and store the Id in gasObj variable ///////////////

var jsonData = JSON.parse(exec(sui_binary_path + ' client objects --json'));

jsonData.forEach((item) => {
  if (item.data.type === "0x3::staking_pool::StakedSui") {
    console.log("objectId:", item.data.objectId);
    processObjectById(item.data.objectId);
  }
  else if (item.data.type === "0x2::coin::Coin<0x2::sui::SUI>" && gasObj === "") {
    gasObj = item.data.objectId;
    console.log("gasObj = ", gasObj);
  }  
});


///////////// Get all Coin Objects and merge them ///////////////
///////////// Exclude the primary_coin and gasObj ///////////////

jsonData = JSON.parse(exec(sui_binary_path + ' client objects --json'));

for (let i = 0; i < jsonData.length; i++) {
  const item = jsonData[i];
  if (item.data.type === "0x2::coin::Coin<0x2::sui::SUI>" && item.data.objectId != gasObj) {
    primary_coin = item.data.objectId;
    break;
  }
}

console.log("Primary Coin: ",primary_coin);

jsonData.forEach((item) => {
  if (item.data.type === "0x2::coin::Coin<0x2::sui::SUI>") {
    if (item.data.objectId != primary_coin && item.data.objectId != gasObj && item.data.content.fields.balance > gas_budget) {
      console.log("objectId:", item.data.objectId);
      mergeObjectById(item.data.objectId);
    }
  }
});



///////////// Deduct gas budget and send the rewards to specified address ///////////////

jsonData = JSON.parse(exec(sui_binary_path + ' client objects --json'));

jsonData.forEach((item) => {
  if (item.data.type === "0x2::coin::Coin<0x2::sui::SUI>") {
    if (item.data.content.fields.balance > gas_budget) {
      console.log("balance: ", item.data.content.fields.balance);
      const balance = item.data.content.fields.balance - gas_budget;
      exec(sui_binary_path + ' client transfer-sui --amount ' + balance + ' --to ' + address + ' --sui-coin-object-id ' + item.data.objectId + ' --gas-budget ' + gas_budget);
      console.log("Sent Rewards to ", address);
    }
  }
});




///////// Functions ///////////

function processObjectById(objectId) {
  exec(sui_binary_path + ' client call --package 0x3 --module sui_system --function request_withdraw_stake --args 0x5 ' + objectId + ' --gas-budget ' + gas_budget);
  console.log(objectId, " - Withdraw Stake Successful");
}

function mergeObjectById(objectId) {
  exec(sui_binary_path + ' client merge-coin --primary-coin ' + primary_coin + ' --coin-to-merge ' + objectId + ' --gas-budget ' + gas_budget);
  console.log(objectId, " - Merge Successful");
}
```
