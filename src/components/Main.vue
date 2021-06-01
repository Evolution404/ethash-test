<template>
  <div class="main">
    <el-form :inline="true" class="demo-form-inline"
      v-loading.fullscreen.lock="fullscreenLoading"
      >
      <el-form-item label="校验的区块号">
        <el-input-number v-model="number" placeholder="区块号"></el-input-number>
      </el-form-item>
    </el-form>
    <el-button @click="getHeader" type="primary">获取主网数据</el-button>
    <el-table
      :data="header"
      :row-style="rowStyle">
      style="width: 100%">
      <el-table-column
        align="center"
        prop="key"
        label="名称"
        >
      </el-table-column>
      <el-table-column
        prop="value"
        label="值"
        align="center"
        >
        <template slot-scope="scope">
          <div>
            <el-tag v-if="scope.row.prefix" size="medium">{{ scope.row.prefix }}</el-tag>
          </div>
          {{scope.row.value}}
        </template>
      </el-table-column>
    </el-table>
    <div style="margin-top: 15px">
      <el-button @click="calc" type="primary">计算rlp编码和封装哈希</el-button>
    </div>
    <el-table
      :data="rlpAndHash"
      >
      style="width: 100%">
      <el-table-column
        align="center"
        prop="key"
        label="名称"
        >
      </el-table-column>
      <el-table-column
        prop="value"
        label="值"
        align="center"
        >
        <template slot-scope="scope">
          <div>
            <el-tag v-if="scope.row.prefix" size="medium">{{ scope.row.prefix }}</el-tag>
          </div>
          {{scope.row.value}}
        </template>
      </el-table-column>
    </el-table>
    <el-button style="margin-top:15px" @click="verify" type="primary">校验</el-button>
    <el-row style="margin-top:15px">
      <el-col :offset="8" :span="8">
        <el-progress :text-inside="true" :stroke-width="24" :percentage="percentage" status="success"></el-progress>
      </el-col>
    </el-row>
    <el-table
      :data="verifyTable"
      :row-style="rowColor"
      >
      style="width: 100%">
      <el-table-column
        align="center"
        prop="key"
        label="名称"
        >
      </el-table-column>
      <el-table-column
        prop="value"
        label="值"
        align="center"
        >
        <template slot-scope="scope">
          <div>
            <el-tag v-if="scope.row.prefix" size="medium">{{ scope.row.prefix }}</el-tag>
          </div>
          {{scope.row.value}}
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<script>
import Ethash from '@ethereumjs/ethash'
import { getEpoc, getCacheSize, getFullSize, bufReverse, getSeed } from '@ethereumjs/ethash/dist/util'
import Web3 from 'web3/dist/web3.min.js'
import { BlockHeader } from '@ethereumjs/block'
import * as rlp from 'rlp'
import BigNumber from "bignumber.js"
import CacheHeader from "../CacheHeader"
import { BN, keccak, keccak256, rlphash, zeros, TWO_POW256 } from 'ethereumjs-util'

