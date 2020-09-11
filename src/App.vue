<template>
  <v-app>
    <v-main>
      <div style="text-align: center; margin-top: 40px">Last Refresh: {{ lastRefresh }}</div>
      <div style="height: 100%; display: flex; align-items: center; justify-items: center; justify-content: center; flex-wrap: wrap;">
        <v-card flat outlined class="ma-5" width="350" v-for="vault in vaults" :key="vault.tokenTicker" style="margin-top: 20px;">
          <v-list-item two-line>
            <v-list-item-content>
              <v-list-item-title class="text-h5 text--black">{{ vault.tokenTicker }}</v-list-item-title>
              <div class="font-weight-bold" v-bind:class="[vault.ROI_day < 0 ? 'red--text' : 'green--text']">{{ vault.vaultHoldingsGrowthNegative ? '' : '+' }}{{ vault.ROI_day }}% daily</div>
            </v-list-item-content>
          </v-list-item>

          <v-card-text>
            <v-row align="center">
              <v-col class="text-h5" cols="12">
                <div>{{ vault.myVaultTokenInUnderlyingTokenAmount }} {{ vault.tokenTicker }}</div>
                <div class="text-h6 green--text">+{{ vault.myDailyGains }} {{ vault.tokenTicker }} daily</div>
              </v-col>
              <v-col class="text-h5" cols="12">
                <div>{{ vault.myVaultTokenInUnderlyingTokenAmountInFiat }} €</div>
                <div class="text-h6 green--text">+{{ vault.myDailyGainsInFiat }} € daily</div>
              </v-col>
            </v-row>
          </v-card-text>

          <v-divider></v-divider>

          <v-list class="transparent">
            <v-list-item>
              <v-list-item-title style="overflow: visible">Invested Amount</v-list-item-title>
              <v-list-item-subtitle class="text-right"> {{ vault.myVaultTokenAmountStart }} {{ vault.tokenTicker }}</v-list-item-subtitle>
            </v-list-item>

            <v-list-item>
              <v-list-item-title style="overflow: visible">Invested since</v-list-item-title>
              <v-list-item-subtitle class="text-right"> {{ (vault.myVaultInvestTimeInHours - (vault.myVaultInvestTimeInHours % 24)) / 24 }}d {{ vault.myVaultInvestTimeInHours % 24 }}h</v-list-item-subtitle>
            </v-list-item>

            <v-list-item>
              <v-list-item-title class="font-weight-bold">Total Earnings</v-list-item-title>
              <v-list-item-subtitle class="text-right font-weight-bold"> {{ vault.myVaultReturnSinceStart.toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 4 }) }} {{ vault.tokenTicker }} </v-list-item-subtitle>
            </v-list-item>

            <v-list-item>
              <v-list-item-title class="font-weight-bold">Total Earnings in €</v-list-item-title>
              <v-list-item-subtitle class="text-right font-weight-bold"> {{ (vault.myVaultReturnSinceStart * vault.tokenPrice).toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }} € </v-list-item-subtitle>
            </v-list-item>

            <v-list-item>
              <v-list-item-title style="overflow: visible">Avg. Earnings per Day</v-list-item-title>
              <v-list-item-subtitle class="text-right"> {{ ((vault.myVaultReturnSinceStart / vault.myVaultInvestTimeInHours) * 24).toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 4 }) }} {{ vault.tokenTicker }} </v-list-item-subtitle>
            </v-list-item>

            <v-list-item>
              <v-list-item-title style="overflow: visible">Avg. Earnings per Day in €</v-list-item-title>
              <v-list-item-subtitle class="text-right"> {{ ((vault.myVaultReturnSinceStart / vault.myVaultInvestTimeInHours) * 24 * vault.tokenPrice).toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }} €</v-list-item-subtitle>
            </v-list-item>
          </v-list>

          <v-divider></v-divider>

          <v-list class="transparent">
            <v-list-item>
              <v-list-item-title style="overflow: visible"
                >Vault Holdings <span class="font-weight-bold" v-bind:class="[vault.vaultHoldingsGrowthNegative ? 'red--text' : 'green--text']">{{ vault.vaultHoldingsGrowthNegative ? '' : '+' }}{{ vault.vaultHoldingsGrowth }}%</span></v-list-item-title
              >
              <v-list-item-subtitle class="text-right"> {{ vault.vaultHoldings }} </v-list-item-subtitle>
            </v-list-item>

            <v-list-item>
              <v-list-item-title>{{ vault.tokenTicker }} Value </v-list-item-title>
              <v-list-item-subtitle class="text-right"> {{ vault.tokenPrice }} € </v-list-item-subtitle>
            </v-list-item>
          </v-list>
        </v-card>
      </div>
    </v-main>
  </v-app>
</template>

<script>
import { ethers } from 'ethers';
import constants from './helpers/constants';

let provider;
if (window.ethereum) provider = new ethers.providers.Web3Provider(window.ethereum);
else provider = new ethers.providers.JsonRpcProvider(atob('aHR0cHM6Ly9tYWlubmV0LmluZnVyYS5pby92My9mN2Q1YjkwMzY3MzY0YmFkYWNhZDI5Njg5OWYyMTMxYQ==')); //doesn't work with Incognito Mode or Safari :(

