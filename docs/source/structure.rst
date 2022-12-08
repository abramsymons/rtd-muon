Structure of a Muon App
***********************

Functions
=========

onRequest
-------------

It is the most important function that every app should implement. This function receives the request as an argument, fetches required data from external sources, does any necessary processing and returns any data needed to be fed to the smart contract. `request` has different attributes. 

.. code-block:: javascript

    onRequest: async function (request) {
        let {
          method,
          data: { params }
        } = request
        ...
    }

The first one is the ``method`` that the user has called. Muon apps can have different methods, each of which should be separately handled in the ``onRequest`` function. The second one is ``data`` that, in turn, has an attribute called ``params`` which includes the parameters that are to be passed to the called `method`.

The simple oracle app mentioned in section 1b returns the price of ETH in terms of USD. If we are to expand the usage of the app to be able to return the price of any given token in terms any unit supported by Coinbase API, `token` and `unit` should be passed to the app the following way:  

.. code-block:: javascript

    /?app=simple_oracle&method=price&params[token]=ETH&params[unit]=USD 
    
In ``onRequest``, parameters can be received and used in the following way:

.. code-block:: javascript
    
    onRequest: async function(request){
        let {
          method,
          data: { params }
        } = request
        let { token, unit } = params
        switch (method) {
          case 'price':
            const response = await axios.get(`https://api.coinbase.com/v2/exchange-rates?currency=${token}`)
            return {
              price: parseInt(response.data['rates'][unit]),
            }

          default:
            throw { message: `Unknown method ${method}` }
        }
    }

signParams
--------------

This is another function that all Muon apps should implement. This method returns a list of all the parameters that are to be included in the signed message and their types. The type of each element defines how each should be encoded and included in the signed message. The available types are ``int256``, ``uint256``, ``bytes256``, ``address``, and ``string``. The first three types support size variations 8, 16, 32, 64, 128 as well. Muon core packs ``appId``, ``requestId`` and the current list, and uses its hash as the message that should be signed.

.. warning::

To ensure that the signed and verified response has accurately covered the requested data, the parameters passed to the app should also be included in the returned value of signParams in addition to the result. Otherwise, the signature queried from the app with certain parameters might be abused and fed to the dApp contract with different ones. If the app has different methods, the method name should be included as well.

If the simple oracle app is to be expanded to contain the token and unit parameters, the signParams should be updated as follows: 

.. code-block:: javascript

    signParams: function(request, result){
      let {
        method,
        data: { params }
      } = request
      let { token, unit } = params
      let { price } = result;
      switch (method) {
        case 'price':
          return [
            { type: 'uint32', value: price },
            { type: 'string', value: token },
            { type: 'string', value: unit },
          ]
        default:
          throw `Unknown method ${method}`
      }
    }

### How to Use Gateway Data ###

For certain use-cases such as getting token prices, the requested data from the TSS network fluctuates momentarily. Obtaining the token price from Coinbase API in the simple oracle app is one such case. The price may fluctuate numerous times in one or two seconds, so the obtained data from different nodes in the TSS network may differ slightly. However, to generate the threshold signature, all nodes should sign exactly the same data.  

To address this problem, Muon’s TSS network makes use of the following data-obtaining procedure. The node that receives the data request from the client, the gateway node, obtains required data, and then shares it with others in the TSS group. The other nodes obtain the required data and compare it with the data from the gateway node. If their obtained data is within a predefined range of the gateway data, they sign the data from the gateway node, not their own data. Finally, the gateway node aggregates the signatures and generates the threshold signature. This way, the threshold signature is on one set of data that was initially obtained by the gateway node.

For such applications, signParams should include the data provided by the gateway node instead of its own price if its own data is marginally different from that of the gateway. Otherwise, it rejects the request. So ``signParams`` should be updated as following: 

.. code-block:: javascript

    const gatwayPrice = request.data?.result?.price || price;
    if (100 * Math.abs(price - gatewayPrice) / price > 0.5) {
      throw 'invalid price'
    }
    return [
      { type: 'uint32', value: gatewayPrice },
      { type: 'string', value: token },
      { type: 'string', value: unit },
    ]

The ``request.data?.result?.price`` is ``undefined`` when it is evaluated on the gateway node; if not, its value is that of the gateway node’s. The price from the gateway node is verified only if the margin is lower than 0.5%.