export default {
  name: 'Main',
  data () {
    return {
      number:"1285",
      header:[],
      headerData:{},
      headerObj:{},
      rlpAndHash: [],
      verifyTable: [],

      sealHash:"",
      nonce:"",

      hash:"d4498a5c72396ed9982001ee6c970448dc0e2702bfb23a1c3dad80759302106f",
      nonce:"8c0f737c734f2872",

      ethash: new Ethash(require('level-mem')()),
      fullscreenLoading: false,
      percentage:0,
    }
  },
  methods:{
    getHeader:function(){
      this.fullscreenLoading = true
      let callback = headerData=>{
        CacheHeader[this.number]=headerData
        headerData.difficulty = Number(headerData.difficulty)
        headerData.uncleHash = headerData.sha3Uncles
        headerData.coinbase = headerData.miner
        headerData.transactionsTrie = headerData.transactionsRoot
        headerData.receiptTrie = headerData.receiptsRoot
        headerData.bloom = headerData.logsBloom
        this.header = [
          {key:'Parent Hash',value:headerData.parentHash,in:true},
          {key:'Uncle Hash',value:headerData.uncleHash,in:true},
          {key:'Coinbase',value:headerData.coinbase,in:true},
          {key:'State Root',value:headerData.stateRoot,in:true},
          {key:'Transactions Trie',value:headerData.transactionsRoot,in:true},
          {key:'Receipt Trie',value:headerData.receiptsRoot,in:true},
          {key:'Bloom',value:headerData.logsBloom.slice(0,60)+"...",in:true},
          {key:'Difficulty',value:headerData.difficulty,in:true},
          {key:'Number',value:headerData.number,in:true},
          {key:'Gas Limit',value:headerData.gasLimit,in:true},
          {key:'Gas Used',value:headerData.gasUsed,in:true},
          {key:'Timestamp',value:headerData.timestamp,in:true},
          {key:'Extra Data',value:headerData.extraData,in:true},
          {key:'Mix Hash',value:headerData.mixHash},
          {key:'Nonce',value:headerData.nonce},
          {key:'Hash',prefix:"keccak256(FullRLP)",value:headerData.hash},
        ]
        this.headerData = headerData
        this.headerObj = BlockHeader.fromHeaderData(headerData)
        this.nonce = headerData.nonce.slice(2)
        this.fullscreenLoading = false
      }
      if(CacheHeader[this.number]){
        callback(CacheHeader[this.number])
      }else{
        let web3 = new Web3("wss://mainnet.infura.io/ws/v3/38ea113e7a70462cb577d4822ca4694a");
        web3.eth.getBlock(this.number).then(callback)
      }
    },
    calc: function(){
      let ethash = this.ethash
      let headerRlp = rlp.encode(this.headerObj.raw().slice(0, -2))
      let sealHash = ethash.headerHash(this.headerObj.raw()).toString("hex")
      this.sealHash = sealHash
      this.rlpAndHash = [
        {key:'RLP',value:headerRlp.toString('hex')},
        {key:'Seal Hash',prefix:"keccak256(RLP)",value:'0x'+sealHash},
      ]
    },
    verify: function (){
      const xor = require('buffer-xor')
      let TWO_POW256 = new BigNumber('0x10000000000000000000000000000000000000000000000000000000000000000')
      let number = this.number
      let hash = Buffer.from(this.sealHash, 'hex')
      let nonce = Buffer.from(this.nonce, 'hex')
      let headerData = this.headerData
      let difficulty = Number(headerData.difficulty)
      let vueobj = this

      this.verifyTable = [
        {key:"Block Number",value:headerData.number,color:"#f0f9eb"},
        {key:"Nonce",value:headerData.nonce,color:"#f0f9eb"},
        {key:"Seal Hash",value:this.sealHash,color:"#f0f9eb"},
      ]
      let ethash = this.ethash
      let reverseNonce = bufReverse(nonce).toString("hex")
      let epoc = getEpoc(this.number)
      let cacheSize = getCacheSize(epoc)
      let fullSize = getFullSize(epoc)
      this.verifyTable.push({key:"Reverse Nonce",value:reverseNonce,color:"oldlace"})
      this.verifyTable.push({key:"Verify Seed",prefix:"keccak512(SealHash+ReverseNonce)",value:keccak(Buffer.from(this.sealHash+reverseNonce,"hex"),512).toString("hex"),color:"oldlace"})
      this.verifyTable.push({key:"Epoch",value:epoc,color:"oldlace"})
      this.verifyTable.push({key:"Cache Size",value:cacheSize,color:"oldlace"})
      this.verifyTable.push({key:"Full Size",value:fullSize,color:"oldlace"})


      let seed = getSeed(Buffer.allocUnsafe(32).fill(0),0,epoc)
      this.verifyTable.push({key:"Seed",value:seed.toString("hex"),color:"oldlace"})

      let CACHE_ROUNDS = 3
      let HASH_BYTES = 64

      let progressCache=(cacheSize,seed,progress)=>{
        const n = Math.floor(cacheSize / HASH_BYTES)
        const total = n-1+n*CACHE_ROUNDS
        let o
        if(progress==0)
          o = [keccak(seed, 512)]
        else
          o = ethash.cache
        let start = Math.floor((total/100)*progress)
        let end = Math.floor((total/100)*(progress+1))
        if(progress>=99)
          end = total
        for(let i=start;i<end;i++){
          if(i<n-1)
            o.push(keccak(o[o.length - 1], 512))
          else{
            let a = i - (n-1)
            a = a % n
            const v = o[a].readUInt32LE(0) % n
            o[a] = keccak(xor(o[(a - 1 + n) % n], o[v]), 512)
          }
        }
        ethash.cache = o
        this.percentage = progress+1
        if(progress<99)
          setTimeout(progressCache,1,cacheSize,seed,progress+1)
        else
          verify()
      }

      let verify = ()=>{
        ethash.epoc=epoc
        let rs = ethash.run(hash, nonce, fullSize)
        this.verifyTable.push({key:"Mix Hash",value:rs.mix.toString("hex"),color:"#fef0f0"})
        let powStr = rs.hash.toString("hex")
        let powZeros = 0
        for(;powStr[powZeros]=='0';powZeros++);
        this.verifyTable.push({key:"POW",prefix:"keccak256(VerifySeed+MixHash)",value:powStr,color:"#fef0f0"})

        let difficultyTarget = TWO_POW256.div(difficulty).toString(16).split('.')[0]
        let targetZeros = 64-difficultyTarget.length
        for(;difficultyTarget.length<64;){
          difficultyTarget = "0"+difficultyTarget
        }
        this.verifyTable.push({key:"Difficulty Target",value:difficultyTarget,color:"#fef0f0"})
        this.$notify({
          title: '区块'+number+' 校验完成',
          message: 'pow:'+powZeros+' zeros,target:'+targetZeros+' zeros',
          type: 'success',
          duration: 0
        })
      }
      if(ethash.epoc!=epoc)
        setTimeout(progressCache,1,cacheSize,seed,0)
      else
        verify()
    },
    rowStyle: function({row,rowIndex}){
      if(row.in)
        return {background:"#f0f9eb"}
      if(row.key!="Hash")
        return {background:"oldlace"}
      return {background:"#fef0f0"}
    },
    rowColor: function({row,rowIndex}){
      if (row.color){
        return {background:row.color}
      }
      return {}
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h1, h2 {
  font-weight: normal;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
