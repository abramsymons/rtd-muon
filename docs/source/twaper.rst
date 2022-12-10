***********************
TWAPER (Price Feed App)
***********************

This section describes how a price feed app is developed using Muon. It is made up of two sections: 

  1- In the first section, we describe how an app is developed to calculate Time Weighted Average Price (TWAP) of a pair of tokens based on the information it gets from a Uniswap pool or one of its forks. 

  2- we explain how to use the app from section one to obtain a token’s price from a route made of pairs that ends with a stablecoin as well as calculating a Volume Weighted Average Price (VWAP) of routes in different exchanges on different chains 


Calculating TWAP of a Pair
==========================

Our off-chain implementation of TWAP has the following benefits over the original on-chain TWAP of Uniswap.
  - It detects and removes outlier prices before calculating averages to prevent price manipulations through applying a sharp rise/fall in the price for a short duration.
  - In order to reject unexpected price changes, it applies a fuse mechanism that stops the system when a short duration average shows a large price volatility compared to a longer one.
  - It does not require periodic transactions to register checkpoints on-chain which are costly and hard to maintain.

Here is the technical details of how the app is implemented:

Obtaining Price Changes
-----------------------

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
--------------------------------

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
------------------

Before calculating the average, prices that are potentially the result of manipulation should be detected and removed from the list. This is technically called outlier detection. At present, a simple algorithm called Z-score is used for outlier detection. 

The Z-score measures how far a data point is away from the mean as a multiple of the standard deviation (std). In simple words, it indicates how many standard deviations an element is from the mean, so 

.. code-block:: javascript

    z_score = abs(x - mean) / std

This means any prices that are higher than the threshold Z score will be considered as an outlier and excluded from the final average. 

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

        // Z score = (price - mean) / std
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

Now we have all the necessary data to calculate the average. To make the process simpler, only the price of ``token0`` in terms of ``token1`` has been calculated so far. However, each pair is made of two tokens, each of which has a price in terms of the other and is the other’s reverse. Mathematically, the average of the reverse of a list of numbers does not equal the reverse of their average. That is why we need to calculate all the reverses and calculate their average to obtain the price average of ``token1`` in terms of ``token0``.

.. code-block:: javascript

    calculateAveragePrice: function (prices, returnReverse) {
        let fn = function (result, el) {
            return returnReverse ? { price0: result.price0.add(el), price1: result.price1.add(Q112.mul(Q112).div(el)) } : result + el
        }
        const sumPrice = prices.reduce(fn, returnReverse ? { price0: new BN(0), price1: new BN(0) } : 0)
        const averagePrice = returnReverse ? { price0: sumPrice.price0.div(new BN(prices.length)), price1: sumPrice.price1.div(new BN(prices.length)) } : sumPrice / prices.length
        return averagePrice
    },

        logOutlierRemoved = this.removeOutlierZScore(logOutlierRemoved)

        const outlierRemoved = []
        const removed = []
        prices.forEach((price, index) => logOutlierRemoved.includes(logPrices[index]) ? outlierRemoved.push(price) : removed.push(price.toString()))

        return { outlierRemoved, removed }
    },

Implementing Fuse Mechanism
---------------------------

Having removed the outliers, the short-term average is generated. At this stage, a fuse mechanism is implemented, through which the short-term average is compared with a longer-term average, for instance an 24-hour average, and stops the system if the result of the comparison shows a large difference.

.. code-block:: javascript

    checkFusePrice: async function (chainId, pairAddress, price, fusePriceTolerance, blocksToFuse, toBlock, abiStyle) {
        const w3 = networksWeb3[chainId]
        const seedBlock = toBlock - blocksToFuse

        const fusePrice = await this.getFusePrice(w3, pairAddress, toBlock, seedBlock, abiStyle)
        if (fusePrice.price0.eq(new BN(0)))
            return {
                isOk0: true,
                isOk1: true,
                priceDiffPercentage0: new BN(0),
                priceDiffPercentage1: new BN(0),
                block: fusePrice.blockNumber
            }
        const checkResult0 = this.isPriceToleranceOk(price.price0, fusePrice.price0, fusePriceTolerance)
        const checkResult1 = this.isPriceToleranceOk(price.price1, Q112.mul(Q112).div(fusePrice.price0), fusePriceTolerance)

        return {
            isOk0: checkResult0.isOk,
            isOk1: checkResult1.isOk,
            priceDiffPercentage0: checkResult0.priceDiffPercentage,
            priceDiffPercentage1: checkResult1.priceDiffPercentage,
            block: fusePrice.blockNumber
        }
    },
    ...
    calculatePairPrice: async function (chainId, abiStyle, pair, toBlock) {
        ...
        const fuse = await this.checkFusePrice(chainId, pair.address, price, pair.fusePriceTolerance, blocksToFuse, toBlock, abiStyle)
        if (!(fuse.isOk0 && fuse.isOk1)) throw { message: `High price gap 0(${fuse.priceDiffPercentage0}%) 1(${fuse.priceDiffPercentage1}%) between fuse and twap price for ${pair.address} in block range ${fuse.block} - ${toBlock}` }
        ...
    },

To calculate the long-term average needed for the fuse mechanism, we use the off-chain implementation of the exact method that DEXes use to calculate on-chain TWAP.

The fact that we make use of different methods for the calculation of short and long-term averages heightens the app’s reliability; if there is a bug in one of the methods or an attack that influences one of them, the other can cover it.  

Some Uniswap forks have made modifications to the on-chain TWAP calculation method originally made by Uniswap. In this app, the original Uniswap version and a well-known fork, Solidly, are implemented. 

.. code-block:: javascript

    getFusePrice: async function (w3, pairAddress, toBlock, seedBlock, abiStyle) {
        const getFusePriceUniV2 = async (w3, pairAddress, toBlock, seedBlock) => {
            ...
        }
        const getFusePriceSolidly = async (w3, pairAddress, toBlock, seedBlock) => {
            ...
        }
        const GET_FUSE_PRICE_FUNCTIONS = {
            UniV2: getFusePriceUniV2,
            Solidly: getFusePriceSolidly,
        }

        return GET_FUSE_PRICE_FUNCTIONS[abiStyle](w3, pairAddress, toBlock, seedBlock)
    },

In this doc, only the original Uniswap implementation is explained. To calculate the long-term average, we make use of the two variables ``price0CumulativeLast`` & ``price1CumulativeLast`` that are available on the pair contract for on-chain TWAP calculations.

Here is the method by which Uniswap calculates time-weighted average: Each time the price changes, it multiplies the previous price by the time period during which that price is valid as the weight of the price. The summation of the results are accumulated in the ``priceCumulativeLast``  which is divided by the total time period resulting in the time-weighted average. The following diagram illustrates how this process works. To get more information, see `here <https://docs.uniswap.org/protocol/V2/concepts/core-concepts/oracles>`_.
