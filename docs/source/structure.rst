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

