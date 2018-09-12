<template>
  <b-container>
    <h5 class="mb-3" v-if="isWeb3">
      <small>YOUR WALLET:</small>&nbsp;
      <b-badge pill variant="success" v-if="signerState">{{ signer.address }}</b-badge>
      <b-badge pill variant="warning" v-else>Are you logged in?!</b-badge>
    </h5>
    <b-alert variant="danger" show fade v-else>MetaMask (or Web3 provider) not loaded?!</b-alert>

    <b-alert variant="info" show fade v-if="isLoading"><strong>Please wait, reading blockchain data...</strong></b-alert>

    <b-alert v-for="m in messages" :key="m.id" :variant="m.variant" @dismissed="delMessage(m.id)" dismissible show fade><strong>{{m.text}}</strong></b-alert>
    <b-alert variant="warning" show fade v-if="isWarnings || isErrors">Something went wrong, may be incorrect network selected?</b-alert>
    <b-alert variant="primary" show fade v-if="ethWithdrawing">Withdrawal in progress. Please confirm transaction in Metamask and wait for transaction
      completed.
    </b-alert>

    <b-card-group deck>
      <b-card border-variant="success" text-variant="white" sub-title="TOKENS" class="mb-2">
        <h6 v-if="contractState"><b-badge>{{tkn_contract.address}}</b-badge></h6>
        <!--<p class="card-text">You owned <strong>{{getBalance}}</strong> of <strong>{{getTotalSupply}}</strong> dividend tokens</p>-->
        <h5>You owned <strong>{{getBalance}}</strong> of <strong>{{getTotalSupply}}</strong> UnicornGO Dividend Tokens (UDT)</h5>
      </b-card>
      <b-card border-variant="primary" text-variant="white" sub-title="WITHDRAWAL" class="mb-2">
        <h6 v-if="contractState"><b-badge>{{mgr_contract.address}}</b-badge></h6>
        <!--<p class="card-text">You can withdraw <strong>{{getPendingWithdrawal}}&Xi;</strong></p>-->
        <h5>You can withdraw <strong>{{getPendingWithdrawal}}&Xi;</strong></h5>
        <b-button variant="outline-primary" :disabled="withdrawState" @click="withdraw">Withdraw</b-button>
      </b-card>
    </b-card-group>
    <b-card-group deck>
      <b-card border-variant="warning" sub-title="YOUR RECENT WITHDRAWALS" class="mb-2">
        <b-table striped small fixed :items="listWithdrawals" v-if="listWithdrawals.length"></b-table>
        <p class="card-text" v-else>list is empty</p>
        <EllipsisSpinner v-if="loadingWithdrawals"/>
      </b-card>
      <b-card border-variant="danger" sub-title="YOUR RECENT EARNINGS" class="mb-2">
        <b-table striped small fixed :items="listEarnings" v-if="listEarnings.length"></b-table>
        <p class="card-text" v-else>list is empty</p>
        <EllipsisSpinner v-if="loadingEarnings"/>
      </b-card>
      <b-card border-variant="info" sub-title="TOTAL RECENT DIVIDENDS INCOME" class="mb-2">
        <b-table striped small fixed :items="listDividends" v-if="listDividends.length"></b-table>
        <p class="card-text" v-else>list is empty</p>
        <EllipsisSpinner v-if="loadingDividends"/>
      </b-card>
    </b-card-group>

  </b-container>
</template>

