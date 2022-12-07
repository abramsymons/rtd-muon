Introduction
============

What is a Muon App?
------------------

Apps in the decentralized environment, called dApps, are made up of a smart contract and a client, that is, a frontend user interface. There are, however, dapps that need an extra component to provide data from off-chain sources, i.e. an oracle. DApps for different purposes have different oracles that read and process data from sources and return signed responses. 

At a first glance, one might think that developing an oracle is the simplest stage in developing a dApp; all there is to do is implementing a function that fetches and processes data to generate a signed output suitable to be fed to its smart contract. 

However, it is not that simple. Data feeds provided by oracles are vulnerable to attacks and manipulation. The server on which an oracle component is run may be hacked and the private key used to sign data feeds can be abused to feed manipulated data to the smart contract. This is why, in addition to developing a simple oracle function, dApps need to deal with implementing solutions for complicated security issues. 

As a viable alternative, dApps can focus on the development of the oracle function as a  Muon app and depend on the multi-layer security scheme that Muon has devised. In simple words, a Muon app refers to an oracle app that is deployed and runs on the Muon network to fetch and process data and generate an output that can be fed to a smart contract reliably. 

A Simple Oracle App
-------------------

Muon apps are currently developed in JS; other programming languages will gradually be added. Here is a sample oracle app whose job is to fetch the price of ETH from Coinbase API.

.. code-block:: javascript

    const { axios } = MuonAppUtils

    module.exports = {
      APP_NAME: 'simple_oracle',

      onRequest: async function(request){
        let { method } = request
        switch (method) {
          case 'eth-price':
            const response = await axios.get('https://api.coinbase.com/v2/exchange-rates?currency=ETH')
            return {
              price: parseInt(response.data['rates']['USD']),
            }

          default:
            throw `Unknown method ${method}`
        }
      },

      signParams: function(request, result){
        let { method } = request;
        let { price } = result;
        switch (method) {
          case 'eth-price':
            return [
              { type: 'uint256', value: price }
            ]
          default:
            throw `Unknown method ${method}`
        }
      }
    }

A Muon app is a module that exports two functions: ``onRequest`` and ``signParams``. The first fetches data, does any necessary processing and returns any data needed to be fed to the smart contract. The second function lists all the parameters that are to be included in the signed message and their types.

Deploying the App
-----------------

Having prepared the simple oracle app, the developer needs to run a local network to test it. There are two prerequisites that should be installed: `Mongo <https://www.mongodb.com/docs/manual/installation/>`_ and `Redis <https://redis.io/docs/getting-started/installation/>`_. After these two have been installed, the following steps should be followed to run the network.

The first step is to clone Muon node’s repository and checkout the testnet branch through:  

.. code-block:: javascript

    $ git clone git@github.com:muon-protocol/muon-node-js.git --recurse-submodules
    $ cd muon-node-js
    $ git checkout testnet

The next step is to install required node modules as follows: 

.. code-block:: javascript

    $ npm install
