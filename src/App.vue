<template>
  <div>
    <h1>My NFT</h1>

    <div v-if="!!currentUser">
      <p>Account: {{ currentUser?.accountId }}</p>
    </div>

    <button v-if="!currentUser" v-on:click="nearLogin">
      Login
    </button>
    <button v-else v-on:click="nearLogout">
      Logout
    </button>

    <div v-if="!!currentUser" class="container">

      <table>
        <tr>
          <td>
            <h3>Mint</h3>
          </td>
          <td>
            <button v-on:click="mint">mint</button>
          </td>
        </tr>

        <tr>
          <td>
            <h3>Get Owner</h3>
          </td>
          <td>
            <input v-model="tokenId" placeholder="Token Id">
            <button v-on:click="getOwner">query</button>
          </td>
        </tr>

        <tr>
          <td>
            <h3>Transfer</h3>
          </td>
          <td>
            <input v-model="receiver" placeholder="Receiver">
            <input v-model="transferTokenId" placeholder="Token Id">
            <button v-on:click="transfer">Transfer</button>
          </td>
        </tr>

      </table>

    </div>

  </div>
</template>

<script>
import * as nearAPI from 'near-api-js'
import getConfig from './near-config'
import Big from 'big.js'

async function initNear () {
  const nearConfig = getConfig(process.env.NODE_ENV || 'testnet')

  // Initializing connection to the NEAR TestNet
  const near = await nearAPI.connect({
    deps: {
      keyStore: new nearAPI.keyStores.BrowserLocalStorageKeyStore()
    },
    ...nearConfig
  })

  // Needed to access wallet
  const walletConnection = new nearAPI.WalletConnection(near)

  // Load in account data
  let currentUser
  if (walletConnection.getAccountId()) {
    currentUser = {
      accountId: walletConnection.getAccountId(),
      balance: (await walletConnection.account().state()).amount
    }
  }

  // Initializing our contract APIs by contract name and configuration
  const contract = await new nearAPI.Contract(walletConnection.account(), nearConfig.contractName, {
    // View methods are read-only – they don't modify the state, but usually return some value
    viewMethods: ['get_token_owner', 'name'],
    // Change methods can modify the state, but you don't receive the returned value when called
    changeMethods: ['mint_to', 'transfer'],
    // Sender is the account ID to initialize transactions.
    // getAccountId() will return empty string if user is still unauthorized
    sender: walletConnection.getAccountId()
  })

  return {
    contract,
    nearConfig,
    currentUser,
    walletConnection
  }
}

const BOATLOAD_OF_GAS = Big(3).times(10 ** 13).toFixed()

export default {
  name: 'App',
  data: () => ({
    contract: null,
    nearConfig: null,
    currentUser: null,
    wallet: null,
    tokenId: null,
    receiver: null,
    transferTokenId: null
  }),
  async mounted () {
    const near = await initNear()
    this.contract = near.contract
    this.nearConfig = near.nearConfig
    this.currentUser = near.currentUser
    this.wallet = near.walletConnection
  },
  methods: {
    nearLogin () {
      this.wallet.requestSignIn(
        this.nearConfig.contractName,
        'My NFT'
      )
    },
    nearLogout () {
      this.wallet.signOut()
      window.location.replace(window.location.origin + window.location.pathname)
    },
    async mint () {
      const tokenId = await this.contract.mint_to(
        { owner_id: this.currentUser.accountId },
        BOATLOAD_OF_GAS
      )
      console.log('minted', tokenId)
      alert(`Minted token ${tokenId}`)
    },
    async getOwner () {
      try {
        const owner = await this.contract.get_token_owner({
          token_id: this.tokenId
        })

        alert(`Owner is ${owner}`)
      } catch (err) {
        console.log(err)
      }
    },
    async transfer () {
      try {
        await this.contract.transfer({
          new_owner_id: this.receiver,
          token_id: this.transferTokenId
        })
        alert(`Token ${this.transferTokenId} transferred to ${this.receiver}`)
      } catch (err) {
        console.log(err)
      }
    }
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
.container {
  margin-top: 40px;
}
table {
  margin: auto;
}
table, td {
  border: 1px solid black;
    border-collapse: collapse;
}
</style>