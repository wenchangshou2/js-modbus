<template>
  <div>
    <h1>PLC控制调试</h1>
  
    <div class="header">
      设备号:
      <input type="number" ref="id" value="1" /> ip地址:
      <input type="text" ref="ipAddr" value="192.168.86.171" /> 端口号:
      <input type="number" ref="Port" value="502" />
    </div>
    <div class="model">
      模式：
      <input type="radio" value="coils" v-model="model">coils</input>
      <input type="radio" value="singleCoil" v-model="model">DiscreteInputs</input>
      <input type="radio" value="holdingRegisters" v-model="model">HoldingRegisters</input>
      <input type="radio" value="inputRegisters" v-model="model">InputRegisters</input>
    </div>
    <div class="addr">
      起始地址:<input type="number" v-model="startAddr" ref="startAddr" value="0">
      寄存器数:<input type="number" ref="count" value="10">
    </div>
    <div>
      <span>是否自动读取
        <input type="checkbox" v-model="repeat" ref="repeatRadio">
      </span>
      </span>
      <button id="connect" @click="connect">连接</button>
      <button id="readCoils" @click="readStatus">读取</button>
          连接状态:{{connectStatus?'已连接':'未连接'}}
    </div>
    <ul>
      <li v-for="(item,index) in recv_data_list" :key="index" style="width:25%">
        <span v-if="model==='coils' || model==='singleCoil'" >
          <!-- {{`addr:${index}:`}} -->
          addr:{{getAddr(index)}}
          <input type="checkbox" id="checkbox" v-model="recv_data_list[index]" @click="writeDI(index,recv_data_list[index])">
        </span>
        <span v-else-if="model==='holdingRegisters'||model==='inputRegisters'">
        addr:{{getAddr(index)}}
          <input style="width:60px;text-align:center" type="text" :value="item" :ref="`AI${index}`"></input>
          <button @click="writeAO(index,item)">更改</button>
        </span>
      </li>
    </ul>
  </div>
</template>
<style>
ul li {
  list-style-type: none;

  height: 25%;

  float: left;
}
</style>

<script>
var modbus = require('jsmodbus')
let client
let timing = 0
export default {
  data() {
    return {
      recv_data_list: [],
      repeat: false,
      connectStatus: false,
      writeStatus: false,
      model: 'coils',
      modelAddr: '000',
      startAddr: 0
    }
  },
  watch: {
    repeat: function (state) {
      console.log('state:', state);
      if (!this.connectStatus && state) {
        alert('未连接设备，不能自动读取')
        this.repeat = false
        return
      }
      if (state) {
        console.log('setinterval');
        timing = setInterval(this.thingTask, 1000)
      } else {
        if (timing != 0) {
          clearInterval(timing)
          timing = 0
        }
      }
    },
    model: function (model) {
      this.recv_data_list = []
      if(model==='coils'){
        this.modelAddr='000'
      }else if(model==='singleCoil'){
        this.modelAddr='100'
      }else if(model==='holdingRegisters'){
        this.modelAddr='300'
      }else if(model==='inputRegisters'){
        this.modelAddr='400'
      }
    }
  },
  methods: {
    getAddr(index){
      let startAddr=parseInt(this.startAddr)
      let curr = startAddr + index
      if (curr < 10) {
        return this.modelAddr+'0'+curr
      }else{
        return this.modelAddr + curr
      }
    },
    readStatus() {
      let startAddr=this.$refs.startAddr.value;
      let count = this.$refs.count.value;
      let model = this.model
      console.log(model);
      if(this.model==='coils'){
        this.readCoils(startAddr, count);
      }else if(this.model==='singleCoil'){
        this.readSingleCoils(startAddr, count);
      }else if(model==='holdingRegisters'){
        this.readHoldingRegisters(startAddr, count);
      }else if(model==='inputRegisters'){
        this.readInputRegisters(startAddr, count);
      }
    },
    writeAO(index,value){
      if (!this.connectStatus) return
      console.log(this.$refs['AI1'][0].value);
      console.log('write ao', index, value)
      let val = this.$refs[`AI${index}`][0].value
      client.writeSingleRegister(index, val).then((resp) => {
        console.log(resp);
      }, console.error)
    },
    readHoldingRegisters(startAddr, count) {
      client.readHoldingRegisters(startAddr, count).then((resp) => {
        this.recv_data_list = resp['register']
        console.log(resp);
      })
      console.log('readholdingregisters');
    },
    readSingleCoils(startAddr, count) {
      if (this.connectStatus) {
        let self = this
        client.readDiscreteInputs(startAddr, count).then(function (resp) {
          console.log('singlecoils', resp);
          self.recv_data_list = resp['coils']
        })
      }
    },
    writeDI(index, val) {
      console.log('index', index, val);
      if (this.connectStatus) {
        client.writeSingleCoil(index, val).then(function (resp) {
          // alert('写入成功')
        }, console.error);
      }
    },
    thingTask() {
      let self = this
      let model = this.model
      let startAddr = this.$refs.startAddr.value
      let count = this.$refs.count.value
      if(!this.connectStatus) return
      if(model==='coils'){
        this.readCoils(startAddr, count)
      }else if(model==='singleCoil'){
        this.readSingleCoils(startAddr, count)
      }else if(model==='holdingRegisters'){
        this.readHoldingRegisters(startAddr, count)
      }else if(model==='inputRegisters'){
        this.readInputRegisters(startAddr, count)
      }
    },
    readInputRegisters(startAddr,count){
      console.log('readinputRegister');
      client.readInputRegisters(startAddr,count).then((resp)=>{
        this.recv_data_list = resp['register']
        console.log(resp);
      })
    },
    readCoils(startAddr, count) {
      console.log('readCoils', startAddr, count);
      let self = this
      client.readCoils(startAddr, count).then(function (resp) {
        self.recv_data_list = resp['coils']
      }, this.onError)
    },
    onConnect() {
      // alert('connect')
      this.connectStatus = true
    },
    writeCoils(index, value) {
    },
    onError(err) {
      this.connectStatus = false
    },
    onData(data) {
      // this.readCoils()
    },
    connect: function (e) {
      let id = this.$refs.id.value
      let host = this.$refs.ipAddr.value
      let port = this.$refs.Port.value
      client = modbus.client.tcp.complete({
        'host': host,
        'port': port,
        'autoReconnect': false,
        'reconnectTimeout': 1000,
        'timeout': 5000,
        'unitId': id
      })
      client.connect()
      client.on('connect', this.onConnect)
      client.on('error', this.onError)
      client.on('data', this.onData)
    }
  }

}
</script>
