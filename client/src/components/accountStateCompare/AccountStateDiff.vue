<template>
<div class="accountStateDiff">
  <!-- <v-btn @click="saveRecord">Save Record</v-btn> -->

  <difference-table 
    @saveRecord="saveRecord"
    :dataRows="linesForDataRecord"
    :dataRecord="dataRecord"
    description="diffs"
  />
</div>
</template>

<script>
// import Currency from '@/components/Currency'
// import Item from "@/components/Item"
import {mapGetters, mapActions} from 'vuex'
import DifferenceTable from './DifferenceTable.vue'

export default {
  name: "AccountStateDiff",
  components:{
    // Item,
    // Currency,
    DifferenceTable
  },
  props: {  
  },
 
    data: () => {
    return {
      
    }
  },
  computed: {
    ...mapGetters([
      'accountStateFromApi',
      'accountStateFromSave',
      'itemTotalsFromApi',
      'itemTotalsFromSave',
      'currencyLookup',
      'itemLookup',
      'user',
      'selectedApiKey',
    ]),
    itemDiffs() {
      return this.calculateItemTotalDiffs(this.itemTotalsFromSave, this.itemTotalsFromApi)
    },
    currDiffs() {
      return this.calculateCurrencyTotalDiffs(this.accountStateFromSave.wallet, this.accountStateFromApi.wallet)
    },
    linesForDataRecord() {
      let lines = []
      this.currDiffs.forEach((currDiff) => {
        const newLine = {
          quantity: currDiff.diffCount,
          element_id: currDiff.id,
          element_type: 'Currency',
        }
        lines.push(newLine)
      })

      Object.keys(this.itemDiffs).forEach((item_id) => {
        const newLine = {
          quantity: this.itemDiffs[item_id].diffCount,
          element_id: this.itemDiffs[item_id].id,
          element_type: 'Item',
          upgrades: this.itemDiffs[item_id].upgrades.join(',')
        }
        lines.push(newLine)
      })
      return lines
    },
    dataRecord() {
      return  {
        // user_id: this.user.id,
        gw2_account_name: this.selectedApiKey.account_name,
        start_time: this.accountStateFromSave.timestampOfState,
        end_time: this.accountStateFromApi.timestampOfState,
        lines: this.linesForDataRecord,
        tags: [],
      }
    }
  },
  methods: {
    ...mapActions('newDataRecord', [
      'updateNewDataRecord',
      'setNewDataRecordLines',
    ]),
    saveRecord() {
      const newDataRecord = {
        user_id: this.user.id,
        gw2_account_name: this.selectedApiKey.account_name,
        start_time: this.accountStateFromSave.timestampOfState,
        end_time: this.accountStateFromApi.timestampOfState,
        lines: this.linesForDataRecord
      }
      this.updateNewDataRecord(newDataRecord)

      //now also get the lines into 
      // let lines = this.linesForDataRecord
      this.setNewDataRecordLines(this.linesForDataRecord)

      this.$router.push('saveNewDataRecord')
    },

    calculateCurrencyTotalDiffs(beforeWallet, afterWallet) {
      let currencyTotalDiffs = {}
      Object.keys(this.currencyLookup).forEach((currId) => {
        currencyTotalDiffs[currId] = {}
      })

      beforeWallet.forEach(x => {
        currencyTotalDiffs[x.id].beforeCount = x.value
      })
      afterWallet.forEach(x => {
        currencyTotalDiffs[x.id].afterCount = x.value
      })

      let result = []
      Object.keys(this.currencyLookup).forEach((currId) => {
        let bef = 0, aft = 0
        if (currencyTotalDiffs[currId].beforeCount) {bef = currencyTotalDiffs[currId].beforeCount}
        if (currencyTotalDiffs[currId].afterCount) {aft = currencyTotalDiffs[currId].afterCount}
        if (bef !== aft) {
          result.push({
            id: parseInt(currId),
            diffCount: aft-bef,
          })
        }
      })

      result.sort((a,b) => {
        return this.currencyLookup[a.id].order - this.currencyLookup[b.id].order
      })
      return result
    },
    calculateItemTotalDiffs(beforeTotals, afterTotals) {
      if (!beforeTotals || !afterTotals) { return null }


      let itemIdSet = new Set()
      let itemTotalDiffs = {}
      Object.keys(beforeTotals).forEach(x => {
        itemIdSet.add(x)
      })
      Object.keys(afterTotals).forEach(x => {
        itemIdSet.add(x)
      })
      Array.from(itemIdSet).forEach((itemId) => {
        let beforeCount = 0, afterCount = 0
        if (beforeTotals[itemId]) { beforeCount = beforeTotals[itemId].count }
        if (afterTotals[itemId]) { afterCount = afterTotals[itemId].count }
        let diffCount = afterCount - beforeCount
        if (diffCount !== 0) {
          let [id, ...upgrades] = itemId.split('_')
          itemTotalDiffs[itemId] = {
            id: parseInt(id),
            diffCount,
            upgrades: upgrades,
            beforeSources: beforeTotals[itemId]?beforeTotals[itemId].sources:[],
            afterSources: afterTotals[itemId]?afterTotals[itemId].sources:[]
          }
        }
      })
      return itemTotalDiffs
    },

    sourceDetailString(beforeSources, afterSources) {
      return `    Before:\n${beforeSources.join('\n')}\n    After:\n${afterSources.join('\n')}`
    }
  }

}
</script>

<style>
  /* #diffItems{
    display: grid;
    justify-content: center;
    grid-template: repeat(auto-fill, 64px) / repeat(10 , 64px);
  } */
</style>