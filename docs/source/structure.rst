Structure of a Muon App
=======================

Functions
---------

`onRequest`

It is the most important function that every app should implement. This function receives the request as an argument, fetches required data from external sources, does any necessary processing and returns any data needed to be fed to the smart contract. `request` has different attributes. 

.. code-block:: javascript

    onRequest: async function (request) {
        let {
          method,
          data: { params }
        } = request
        ...
    }

The first one is the `method` that the user has called. Muon apps can have different methods, each of which should be separately handled in the `onRequest` function. The second one is `data` that, in turn, has an attribute called `params` which includes the parameters that are to be passed to the called `method`.

The simple oracle app mentioned in section 1b returns the price of ETH in terms of USD. If we are to expand the usage of the app to be able to return the price of any given token in terms any unit supported by Coinbase API, `token` and `unit` should be passed to the app the following way:  

.. code-block:: javascript

    /?app=simple_oracle&method=price&params[token]=ETH&params[unit]=USD 
    
In `onRequest`, parameters can be received and used in the following way:

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

`signParams`

This is another function that all Muon apps should implement. This method returns a list of all the parameters that are to be included in the signed message and their types. The type of each element defines how each should be encoded and included in the signed message. The available types are `int256`, `uint256`, `bytes256`, `address`, and `string`. The first three types support size variations 8, 16, 32, 64, 128 as well. Muon core packs `appId`, `requestId` and the current list, and uses its hash as the message that should be signed.

.. warning::

To ensure that the signed and verified response has accurately covered the requested data, the parameters passed to the app should also be included in the returned value of signParams in addition to the result. Otherwise, the signature queried from the app with certain parameters might be abused and fed to the dApp contract with different ones. If the app has different methods, the method name should be included as well.

