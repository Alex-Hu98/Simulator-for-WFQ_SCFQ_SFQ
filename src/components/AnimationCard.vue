<template>
  <div>
    <a-row style="border-bottom: 1px dashed #ddd">
      <a-col :span="8" style="padding-bottom: 15px">
          <a-button type="link" @click="applyWFQ" block>WFQ</a-button>
      </a-col>
      <a-col :span="8" style="padding-bottom: 15px">
          <a-button type="link" @click="applySFQ" block>SFQ</a-button>
      </a-col>
      <a-col :span="8" style="padding-bottom: 15px">
          <a-button type="link" @click="applySCFQ" block>SCFQ</a-button>
      </a-col>
    </a-row>
    <a-row style="min-height: 400px" type="flex" justify="space-between">
      <a-col :span="14" style="padding-right: 10px">
        <!-- <a-button type="primary"  @click="finishNow" v-if="realTime >= 0">
          <template #icon shape="circle">
            <fast-forward-outlined/>
          </template>
        </a-button>
        -->

        <a-list item-layout="horizonal" :data-source="queueList">
          <template #renderItem="{ item, index }">
            <a-list-item>
              <div type="flex" style="width: 100%">
                <div>
                  <a-tag :color="$packageColor(index)"> Weight: {{ item.weight }}</a-tag>
                </div>
                <div style="width: 100%">
                  <p
                    style="
                      font-weight: bold;
                      font-size: 16px;
                      line-height: 30px;
                      text-align: right;
                    "
                  >
                    {{ `Flow #${index + 1}` }}
                  </p>
                  <a-list
                    :locale="{ emptyText: 'Empty list' }"
                    :grid="{ gutter: 4, xs: 1, sm: 2, lg: 8 }"
                    :data-source="item.list"
                    class="queue"
                  >
                    <template #renderItem="{ item }">
                      <a-list-item
                         v-if="item.arrTime <= realTime"
                      >
                        <a-card
                          class="package-small"
                          :body-style="{ padding: '2px' }"
                          :style="`background: ${$packageColor(item.flowId)};`"
                        >
                          <!-- <p :style="`font-weight: bold;`">
                            <codepen-circle-filled />
                          </p> -->
                          <p :style="'float: left; font-weight: bold'">#{{ item.finishTag }}</p>
                          <p :style="'float: right; font-weight: bold'">#{{ item.startTag }}</p>
                          <p>{{ item.size }}</p>
                        </a-card>
                      </a-list-item>
                    </template>
                  </a-list>
                </div>
              </div>
            </a-list-item>
          </template>
        </a-list>
      </a-col>
      <a-col :span="10" style="border-left: 1px dashed #ddd">
        <div 
          style="width: 100%"
        >
          <p
            style="
              font-weight: bold;
              font-size: 16px;
              line-height: 30px;
              text-align: right;
             "
             v-if="realTime >= 0"
          >
            {{ `#Real Time: ${realTime}` }}
          </p>
        </div>
        <a-list
          :grid="{ gutter: 16, xs: 1, sm: 2, lg: 4 }"
          :locale="{ emptyText: 'No more packages' }"
          :data-source="departedPackets"
        >
          <template #renderItem="{ item }">
            <a-list-item>
              <a-card
                class="package"
                :body-style="{ padding: '10px' }"
                :style="`background: ${$packageColor(item.flowId)};`"
              >
                <!-- <p :style="'font-weight: bold; text-align: left'">
                    <codepen-circle-filled/>
                    {{ index + 1 }}
                </p> -->
                <p :style="'float: left; font-weight: bold'">#S{{ item.startTag }}</p>
                <p :style="'float: right; font-weight: bold'">#F{{ item.finishTag }}</p>
                <br><p>Size:{{ item.size }}</p>
              </a-card>
            </a-list-item>
          </template>
        </a-list>
      </a-col>
    </a-row>
    <a-row>
      <v-chart class="chart" :option="option" v-if="plots.length > 0"></v-chart>
    </a-row>
  </div>