Another essential piece of data that should be added to the returned list of ``signParams`` in some applications is the request’s timestamp. If the timestamp is not included for a token price, for instance, an old price signed a long time ago may be fed into the dApp. The points explained above are also true about timestamps; that is, the times when different nodes receive requests may differ slightly. So all nodes need to sign the gateway node’s time. Gateway time can be accessed via ``request.data.timestamp``.

Timestamp deviation does not need to be manually verified in the code the way that is done for price. When a node receives a request from the gateway node, it checks ``request.data.timestamp`` whether the time gap is not more than 30 seconds. Otherwise, it rejects the request. So it is sufficient to include ``request.data.timestamp`` in the returned list of ``signParams`` the following way.  

.. code-block:: javascript

    return [
      { type: 'uint32', value: gatewayPrice },
      { type: 'string', value: token },
      { type: 'string', value: unit },
      { type: 'uint32', value: request.data.timestamp },
    ]

Memory
======

Although the Muon oracle network is stateless, there are applications that need TTL-based caching. Suppose the simple oracle app is to limit the number of requests to Coinbase API and cache the response for a short period, for example 5 seconds. The ``readLocalMem`` and ``writeLocalMem`` functions as follows:

.. code-block:: javascript

    let data = await this.readLocalMem(`price-${token}`)
    if (!data) {
      const response = await axios.get(`https://api.coinbase.com/v2/exchange-rates?currency=${token}`)  
      data = JSON.stringify(response.data)
      await this.writeLocalMem(`price-${token}`, [{type: "string", value: data}], 5)
    }
    data = JSON.parse(data)
    return {
      price: parseInt(data['rates'][unit]),
    }

One of the use-cases of these functions is the implementation of a locking system. To do so, reading and writing cannot be run separately because if there are two concurrent requests, they may both acquire the lock simultaneously. To solve this problem, ``{ getset: true }`` can be passed to ``writeLocalMem``. Doing so, the ``writeLocalMem`` first reads the value prior to its writing and returns it. This assures that reading and writing occur in an atomic way.

.. code-block:: javascript

    const alreadyLocked = await this.writeLocalMem(
    `lock-${user}`,
      [{ type: "bool", value: true }],
      5,
      { getset: true }
    );
    if (alreadyLocked) throw user locked;
    // the code block requires acquiring the lock

Utilities
=========

Developers can use ``MuonAppUtils`` to access available utilities for developing Muon apps.

.. code-block:: javascript

    const { axios } = MuonAppUtils

Here is the list of available utilities: 

.. code-block:: javascript

    ‍const axios = require('axios')
    const Web3 = require('web3')
    const tron = require('../utils/tron')
    const { flatten, groupBy } = require('lodash')
    const { BigNumber } = require('bignumber.js')

    const { toBaseUnit } = require('../utils/crypto')
    const { timeout, floatToBN } = require('../utils/helpers')
    const util = require('ethereumjs-util')
    const ws = require('ws')
    const ethSigUtil = require('eth-sig-util')
    const {
      getBlock: ethGetBlock,
      getBlockNumber: ethGetBlockNumber,
      getPastEvents: ethGetPastEvents,
      read: ethRead,
      call: ethCall,
      getTokenInfo: ethGetTokenInfo,
      getNftInfo: ethGetNftInfo,
      hashCallOutput: ethHashCallOutput
    } = require('../utils/eth')

    const soliditySha3 = require('../utils/soliditySha3');

    const { multiCall } = require('../utils/multicall')
    const { BNSqrt } = require('../utils/bn-sqrt')

    global.MuonAppUtils = {
      axios,
      Web3,
      flatten,
      groupBy,
      tron,
      ws,
      timeout,
      BN: Web3.utils.BN,
      BigNumber,
      toBN: Web3.utils.toBN,
      floatToBN,
      multiCall,
      ethGetBlock,
      ethGetBlockNumber,
      ethGetPastEvents,
      ethRead,
      ethCall,
      ethGetTokenInfo,
      ethGetNftInfo,
      ethHashCallOutput,
      toBaseUnit,
      soliditySha3,
      ecRecover: util.ecrecover,
      recoverTypedSignature: ethSigUtil.recoverTypedSignature,
      recoverTypedMessage: ethSigUtil.recoverTypedMessage,
      BNSqrt: BNSqrt
    }


