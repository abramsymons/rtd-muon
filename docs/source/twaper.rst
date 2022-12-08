#######################
TWAPER (Price Feed App)
#######################

This section describes how a price feed app is developed using Muon. It is made up of two sections: 

  1- In the first section, we describe how an app is developed to calculate Time Weighted Average Price (TWAP) of a pair of tokens based on the information it gets from a Uniswap pool or one of its forks. 

  2- we explain how to use the app from section one to obtain a token’s price from a route made of pairs that ends with a stablecoin as well as calculating a Volume Weighted Average Price (VWAP) of routes in different exchanges on different chains 


**************************
Calculating TWAP of a Pair
**************************

Our off-chain implementation of TWAP has the following benefits over the original on-chain TWAP of Uniswap.
  - It detects and removes outlier prices before calculating averages to prevent price manipulations through applying a sharp rise/fall in the price for a short duration.
  - In order to reject unexpected price changes, it applies a fuse mechanism that stops the system when a short duration average shows a large price volatility compared to a longer one.
  - It does not require periodic transactions to register checkpoints on-chain which are costly and hard to maintain.

Here is the technical details of how the app is implemented:

Obtaining Price Changes
=======================

To calculate TWAP, a time period is defined with a source and a destination time. Also, the token prices for the defined period should be obtained from each block. It seems that the app needs to call ``getReserves`` for each block, calculate the price for the block by dividing ``_reserve1`` to ``_reserve0``, and calculate the average of all the prices. 

Following such an approach is a time-consuming and costly procedure because it requires numerous calls to the blockchain RPC endpoint. For instance, if we need the average for a 30-minute period for a network with 15-second blocks, there should be 120 calls of  ``getReserves``. 

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

At this stage, the initial state and list of reserves values are available, so the price list can be generated. 

.. code-block:: javascript

    createPrices: function (seed, syncEventsMap, blocksToSeed) {
        let prices = [seed.price0]
        let price = seed.price0
        // fill prices and consider a price for each block between seed and current block
        for (let blockNumber = seed.blockNumber + 1; blockNumber <= seed.blockNumber + blocksToSeed; blockNumber++) {
            // use block event price if there is an event for the block
            // otherwise use last event price
            if (syncEventsMap[blockNumber]) {
                const { reserve0, reserve1 } = syncEventsMap[blockNumber].returnValues
                price = this.calculateInstantPrice(reserve0, reserve1)
            }
            prices.push(price)
        }
        return prices
    },

Each pair is made up of two tokens. To calculate the price of ``token0`` in terms of ``token1`` from the reserves, ``reserve1`` should be divided by ``reserve0``. As there are no floating point numbers in Solidity, and price may be a floating point number, a quotient named ``Q112`` is used to retain the precision of the price by multiplying it by ``2^112``. 

.. code-block:: javascript

    calculateInstantPrice: function (reserve0, reserve1) {
        // multiply reserveA into Q112 for precision in division 
        // reserveA * (2 ** 112) / reserverB
        const price0 = new BN(reserve1).mul(Q112).div(new BN(reserve0))
        return price0
    },