</template>


<script>
//import {FastForwardOutlined} from "@ant-design/icons-vue";
import ECharts from 'vue-echarts';
import {use} from "echarts/core";
import {CanvasRenderer} from 'echarts/renderers';
import {LineChart} from 'echarts/charts';
import {GridComponent, TooltipComponent} from 'echarts/components';

use([
  CanvasRenderer,
  LineChart,
  GridComponent,
  TooltipComponent
]);


export default {
  name: "AnimationCard",
  props: {
  
  },
  components: {
    'v-chart': ECharts
  },
  computed: {
    
  },
  data() {
    return {
      plots: [],
      option: null,
      
      test: [],
    
      realTime: -1,
      credit: 0,
      weightSum: 0.0,
      
      departedPackets: [],
      rawPackets: [],
      queueAmount: 3,
      queueList: [],
      timer: null,
      timeInterval: 1,
    };
  },
  methods: {
    sortFinTag(a, b) {
      return a.finishTag - b.finishTag;
    },
    sortStartTag(a,b) {
      return a.startTag - b.startTag;
    },
    sortArrTime(a, b) {
      return a.arrTime - b.arrTime;
    },
    
    popPacketByFin() { 
      //keep track of real time
      this.realTime++;
      if(this.credit < 0) this.credit++;//how much bits of current packet are unserved
      
      
      //sort by finish tag
      for (let i = 0; i <= this.queueList.length - 1; i++){
        this.queueList[i].list.sort(this.sortFinTag);
      }
      
      
      //choose a packet with the smallest finish tag to serve
      let item = this.queueList[0].list[0];
      for (let i = 0; i <= this.queueList.length - 1; i++){
        let tmp = this.queueList[i].list[0];
        if(item == null || (tmp != null && tmp.finishTag < item.finishTag)){
          item = tmp;
        }
      }
      
      //stop serving
      if (item == null && this.credit == 0) {
        clearInterval(this.timer);
        return;
      }
      
      //serve the chosen packet when no other packet is serving
      if(this.credit == 0){
        item = this.queueList[item.flowId].list.shift();
        this.departedPackets.push(item);
        this.credit -= item.size;
      }
      
    },
    
    popPacketByStart() { 
      //keep track of real time
      this.realTime++;
      //how much bits of current packet are unserved
      if(this.credit < 0) this.credit++;
      
      
      //sort by start tag
      for (let i = 0; i <= this.queueList.length - 1; i++){
        this.queueList[i].list.sort(this.sortStartTag);
      }
      
      
      //choose a packet with the smallest finish tag to serve
      let item = this.queueList[0].list[0];
      for (let i = 0; i <= this.queueList.length - 1; i++){
        let tmp = this.queueList[i].list[0];
        if(item == null || tmp != null && tmp.startTag < item.startTag){
          item = tmp;
        }
      }
      
      //stop serving
      if (item == null && this.credit == 0) {
        clearInterval(this.timer);
        return;
      }
      
      //serve the chosen packet when no other packet is serving
      if(this.credit == 0){
        item = this.queueList[item.flowId].list.shift();
        this.departedPackets.push(item);
        this.credit -= item.size;
      }
      
    },
    
    initial() {
      if (this.timer) {
        clearInterval(this.timer);
      }
      this.realTime = -1;
      this.credit = 0;
      this.weightSum = 0.0;
      
      //for test
      this.text = [];
      
      this.queueList = [];
      this.departedPackets = [];
      
      //generate packets
      /*
      //example 1
      this.rawPackets = [{size: 3, flowId: 0, arrTime: 3, startTag: 0, finishTag: 0}, 
                         {size: 4, flowId: 1, arrTime: 0, startTag: 0, finishTag: 0},
                         {size: 1, flowId: 1, arrTime: 8, startTag: 0, finishTag: 0},
                         {size: 2, flowId: 2, arrTime: 0, startTag: 0, finishTag: 0},
                         {size: 3, flowId: 2, arrTime: 4, startTag: 0, finishTag: 0},
                        ];
      */                 
      //example 2
      this.rawPackets = [{size: 2, flowId: 0, arrTime: 2, startTag: 0, finishTag: 0},
                         {size: 4, flowId: 1, arrTime: 0, startTag: 0, finishTag: 0}, 
                         {size: 3, flowId: 2, arrTime: 3, startTag: 0, finishTag: 0}, 
                        ];
                        
      
      //sort by arrival time
      this.rawPackets.sort(this.sortArrTime);
      
      //initial flow queues
      for (let i = 0; i < this.queueAmount; i++) {
        this.queueList.push({preFin: 0, list: [], weight: 0});
      }
      this.queueList[0].weight = 1;
      this.queueList[1].weight = 2;
      this.queueList[2].weight = 3;
      
      //calculate the sum of weights of all active flows
      for (let i = 0; i < this.queueAmount; i++){
        this.weightSum += this.queueList[i].weight;
      }
      
      //clear up plot data
      this.plots = [];
      this.option = {
        xAxis: {
          type: 'value',
          name: 'real time',
          min: 0,
          minInterval: 1,
        },
        yAxis: {
          type: 'value',
          name: 'virtual round',
          min: 0,
        },
        series: [{
          data: this.plots,
          type: 'line',
          smooth: false,
        }],
      };
    },
    
    applyWFQ() {
      this.initial();
      this.tagOnWFQ();
      //sort tagged packets by arrival time
      this.rawPackets.sort(this.sortArrTime);
      this.timer = setInterval(this.onWFQ, this.timeInterval * 1000);
    },
    applySFQ() {
      this.initial();
      this.timer = setInterval(this.onSFQ, this.timeInterval * 1000);
    },
    applySCFQ() {
      this.initial();
      this.timer = setInterval(this.onSCFQ, this.timeInterval * 1000);
    },
    finishNow() {
      clearInterval(this.timer);
      while (this.credit < 0) {
        this.onSFQ()
      }
    },
    
    transmit() {
      
    },
    tagOnWFQ(){
      let remainingBits = [];
      let waitingQueue = [];
      let taggedPackets = [];
      for(let i = 0; i < this.queueList.length; i++){
        remainingBits.push(0.0);
        waitingQueue.push([]);
      }
      
      let curRealTime = 0.0;
      let lastUpdate = 0.0;
      let curVirtualTime = 0.0;
      
   
      for(;;){
        //plot virtual time
        let index = 0;
        for(; index < this.plots.length; index++){
          if(this.plots[index][0] == curRealTime) break;  
        }
        if(index == this.plots.length){
          this.plots.push([curRealTime, curVirtualTime]);
        }
        
        
        //slope before current event
        let slope = 0.0;
        for(let i = 0; i < remainingBits.length; i++){
          if(remainingBits[i] > 0){
            slope += this.queueList[i].weight;
          }
        }
        
        
        //update remaining bits
        let newSlope = slope;
        for(let i = 0; i < remainingBits.length; i++){
          if(parseFloat(remainingBits[i]).toFixed(10) > 0){
            remainingBits[i] -= (curRealTime - lastUpdate)*(this.queueList[i].weight/slope);
          }else{
            continue;
          }
          //if the active flow has transmitted current packet, start to serve the waiting packets
          if(parseFloat(remainingBits[i]).toFixed(10) <= 0){
            if(waitingQueue[i].length > 0){
              remainingBits[i] = waitingQueue[i].shift().size;
            }else{
              newSlope -= this.queueList[i].weight;
            }
          }
        }
        slope = newSlope;
        
        //update start tag and finish tag
        for(; this.rawPackets.length > 0;){
          let item = this.rawPackets[0];
          if(item.arrTime > curRealTime) break;
          item = this.rawPackets.shift();
          item.startTag = Math.max(parseFloat(curVirtualTime).toFixed(1), this.queueList[item.flowId].preFin);
          item.finishTag = item.startTag + item.size/(this.queueList[item.flowId].weight/this.weightSum);
          this.queueList[item.flowId].preFin = item.finishTag;
          taggedPackets.push(item);
          //if the flow is inactive now in GPS, start serving the newly arriving packet
          if(remainingBits[item.flowId] > 0){
            waitingQueue[item.flowId].push(item);
          }else{
            remainingBits[item.flowId] = item.size;
            slope += this.queueList[item.flowId].weight;
          }
        }
        
        
        
        //trace the next departure event
        if(slope == 0 && this.rawPackets.length == 0) break;
        lastUpdate = curRealTime;
        let next = 0;
        for(let i = 0; i < remainingBits.length; i++){
          let tmp = remainingBits[i]/(this.queueList[i].weight/slope);
          if(parseFloat(tmp).toFixed(10) <= 0) continue;
          if(next == 0 || tmp < next) next = tmp;
        }
        if(parseFloat(next).toFixed(10) <= 0) break;
        curRealTime = curRealTime + next;
        //trace the next arrival event
        if(this.rawPackets.length > 0 && this.rawPackets[0].arrTime < curRealTime) 
          curRealTime = this.rawPackets[0].arrTime;
        curVirtualTime = curVirtualTime + (curRealTime - lastUpdate)/(slope/this.weightSum);   
        
      }
      
      
      //tag packets without tag
      this.rawPackets = taggedPackets; 
    },
    
    onWFQ(){
      for(; this.rawPackets.length > 0;){
        let item = this.rawPackets[0];
        if(item == null || item.arrTime > this.realTime + 1) break;
        item = this.rawPackets.shift();
        this.queueList[item.flowId].list.push(item);
      }
      
      this.popPacketByFin();
    },
    onSFQ(){
      for(; this.rawPackets.length > 0;){
        let item = this.rawPackets[0];
        if(item == null || item.arrTime > this.realTime + 1) break;
        item = this.rawPackets.shift();
        if(this.departedPackets.length > 0){
          let servingPkt = this.departedPackets[this.departedPackets.length - 1];
          item.startTag = servingPkt.startTag;
        }
        item.startTag = Math.max(item.startTag, this.queueList[item.flowId].preFin);
        item.finishTag = item.startTag + item.size/(this.queueList[item.flowId].weight/this.weightSum);
        this.queueList[item.flowId].list.push(item);
        this.queueList[item.flowId].preFin = item.finishTag;
      }
      
      this.popPacketByStart();
    },
    onSCFQ(){
      for(; this.rawPackets.length > 0;){
        let item = this.rawPackets[0];
        if(item == null || item.arrTime > this.realTime + 1) break;
        item = this.rawPackets.shift();
        if(this.departedPackets.length > 0){
          let servingPkt = this.departedPackets[this.departedPackets.length - 1];
          item.startTag = servingPkt.startTag;
        }
        item.startTag = Math.max(item.startTag, this.queueList[item.flowId].preFin);
        item.finishTag = item.startTag + item.size/(this.queueList[item.flowId].weight/this.weightSum);
        this.queueList[item.flowId].list.push(item);
        this.queueList[item.flowId].preFin = item.finishTag;
      }
      
      this.popPacketByFin();
    },
    
  },
  beforeUnmount() {
    clearInterval(this.timer);
  },
};
</script>

<style scoped>
.chart {
  width: calc(100% - 150px);
  height: 500px;
}
.package {
  color: #fff;
  border-radius: 10px;
}
.package >>> p {
  font-size: 16px;
  font-weight: 500;
  margin: 4px 0;
  text-align: center;
}
.package >>> .anticon {
  font-size: 22px;
}
.package-small {
  color: #fff;
  border-radius: 5px;
}
.package-small >>> p {
  font-size: 12px;
  font-weight: 500;
  margin: 4px 0;
  text-align: center;
}
.bound-text {
  width: 30px;
  height: 30px;
  border-radius: 15px;
  text-align: center;
  line-height: 30px;
  font-size: 18px;
  color: #fff;
}
.queue >>> .ant-row {
  flex-direction: row-reverse;
}
</style>
