<template>
  <v-layout column justify-center align-center>
    <v-flex xs12 sm8 md6>
      <div class="text-align-center mb-5">
        <v-progress-circular
          v-if="pending"
          :size="70"
          :width="7"
          color="purple"
          indeterminate
        />
        <span v-if="!pending" class="display-4">
          {{ counter }}
        </span>
      </div>
    </v-flex>
    <v-flex xs12 sm8 md6>
      <div class="text-align-center">
        <v-btn
          fab
          dark
          color="primary"
          :disabled="pending || counter < 1"
          @click="sub"
        >
          <v-icon dark>remove</v-icon>
        </v-btn>
        <v-btn fab dark color="indigo" @click="add">
          <v-icon dark>add</v-icon>
        </v-btn>
      </div>
    </v-flex>
  </v-layout>
</template>

<script>
import { ethers } from 'ethers'
import {
  NonceTxMiddleware,
  SignedEthTxMiddleware,
  CryptoUtils,
  Client,
  LoomProvider,
  Address,
  LocalAddress,
  Contracts,
  EthersSigner,
  createDefaultTxMiddleware
} from 'loom-js'
import SimpleStoreJSON from '../truffle/build/contracts/SimpleStore.json'
const Web3 = require('web3')

export default {
  data: function() {
    return {
      counter: 0,
      pending: true,
      web3js: null,
      chainId: 'extdev-plasma-us1',
      writeUrl: 'wss://extdev-plasma-us1.dappchains.com/websocket',
      readUrl: 'wss://extdev-plasma-us1.dappchains.com/queryws',
      networkId: 9545242630824,
      callerChainId: 'eth',
      ethAddress: null,
      client: null,
      loomProvider: null,
      contract: null,
      publicKey: null
    }
  },

  async mounted() {
    await this.initWeb3()
    await this.init()
    await this.getContract()
  },

  methods: {
    async initWeb3() {
      let web3js
      if (window.ethereum) {
        window.web3 = new Web3(window.ethereum)
        web3js = new Web3(window.ethereum)
        await window.ethereum.enable()
      } else if (window.web3) {
        window.web3 = new Web3(window.web3.currentProvider)
        web3js = new Web3(window.web3.currentProvider)
      } else {
        alert('Metamask is not Enabled')
      }
      if (web3js) {
        this.web3js = web3js
      }
    },

    async init() {
      const privateKey = CryptoUtils.generatePrivateKey()
      this.publicKey = CryptoUtils.publicKeyFromPrivateKey(privateKey)

      this.client = new Client(this.chainId, this.writeUrl, this.readUrl)

      const ethersProvider = new ethers.providers.Web3Provider(
        this.web3js.currentProvider
      )
      const signer = ethersProvider.getSigner()
      this.ethAddress = await signer.getAddress()

      const to = new Address(
        this.callerChainId,
        LocalAddress.fromHexString(this.ethAddress)
      )
      const from = new Address(
        this.client.chainId,
        LocalAddress.fromPublicKey(this.publicKey)
      )

      this.client.txMiddleware = createDefaultTxMiddleware(
        this.client,
        privateKey
      )

      const addressMapper = await Contracts.AddressMapper.createAsync(
        this.client,
        from
      )

      if (await addressMapper.hasMappingAsync(to)) {
        console.log('Mapping already exists.')
      } else {
        console.log('Adding a new mapping.')
        const ethersSigner = new EthersSigner(signer)
        await addressMapper.addIdentityMappingAsync(from, to, ethersSigner)
      }

      this.loomProvider = new LoomProvider(this.client, privateKey)
      this.loomProvider.callerChainId = this.callerChainId
      this.loomProvider.setMiddlewaresForAddress(to.local.toString(), [
        new NonceTxMiddleware(to, this.client),
        new SignedEthTxMiddleware(signer)
      ])
      return true
    },

    async getContract() {
      const web3 = new Web3(this.loomProvider)
      this.contract = new web3.eth.Contract(
        SimpleStoreJSON.abi,
        SimpleStoreJSON.networks[this.networkId].address
      )
      const initialValue = await this.contract.methods.get().call({
        from: this.ethAddress
      })
      this.counter = parseInt(initialValue, 10)
      this.pending = false
    },

    async setValue() {
      await this.contract.methods.set(parseInt(this.counter, 10)).send({
        from: this.ethAddress
      })
    },

    async add() {
      this.pending = true
      this.counter += 1
      await this.setValue()
      this.pending = false
    },

    async sub() {
      if (this.counter < 1) return
      this.pending = true
      this.counter -= 1
      await this.setValue()
      this.pending = false
    }
  }
}
</script>
