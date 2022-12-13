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

Detecting Outliers
==================

Before calculating the average, prices that are potentially the result of manipulation should be detected and removed from the list. This is technically called *outlier* detection. At present, a simple algorithm called *Z-score* is used for outlier detection. 

The Z-score measures how far a data point is away from the mean as a multiple of the standard deviation (std). In simple words, it indicates how many standard deviations an element is from the mean, so 
 
.. code-block:: javascript

    z_score = abs(x - mean) / std

This means any price with a Z-score higher than the threshold will be considered an outlier and excluded from the final average. 

.. code-block:: javascript

	std: function (arr) {
	    let mean = arr.reduce((result, el) => result + el, 0) / arr.length
	    arr = arr.map((k) => (k - mean) ** 2)
	    let sum = arr.reduce((result, el) => result + el, 0)
	    let variance = sum / arr.length
	    return Math.sqrt(variance)
	},

	removeOutlierZScore: function (prices) {
	    const mean = this.calculateAveragePrice(prices)
	    // calculate std(standard deviation)
	    const std = this.std(prices)
	    if (std == 0) return prices

	    // Z score = abs(price - mean) / std
	    // price is not reliable if Z score > threshold
	    return prices.filter((price) => Math.abs(price - mean) / std < THRESHOLD)
	},

For outlier detection based on Z-score, the price logarithm is used because price is  logarithmic in nature. Essentially, using the log of prices can better show the viewer the rate of change over time. If prices are considered linearly, price change from 1 to 2 equals price change from 1001 to 1002. In logarithmic viewpoint, however, these two changes are clearly different. 

The process of removing outliers is done twice. Calculating the average including outliers makes the average and the resulting standard deviation biased. Repeating the outlier detection process after cleaning the data set by removing any obviously outlying prices in the first run assures us that more subtle outliers can be detected as well. Although this approach may cause the removal of prices that are not the result of price manipulation, it drastically reduces the chances of not detecting a manipulated price.   

.. code-block:: javascript

	removeOutlier: function (prices) {
	    const logPrices = []
	    prices.forEach((price) => {
	        logPrices.push(Math.log(price));
	    })
	    let logOutlierRemoved = this.removeOutlierZScore(logPrices)

	    logOutlierRemoved = this.removeOutlierZScore(logOutlierRemoved)

	    const outlierRemoved = []
	    const removed = []
	    prices.forEach((price, index) => logOutlierRemoved.includes(logPrices[index]) ? outlierRemoved.push(price) : removed.push(price.toString()))

	    return { outlierRemoved, removed }
	},

Now we have all the necessary data to calculate the average. To make the process simpler, only the price of ``token0`` in terms of ``token1`` has been calculated so far. However, each pair is made of two tokens, each of which has a price in terms of the other and is the other’s reverse. Mathematically, the average of the reverses of multiple numbers does not equal the reverse of their average. That is why we need to calculate all the reverses and then their average to obtain the time weighted average price of ``token1`` in terms of ``token0``.

.. code-block:: javascript

	calculateAveragePrice: function (prices, returnReverse) {
	    let fn = function (result, el) {
	        return returnReverse ? { price0: result.price0.add(el), price1: result.price1.add(Q112.mul(Q112).div(el)) } : result + el
	    }
	    const sumPrice = prices.reduce(fn, returnReverse ? { price0: new BN(0), price1: new BN(0) } : 0)
	    const averagePrice = returnReverse ? { price0: sumPrice.price0.div(new BN(prices.length)), price1: sumPrice.price1.div(new BN(prices.length)) } : sumPrice / prices.length
	    return averagePrice
	},