<script>
  /* eslint-disable no-unused-vars,no-console */

  import ethers from 'ethers'
  import EllipsisSpinner from "./EllipsisSpinner";
  export default {
    name: 'UnicornDividends',
    components: {EllipsisSpinner},
    data() {
      return {
        tkn_contract: null,
        mgr_contract: null,
        provider: null,
        signer: null,
        balance: ethers.utils.bigNumberify(0),
        pendingWithdrawal: ethers.utils.bigNumberify(0),
        totalSupply: ethers.utils.bigNumberify(0),
        ethWithdrawing: false,
        listDividends: [],
        listEarnings: [],
        listWithdrawals: [],
        loadingDividends: false,
        loadingEarnings: false,
        loadingWithdrawals: false,
        messages: []
      }
    },
    computed: {
      isErrors() {
        return this.messages.filter(i => i.variant === 'danger').length > 0
      },
      isWarnings() {
        return this.messages.filter(i => i.variant === 'warning').length > 0
      },
      isLoading() {
        return this.loadingWithdrawals || this.loadingDividends || this.loadingEarnings
      },

      getBalance() {
        return ethers.utils.formatUnits(this.balance, 3)
      },
      getPendingWithdrawal() {
        return ethers.utils.formatEther(this.pendingWithdrawal)
      },
      getTotalSupply() {
        return ethers.utils.formatUnits(this.totalSupply, 3)
      },
      withdrawState() {
        return this.pendingWithdrawal.isZero() || this.ethWithdrawing
      },
      isWeb3() {
        /* eslint-disable no-undef */
        return typeof web3 !== 'undefined'
        /* eslint-enable no-undef */
      },
      getCurrentProvider() {
        /* eslint-disable no-undef */
        return typeof web3 !== 'undefined' ? web3.currentProvider : null
        /* eslint-enable no-undef */
      },
      signerState() {
        return this.signer !== null
      },
      providerState() {
        return this.provider !== null
      },
      contractState() {
        return this.tkn_contract !== null && this.mgr_contract !== null
      },


      getNetwork() {
        if (typeof web3 !== 'undefined') {
          /* eslint-disable no-undef */
          switch (web3.currentProvider.publicConfigStore.getState().networkVersion) {
            /* eslint-enable no-undef */
            case '2':
              return 'morden'
            case '3':
              return 'ropsten'
            case '4':
              return 'rinkeby'
            case '42':
              return 'kovan'
            // case '1':
            //   return 'mainnet'
            default:
              return 'homestead'
          }
        }
        return 'homestead'
      },
      // web3state() {
      //   /* eslint-disable no-undef */
      //   return typeof web3 !== 'undefined' ? web3.currentProvider.publicConfigStore.getState() : {}
      //   /* eslint-enable no-undef */
      // }
    },
    watch: {
      'signer': function (signer) {
        this.initItems()
      },
      // 'web3state': function (state) {
      //   console.log(state)
      // }
    },
    methods: {
      addMessage(variant, text) {
        this.messages.push({
          id: this.messages.length + 1,
          variant,
          text
        })
      },
      delMessage(id) {
        let idx = this.messages.findIndex(i => i.id === id)
        this.messages.splice(idx, 1)
      },
      reloadEarnings() {
        this.loadingEarnings = true
        let topicAddress = '0x000000000000000000000000' + this.signer.address.substring(2);
        this.provider.getLogs({
          fromBlock: 0,
          toBlock: 'latest',
          address: process.env.VUE_APP_D_MGR_ADDRESS,
          topics: [
            this.mgr_contract.interface.events.WithdrawalAvailable.topics[0],
            topicAddress
          ]
        }).then(logs => {
          this.listEarnings = logs.filter(e => !e.removed)
            .sort((a, b) => {
              //desc sorting
              let deltaBlock = b.blockNumber - a.blockNumber
              return deltaBlock !== 0 ? deltaBlock : b.logIndex - a.logIndex;
            })
            .map(l => {
              let r = this.mgr_contract.interface.events.WithdrawalAvailable.parse(l.topics, l.data)
              return {
                blockNumber: l.blockNumber,
                amount: ethers.utils.formatEther(r.amount)
              }
            })
          this.loadingEarnings = false
        }).catch(e => {
          console.log(e)
          this.addMessage('danger', 'Error occurred during fetching earnings')
          this.loadingEarnings = false
        })
      },
      reloadWithdrawals() {
        this.loadingWithdrawals = true
        let topicAddress = '0x000000000000000000000000' + this.signer.address.substring(2)
        Promise.all([
          this.mgr_contract.functions.pendingWithdrawals(this.signer.address),
          this.provider.getLogs({
            fromBlock: 0,
            toBlock: 'latest',
            address: process.env.VUE_APP_D_MGR_ADDRESS,
            topics: [
              this.mgr_contract.interface.events.WithdrawalPayed.topics[0],
              topicAddress
            ]
          })
        ]).then(res => {
          this.pendingWithdrawal = res[0]
          this.listWithdrawals = res[1].filter(e => !e.removed)
            .sort((a, b) => {
              //desc sorting
              let deltaBlock = b.blockNumber - a.blockNumber
              return deltaBlock !== 0 ? deltaBlock : b.logIndex - a.logIndex;
            })
            .map(l => {
              let r = this.mgr_contract.interface.events.WithdrawalPayed.parse(l.topics, l.data)
              return {
                blockNumber: l.blockNumber,
                amount: ethers.utils.formatEther(r.amount)
              }
            })
          this.loadingWithdrawals = false
        }).catch(e => {
          console.log(e)
          this.addMessage('danger', 'Error occurred during fetching withdrawals')
          this.loadingWithdrawals = false
        })
      },
      reloadDividends() {
        this.loadingDividends = true
        this.provider.getLogs({
          fromBlock: 0,
          toBlock: 'latest',
          address: process.env.VUE_APP_D_MGR_ADDRESS,
          topics: this.mgr_contract.interface.events.DividendPayment.topics
        }).then(logs => {
          this.listDividends = logs.filter(e => !e.removed)
            .sort((a, b) => {
              //desc sorting
              let deltaBlock = b.blockNumber - a.blockNumber
              return deltaBlock !== 0 ? deltaBlock : b.logIndex - a.logIndex;
            })
            .map(l => {
              let r = this.mgr_contract.interface.events.DividendPayment.parse(l.topics, l.data)
              return {
                blockNumber: l.blockNumber,
                incomePayment: ethers.utils.formatEther(r.paymentPerShare.mul(this.totalSupply))
              }
            })
          this.loadingDividends = false
        }).catch(e => {
          console.error(e)
          this.addMessage('danger', 'Error occurred during fetching dividends')
          this.loadingDividends = false
        })
      },
      initItems() {
        this.mgr_contract = new ethers.Contract(process.env.VUE_APP_D_MGR_ADDRESS, process.env.VUE_APP_D_MGR_ABI, this.signer)
        this.tkn_contract = new ethers.Contract(process.env.VUE_APP_D_TKN_ADDRESS, process.env.VUE_APP_D_TKN_ABI, this.signer)

        Promise.all([
          this.tkn_contract.functions.totalSupply(),
          this.tkn_contract.functions.balanceOf(this.signer.address),
          // this.mgr_contract.functions.pendingWithdrawals(this.signer.address)
        ]).then(res => {
          this.totalSupply = res[0]
          this.balance = res[1]
          // this.pendingWithdrawal = res[2]
          this.reloadDividends()
          this.reloadEarnings()
          this.reloadWithdrawals()
        }).catch(e => {
          this.addMessage('danger', 'Error occurred during reading blockchain')
          console.log(e)
        })
      },
      withdraw() {
        this.ethWithdrawing = true
        this.mgr_contract.functions.withdrawDividend()
          .then(tx => {
            // console.log('withdraw', tx)
            this.provider.waitForTransaction(tx.hash).then(tx => {
              this.ethWithdrawing = false
              // console.log('withdraw completed', tx)
              this.reloadWithdrawals()
            })
          }, e => {
            this.ethWithdrawing = false
          })
      },
    },
    mounted() {
      if (this.isWeb3) {
        /* eslint-disable no-undef */
        this.provider = new ethers.providers.Web3Provider(this.getCurrentProvider, this.getNetwork)
        /* eslint-enable no-undef */
        // this.network = this.provider.name
        // console.log(this.network)
        this.provider.listAccounts().then(accs => {
          this.signer = this.provider.getSigner(accs[0])
          // console.log(this.signer)
        })
      } else {
        this.provider = ethers.providers.getDefaultProvider('homestead')
      }
      // console.log(this.provider)
      // this.initItems()
    }
  }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only lang="sass"-->
<!--<style scoped>-->

<!--</style>-->