export default {
  name: 'App',
  data() {
    return {
      vaults: [],
      lastRefresh: '',
    };
  },
  created() {
    this.queryVaults();
    this.lastRefresh = new Date().toLocaleTimeString('de-DE');
    setInterval(() => {
      this.queryVaults();
      this.lastRefresh = new Date().toLocaleTimeString('de-DE');
    }, 60000);
  },
  methods: {
    async queryVaults() {
      const YEARN_VAULT_CONTROLLER = new ethers.Contract(constants.YEARN_VAULT_CONTROLLER_ADDR, constants.YEARN_VAULT_CONTROLLER_ABI, provider);
      const now = Math.round(Date.now() / 1000);

      const dayBlocks = [];
      for (let day = 1; day < 2; day++) {
        dayBlocks.push(parseInt(await this.getBlockNumberFromTimestamp(now - 60 * 60 * 24 * day)));
      }

      const prices = await this.lookUpPrices(['ethereum', 'dai']);

      const vaultCompatibleTokens = [
        ['DAI', prices['dai'].eur, constants.DAI_TOKEN_ADDR, <TOKEN_AMOUNT>, <INVESTMENT_TIME_AS_TIMESTAMP>], //e.g. ['DAI', prices['dai'].eur, constants.DAI_TOKEN_ADDR, 34000.5, 15995d2000000]
        ['ETH', prices['ethereum'].eur, constants.WETH_TOKEN_ADDR, <TOKEN_AMOUNT>, <INVESTMENT_TIME_AS_TIMESTAMP>],
      ];

      this.vaults = await Promise.all(
        vaultCompatibleTokens.map(async function(token) {
          const tokenTicker = token[0];
          const tokenPrice = token[1];
          const tokenAddr = token[2];

          const vaultAddress = await YEARN_VAULT_CONTROLLER.vaults(tokenAddr);

          const tokenContract = new ethers.Contract(tokenAddr, constants.ERC20_ABI, provider);
          const vaultContract = new ethers.Contract(vaultAddress, constants.YEARN_VAULT_ABI, provider);

          const currentPricePerFullShare = await vaultContract.getPricePerFullShare();
          const decimals = parseInt(await tokenContract.decimals());

          const vaultHoldings = (await vaultContract.balance()) / 10 ** decimals;

          let vaultHoldingsDayAgo = 0;
          try {
            vaultHoldingsDayAgo = (await vaultContract.balance({ blockTag: dayBlocks[0] })) / 10 ** decimals;
          } catch (e) {
            console.error(e);
          }

          const myVaultTokenAmount = (await vaultContract.balanceOf(constants.MY_ADDRESS)) / 10 ** (await vaultContract.decimals());
          const myVaultTokenInUnderlyingTokenAmount = (myVaultTokenAmount * currentPricePerFullShare) / 1e18;

          let ROI_day = 0;

          try {
            const pastPricePerFullShare = await vaultContract.getPricePerFullShare({ blockTag: dayBlocks[0] });
            ROI_day = (currentPricePerFullShare / pastPricePerFullShare - 1) * 100;
          } catch (e) {
            console.error(e);
          }

          const myDailyGains = (myVaultTokenInUnderlyingTokenAmount * ROI_day) / 100;

          return {
            tokenTicker: tokenTicker,
            tokenAddr: tokenAddr,
            tokenPrice: tokenPrice,
            tokenContractInstance: tokenContract,
            vaultTicker: await vaultContract.symbol(),
            vaultHoldings: vaultHoldings.toLocaleString('de-DE', { maximumFractionDigits: 0 }),
            vaultHoldingsDayAgo: vaultHoldingsDayAgo.toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 2 }),
            vaultHoldingsGrowth: (((vaultHoldings - vaultHoldingsDayAgo) / vaultHoldingsDayAgo) * 100).toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 3 }),
            vaultHoldingsGrowthNegative: (((vaultHoldings - vaultHoldingsDayAgo) / vaultHoldingsDayAgo) * 100 < 0) | 0,
            vaultHoldingsInFiat: (vaultHoldings * tokenPrice).toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 2 }),
            myVaultTokenAmountStart: token[3].toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 4 }),
            myVaultTokenAmount: myVaultTokenAmount.toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 4 }),
            myVaultTokenInUnderlyingTokenAmount: myVaultTokenInUnderlyingTokenAmount.toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 4 }),
            myVaultTokenInUnderlyingTokenAmountInFiat: (myVaultTokenInUnderlyingTokenAmount * tokenPrice).toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 2 }),
            myVaultInvestTimeInHours: Math.ceil(Math.abs(new Date() - new Date(token[4])) / (1000 * 60 * 60)),
            myVaultReturnSinceStart: myVaultTokenInUnderlyingTokenAmount - token[3],
            myDailyGains: myDailyGains.toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 2 }),
            myDailyGainsInFiat: (myDailyGains * tokenPrice).toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 2 }),
            ROI_day: ROI_day.toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 3 }),
          };
        })
      );
    },

    async lookUpPrices(token_array) {
      return new Promise((resolve, reject) => {
        try {
          const tokens = token_array.join('%2C');
          const url = 'https://api.coingecko.com/api/v3/simple/price?ids=' + tokens + '&vs_currencies=eur';
          const options = {
            method: 'GET',
            headers: {
              Accept: 'application/json',
              'Content-Type': 'application/json;charset=UTF-8',
            },
          };
          fetch(url, options).then((response) => {
            resolve(response.json());
          });
        } catch (err) {
          return reject(err);
        }
      });
    },

    async getBlockNumberFromTimestamp(timestamp) {
      return new Promise((resolve, reject) => {
        try {
          const url = `https://api.etherscan.io/api?module=block&action=getblocknobytime&timestamp=${timestamp}&closest=before&apikey=XRFWK1IDBR545CXNJ6NQSYAVINUQB7IDV1`;
          const options = {
            method: 'GET',
            headers: {
              Accept: 'application/json',
              'Content-Type': 'application/json;charset=UTF-8',
            },
          };
          fetch(url, options)
            .then((response) => response.json())
            .then((responseJson) => resolve(responseJson.result));
        } catch (err) {
          return reject(err);
        }
      });
    },
  },
};
</script>

<style>
#app {
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>
