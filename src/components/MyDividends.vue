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
        <p><small v-if="contractState" class="text-muted">Contract: <b-link :href="'https://etherscan.io/address/'+tkn_contract.address">{{tkn_contract.address}}</b-link></small></p>
        <!--<p class="card-text">You owned <strong>{{getBalance}}</strong> of <strong>{{getTotalSupply}}</strong> dividend tokens</p>-->
        <h5 class="fix">You owned <b-badge variant="success" pill>{{getBalance}}</b-badge> of <b-badge pill class="fix">{{getTotalSupply}}</b-badge> UnicornGO Dividend Tokens (UDT)</h5>
      </b-card>
      <b-card border-variant="primary" text-variant="white" sub-title="WITHDRAWAL" class="mb-2">
        <p><small v-if="contractState" class="text-muted">Contract: <b-link :href="'https://etherscan.io/address/'+mgr_contract.address">{{mgr_contract.address}}</b-link></small></p>

        <!--<h6 v-if="contractState"><small>Contract:</small> <b-badge>{{mgr_contract.address}}</b-badge></h6>-->
        <!--<p class="card-text">You can withdraw <strong>{{getPendingWithdrawal}}&Xi;</strong></p>-->
        <h5 class="fix">You can withdraw <b-badge variant="primary" pill>{{getPendingWithdrawal}}<svg class="symbol"><use xlink:href="#ethereum"></use></svg></b-badge></h5>
        <b-button variant="outline-primary" :disabled="withdrawState" @click="withdraw">Withdraw</b-button>
      </b-card>
    </b-card-group>
    <b-card-group deck>
      <b-card border-variant="warning" text-variant="white" sub-title="YOUR RECENT WITHDRAWALS" class="mb-2">
        <template v-if="isWithdrawals">
          <h5 class="fix">Total:&nbsp;<b-badge variant="warning" pill>{{getWithdrawalsSum}}<svg class="symbol"><use xlink:href="#ethereum"></use></svg></b-badge></h5>
          <b-table striped small fixed :items="getWithdrawals"></b-table>
        </template>
        <p class="card-text" v-else>list is empty</p>
        <EllipsisSpinner v-if="loadingWithdrawals"/>
      </b-card>
      <b-card border-variant="danger" text-variant="white" sub-title="YOUR RECENT EARNINGS" class="mb-2">
        <!--<b-table striped small fixed :items="listEarnings" v-if="listEarnings.length"></b-table>-->
        <template v-if="isEarnings">
          <h5 class="fix">Total:&nbsp;<b-badge variant="danger" pill>{{getEarningsSum}}<svg class="symbol"><use xlink:href="#ethereum"></use></svg></b-badge></h5>
          <b-table striped small fixed :items="getEarnings"></b-table>
        </template>

        <p class="card-text" v-else>list is empty</p>
        <EllipsisSpinner v-if="loadingEarnings"/>
      </b-card>
      <b-card border-variant="info" text-variant="white" sub-title="TOTAL RECENT DIVIDENDS INCOME" class="mb-2">
        <!--<h5>Total sum: {{getDividendsSum}}</h5>-->
        <!--<b-table striped small fixed :items="getDividends"></b-table>-->
        <template v-if="isDividends">
          <h5 class="fix">Total:&nbsp;<b-badge variant="info" pill>{{getDividendsSum}}<svg class="symbol"><use xlink:href="#ethereum"></use></svg></b-badge></h5>
          <b-table striped small fixed :items="getDividends"></b-table>
        </template>
        <p class="card-text" v-else>list is empty</p>
        <EllipsisSpinner v-if="loadingDividends"/>
      </b-card>
    </b-card-group>
    <div style="display: none">
      <svg id="ethereum" viewBox="0 0 2.646 3.704" width="100%" height="100%"><title>ethereum</title><path d="M1.323 0l-.025.084v2.45l.025.024 1.137-.672z"></path><path d="M1.323 2.558V0L.186 1.886zm0 .216l-.014.017v.872l.014.041L2.46 2.102z"></path><path d="M1.323 2.774L.186 2.102l1.137 1.602z"></path></svg>
      <!--<svg id="candy" viewBox="0 0 2.381 3.704" width="100%" height="100%"><title>candy</title><path d="M1.794.48s.014-.657-.347-.12C1.357.244.992-.05.897.007c-.096.058.18.57.249.824-.377.055-.717.365-.834.803-.118.438.022.876.32 1.112-.185.185-.688.455-.627.589.06.133.518.025.653-.03.045.645.36.07.36.07s.54.528.506.247c-.03-.262-.178-.535-.272-.709.378-.052.72-.363.838-.803.118-.44-.023-.88-.325-1.114.17-.104.433-.267.591-.478.17-.226-.562-.038-.562-.038zm-.572 2.664c.017.14-.307-.202-.307-.202-.067.027-.3.13-.343.078-.037-.046.087-.116.18-.196a.81.81 0 0 0 .38.097.482.482 0 0 1 .09.223zm.062-2.096a.015.015 0 0 1-.013.011c-.318.037-.595.297-.689.647-.094.35.016.714.274.905a.015.015 0 0 1 .003.02.013.013 0 0 1-.017.006.708.708 0 0 1-.364-.385.913.913 0 0 1-.033-.582.913.913 0 0 1 .32-.489.709.709 0 0 1 .51-.15c.007.001.01.01.009.017zM1.855.78a.482.482 0 0 1-.19.148.81.81 0 0 0-.376-.106C1.248.708 1.176.585 1.23.563c.064-.024.214.182.258.24 0 0 .452-.135.367-.022z" stroke-width="17.65"></path></svg>-->
    </div>
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

      getDividendsSum() {
        return ethers.utils.formatEther(this.listDividends.reduce((sum, i) => sum.add(i.incomePayment), ethers.utils.bigNumberify(0)))
      },
      getDividends() {
        return this.listDividends.map(i => {
          return {
            blockNumber: i.blockNumber,
            amount: ethers.utils.formatEther(i.incomePayment)
          }
        })
      },
      isDividends() {
        return this.listDividends.length > 0
      },

      getWithdrawalsSum() {
        return ethers.utils.formatEther(this.listWithdrawals.reduce((sum, i) => sum.add(i.amount), ethers.utils.bigNumberify(0)))
      },
      getWithdrawals() {
        return this.listWithdrawals.map(i => {
          return {
            blockNumber: i.blockNumber,
            amount: ethers.utils.formatEther(i.amount)
          }
        })
      },
      isWithdrawals() {
        return this.listWithdrawals.length > 0
      },
      getEarningsSum() {
        return ethers.utils.formatEther(this.listEarnings.reduce((sum, i) => sum.add(i.amount), ethers.utils.bigNumberify(0)))
      },
      getEarnings() {
        return this.listEarnings.map(i => {
          return {
            blockNumber: i.blockNumber,
            amount: ethers.utils.formatEther(i.amount)
          }
        })
      },
      isEarnings() {
        return this.listEarnings.length > 0
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
                // amount: ethers.utils.formatEther(r.amount)
                amount: r.amount
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
                // amount: ethers.utils.formatEther(r.amount)
                amount: r.amount
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
                // incomePayment: ethers.utils.formatEther(r.paymentPerShare.mul(this.totalSupply))
                incomePayment: r.paymentPerShare.mul(this.totalSupply)
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
<style lang="scss" scoped>

  h5.fix, .h5.fix {
    font-size: 1.2rem;
    .badge {
      font-size: 90%;
    }
  }
  .symbol {
    display: inline-block;
    position: relative;
    fill: currentColor;
    width: 1rem;
    height: 1rem;
    margin-top: -0.2rem;
    /*margin-right: 0.3rem;*/
    /*margin-left: 0.3rem;*/
    vertical-align: middle;
  }
</style>
