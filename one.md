There are broadly 3 main components you will need in any general MEV strategy:

1. Data Feed
2. Profitability Logic
3. Execution and Monitoring

# Setup
Ultimately, you'll need two things to run a strategy. A connection to an Ethereum full node and a computer capable of running your scripts.

There are many ways in which you can acquire a connection to a node. Below are a few methods and some corresponding remarks:
| Method | Convenience | Cost | Competitiveness | Notes |
| ----------- | ----------- | ----------- | ----------- | ----------- |
| [Infura](https://infura.io/)/[Alchemy](https://www.alchemy.com/)/Centralized Provider | Very | None | Not | Low overhead, potential rate-limiting if too many queries |
| [Blocknative](https://www.blocknative.com/) | Very | High | Decent | Good option if you have money to blow |
| [Quicknode](https://www.quicknode.com/) | Medium | Medium | Decent | Provision your own node without dealing with sysadmin | 
| Run your own | Awful | Medium | Best | This is what you'll have to do when you productionize strategies. No rate limiting, fastest queries, infinite customizability |

If you are just getting started. Just sign up for a free account with Infura or Alchemy. Chances are you'll hit their rate limits for the free tier quickly so I recommend making an account on each and just switching rpc urls when you hit your daily limit.

Now all you'll need is to be able to query the blockchain state programatically. Head over to [web3py](https://web3py.readthedocs.io/en/stable/quickstart.html) and setup your node connection in python:

```
from web3 import Web3

# HTTPProvider:
w3 = Web3(Web3.HTTPProvider('http://127.0.0.1:8545'))
# WebsocketProvider:
w3 = Web3(Web3.WebsocketProvider('wss://127.0.0.1:8546'))
w3.isConnected() == True
```
Obviously replace the urls with the url provided to you by Infura/Alchemy.

Side note, HTTP and Websocket are different web transport protocols that most web3 providers both provide. A good (potentially inaccurate) way to think about the difference is: http consists of a request-reply pattern. That means every request needs to initialize a network handshake and wait for a response. In contrast, a websocket connection consists of an always-on stream of data between the requester and replier. This generally means websocket connections are faster and less latent. Use websocket if given an option to.

For the rest of this guide, we will assume you have a successfully instantiated node connection with the name: `w3` 

# Data Feed
The first step in any mev strategy will require getting access to on chain data. For the purposes of this exercise we will be building a Uniswap V2 data feed.

### Step 1: Find Uniswap V2 Pairs
Naively, you could simply go to v2.info.uniswap.org/pairs and click on a few pairs to get the pair contract for each trading pair. Obviously this method will not scale. 

The Uniswap V2 factory is responsible for the creation of every single Uniswap V2 trading pair. Luckily enough, the [factory contract](https://etherscan.io/address/0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f) exposes two convenient functions to allow us to find every single Uniswap V2 pair ever created:

```
function allPairs(uint256) external view returns (address pair);
function allPairsLength() external view returns (uint256);
```
[docs](
https://docs.uniswap.org/protocol/V2/reference/smart-contracts/factory#allpairs)

As their names suggest, these functions will allow you to query an array consisting of the address of every single pair ever deployed. Now all you need to do is query them all:

```
uniswapFactory=w3.eth.contract(<factoryAddress>, abi=<factoryABI>)

allPairs=[]
for i in range(uniswapFactory.functions.allPairsLength().call()):
    pair = uniswapFactory.functions.allPairs(i).call()
    allPairs.append(pair)
```

A couple notes: You may notice in the first line we instantiated a web3 contract object consisting of the address of the factory contract and the abi. ABI stands for Application Binary Interface. It describes the standardized manner in which smart contracts on Ethereum pass data among each other. How would the web3py library know that the factory smart contract contains the function named allPairs() that takes a single argument of type uint256? That is something specified in the abi that we pass into when instantiating the smart contract. If you want to dive deep look [here](https://docs.soliditylang.org/en/latest/abi-spec.html).

The abi of the factory contract is located [here](https://unpkg.com/@uniswap/v2-core@1.0.0/build/IUniswapV2Factory.json). You can also find the abi for *any* verified contract on Etherscan by clicking the [code tab](https://etherscan.io/address/0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f#code) and scrolling to where it says "Contract ABI"

To load the abi, either download the .json file or copy and paste it into a local file and load it into your program like so:

```
import json
factoryABI = json.load(open('/path to file/factory.json'))
```
Now you may quickly realize doing this manually may become quite painful. Luckily Etherscan has an api which allows you to programatically query the abi of any verified contract. An implementation is out of scope for this guide but details can be found [here](https://docs.etherscan.io/api-endpoints/contracts).

Another thing you may realize is that Uniswap currently has tens of thousands of pairs created. The loop written above will likely take a very long time to complete. We'll cover more advanced patterns for mass querying of data later but for now limit your query to 1000 pairs.

Moving on, we finally begin querying price data of each uniswap pair.

### Step 2: Get Price of a Uniswap Pair

From the Uniswap V2 whitepaper, we can define the instantaneous spot price (i.e., how much of Token 0 will I get for swapping X Token 1 as X -> 0) as Token 0 Reserve Balance/Token 1 Reserve Balance. For example, let's say there are 10 of Token 0 and 2 of Token 1 in the Uniswap Pair. That means the price of Token 1 wrt Token 0 is Reserve0/Reserve1 = 10/2 = 5 or Token 1 is worth 5 of Token 0 (We will cover how to calculate price impact in the next section).

Every single Uniswap V2 pair has an identical ABI. The function signatures of one pair are exactly the same as the other (obviously the function outputs will not have the same values). There is however only 1 function we care about:

```
function getReserves() external view returns (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast);
```
[docs](
https://docs.uniswap.org/protocol/V2/reference/smart-contracts/pair#getreserves)

As the name suggest, calling this function will return 3 elements: the balance of token0, the balance of token1 and the timestamp of the last trade that has changed the reserve balances respectively.

```
uniswapPair=w3.eth.contract(<pairAddress>, abi=<pairABI>)
r0, r1, timestamp = uniswapPair.functions.getReserves().call()
token0SpotPrice = r1/r0
token1SpotPrice = r0/r1
```
Try this with the WETH-USDC [Uniswap V2 pair](https://v2.info.uniswap.org/pair/0xb4e16d0168e52d35cacd2c6185b44281ec28c9dc).
