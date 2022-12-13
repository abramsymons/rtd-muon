#######################
TWAPER (Price Feed App)
#######################

This is a technical document describing how a price feed app is developed using Muon. This app has two capabilities:

  - Calculating the TWAP of a normal ERC-20 token
  - Calculating the TWAP of an LP token

This document is made up of three sections: 

   1- In the first section, we describe how a module is developed to calculate Time Weighted Average Price (TWAP) of a pair of tokens based on the information it gets from a Uniswap pool or one of its forks. 
   
   2- In this section, we explain how to use the module from section one to obtain a token’s price from routes made of pairs that end with a stablecoin. These routes can be in different exchanges on different chains.
   
   3- In section 3, we demonstrate how the price of an LP token is calculated using the procedure in section 2.  

**************************
Calculating TWAP of a Pair
**************************

Our off-chain implementation of TWAP has the following benefits over the original on-chain TWAP of Uniswap.

  - It detects and removes outlier prices before calculating averages to prevent price manipulations through applying a sharp rise/fall in the price for a short duration.
  - In order to reject unexpected price changes, it applies a fuse mechanism that stops the system when a short duration average shows a large price volatility compared to a longer one.
  - It does not require periodic transactions to register checkpoints on-chain which are costly and hard to maintain.

Here is the technical details of how the module is implemented:

Obtaining Price Changes
=======================

To calculate TWAP, a time period is defined with a source and a destination time. Also, the token prices for the defined period should be obtained from each block. It seems that the app needs to call ``getReserves`` for each block, calculate the price for the block by dividing ``_reserve1`` to ``_reserve0``, and calculate the average of all the prices. 

Following such an approach is a time-consuming and costly procedure because it requires numerous calls to the blockchain RPC endpoint. For instance, if we need the average for a 30-minute period for a network with 15-second blocks, there should be 120 calls of ``getReserves``. 

To solve this problem, a single request is sent to query all ``Sync`` events that are emitted each time the reserves are updated due to minting, burning and swapping. 

.. code-block:: javascript

    getSyncEvents: async function (chainId, seedBlockNumber, pairAddress, blocksToSeed) {
	    const w3 = networksWeb3[chainId]
	    const pair = new w3.eth.Contract(UNISWAPV2_PAIR_ABI, pairAddress)
	    const options = {
	        fromBlock: seedBlockNumber + 1,
	        toBlock: seedBlockNumber + blocksToSeed
	    }
	    const syncEvents = await pair.getPastEvents("Sync", options)
      
When there is more than one change in a block’s reserves, only the final should be considered and the others are excluded.
 
.. code-block:: javascript
 
 	let syncEventsMap = {}
	    // {key: event.blockNumber => value: event}
	    syncEvents.forEach((event) => syncEventsMap[event.blockNumber] = event)
	    return syncEventsMap
	},
  
Listing the Price for Each Block
================================

Now there is a list of all the blocks in which reserves have changed and the values of the reserves in those blocks for the defined time period. To calculate the TWAP, a list of prices is needed that shows the final price in each block. To generate this list, we require the initial reserve state in the seed block in addition to the list of reserves values. 

.. code-block:: javascript

	getSeed: async function (chainId, pairAddress, blocksToSeed, toBlock) {
	    const w3 = networksWeb3[chainId]
	    const seedBlockNumber = toBlock - blocksToSeed

	    const pair = new w3.eth.Contract(UNISWAPV2_PAIR_ABI, pairAddress)
	    const { _reserve0, _reserve1 } = await pair.methods.getReserves().call(seedBlockNumber)
	    const price0 = this.calculateInstantPrice(_reserve0, _reserve1)
	    return { price0: price0, blockNumber: seedBlockNumber }
	},
  
