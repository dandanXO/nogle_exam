<template>
  <div>
    <h2>TABLE</h2>
    <div class="quote-table">
      <div class="quote-header">
        <div class="quote-header-cell">Price</div>
        <div class="quote-header-cell">Size</div>
      </div>
      <div class="quote-colum_sell" v-if="sortedSellQuotes().length">
        <div v-for="(index) in 8" :key="sortedSellQuotes()[index]?.id" class="quote-row" :class="{ 'highlightSell': sortedSellQuotes()[index]?.isNew, 'buy-quote': sortedSellQuotes()[index]?.type === 'buy', 'sell-quote': sortedSellQuotes()[index]?.type === 'sell' }">
          <div class="quote-cell" :style="{ color: sortedSellQuotes()[index]?.textColor }">
            {{ formatNumber(sortedSellQuotes()[index]?.price) }}
          </div>
          <div class="quote-cell">{{ formatNumber(sortedSellQuotes()[index]?.size) }}</div>
        </div>
      </div>
      <!-- <div v-for="(quote) in sortedSellQuotes()" :key="quote.id" class="quote-row" :class="{ 'highlightSell': quote.isNew, 'buy-quote': quote.type === 'buy', 'sell-quote': quote.type === 'sell' }">
        <div class="quote-cell" :style="{ color: quote.textColor }">
          {{ formatNumber(quote.price) }}
        </div>
        <div class="quote-cell">{{ formatNumber(quote.size) }}</div>
      </div> -->
      <div class="real-price" :class="{'real-price_up': realPriceUp, 'real-price_down': !realPriceUp}">
        {{ formatNumber(realPrice) }} 
        <span v-if="realPriceUp" class="">&uarr;</span> 
        <span v-else class="">&darr;</span>
      </div>
      <div v-if="sortedSellQuotes().length">
        <div v-for="(index) in 8" :key="sortedQuotes()[index]?.id" class="quote-row" :class="{ 'highlightbuy': sortedQuotes()[index]?.isNew, 'buy-quote': sortedQuotes()[index]?.type === 'buy', 'sell-quote': sortedQuotes()[index]?.type === 'sell' }">
          <div class="quote-cell" :style="{ color: sortedQuotes()[index]?.textColor }">
            {{ formatNumber(sortedQuotes()[index]?.price) }}
          </div>
          <div class="quote-cell">{{ formatNumber(sortedQuotes()[index]?.size) }}</div>
        </div>
      </div>  
      <!-- <div v-for="(quote) in sortedQuotes()" :key="quote.id" class="quote-row" :class="{ 'highlightbuy': quote.isNew, 'buy-quote': quote.type === 'buy', 'sell-quote': quote.type === 'sell' }">
        <div class="quote-cell" :style="{ color: quote.textColor }">
          {{ formatNumber(quote.price) }}
        </div>
        <div class="quote-cell">{{ formatNumber(quote.size) }}</div>
      </div> -->
    </div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';

export default {
  setup() {
    const maxQuotesPerDirection = 20;
    const buyQuotes = ref([]);
    const sellQuotes = ref([]);
    const realPrice = ref([]);
    const realPriceUp = ref(true)

    const ws1 = ref();
    const tempForRealPrice = ref([]);
     // WebSocket URL
    const wsUrl = 'wss://ws.btse.com/ws/oss/futures';
    const wsUrl2 ='wss://ws.btse.com/ws/futures'
    const ws2 = new WebSocket(wsUrl2)
    ws2.onopen = (event)=>{
      ws2.send(
        JSON.stringify({
          "op": "subscribe",
          "args": [
            "tradeHistoryApi:BTCPFC"
          ]
        })
      )
    }
    ws2.onmessage = (event)=>{
      const data = JSON.parse(event.data);
      if(data.data){
        data.data.forEach(item=>{
        tempForRealPrice.value = realPrice.value
        realPrice.value = item.price
        if(item.side == 'BUY'){
          realPriceUp.value = true
        }else{
          realPriceUp.value = false
        }
        })
      }
      
     
    }
    const createWs = () => {
      ws1.value = new WebSocket(wsUrl)
      ws1.value.onopen = wsonOpen
      ws1.value.onclose = wsonClose
      ws1.value.onmessage = wsonMessage
      ws1.value.onerror = wsonError
    }
    createWs()
    function wsonOpen(){
      ws1.value.send(
        JSON.stringify({
          "op": "subscribe",
          "args": [
            "update:BTCPFC_0"
          ]
        })
      )
    }
    function wsonClose(event){
    }
    function wsonMessage (event){
      // 在這裡處理從 WebSocket 收到的數據
      const data = JSON.parse(event.data);
      if (data.topic === 'update:BTCPFC_0' && data.data.type === 'delta') {
         // 處理資料的邏輯，例如更新買單和賣單
         if (data.data.bids.length) {
          data.data.bids.forEach(([price, size]) => {
            // 查找是否存在相同價格的買單
            const existingBuyOrderIndex = buyQuotes.value.findIndex((quote) => quote.type === 'buy' && quote.price === price);

            if (existingBuyOrderIndex !== -1 && size != 0) {
              // 如果存在相同價格的買單，則更新現有買單的大小（size）和 id
              buyQuotes.value[existingBuyOrderIndex].size = size;
              buyQuotes.value[existingBuyOrderIndex].id = price;
              buyQuotes.value[existingBuyOrderIndex].isNew = false; // 不標記為新
              
            } else {
              buyQuotes.value = [...buyQuotes.value].sort((a, b) => b.price - a.price);
              if(size != 0 ){
                if (buyQuotes.value.length < maxQuotesPerDirection) {
                // 如果不存在相同價格的買單且尚未達到最大限制，則添加新的買單
                buyQuotes.value.push({ id: price, price, size, textColor: '#00b15d', bgColor: 'rgba(16, 186, 104, 0.12)', type: 'buy', isNew: true });
              } else {
                // 如果超過最大限制，則取代最大的
                const oldestBuyOrderIndex = buyQuotes.value.findIndex((quote) => quote.type === 'buy' && quote.id === Math.min(...buyQuotes.value.map((quote) => quote.id)));
                if(oldestBuyOrderIndex == -1){
                  buyQuotes.value.push({ id: price, price, size, textColor: '#00b15d', bgColor: 'rgba(16, 186, 104, 0.12)', type: 'buy', isNew: true });
                  buyQuotes.value = [...buyQuotes.value].sort((a, b) => b.price - a.price);
                  if (buyQuotes.value.length > maxQuotesPerDirection) {
                    buyQuotes.value.pop();
                  }
                }else{
                  buyQuotes.value.splice(oldestBuyOrderIndex, 1);
                  buyQuotes.value.push({ id: price, price, size, textColor: '#00b15d', bgColor: 'rgba(16, 186, 104, 0.12)', type: 'buy', isNew: true });
                }
                
              }
              }else{
                {
                const oldestBuyOrderIndex = buyQuotes.value.findIndex((quote)=>{
                  return quote.price == price
                })
                if(oldestBuyOrderIndex >=0){
                  buyQuotes.value.splice(oldestBuyOrderIndex, 1);
                }
              }
              }
            }
          });
        }
        
        if (data.data.asks.length) {
          data.data.asks.forEach(([price, size]) => {
            // 查找是否存在相同價格的賣單
            const existingSellOrderIndex = sellQuotes.value.findIndex((quote) => quote.type === 'sell' && quote.price === price);

            if (existingSellOrderIndex !== -1 && size != 0) {
              // 如果存在相同價格的買單，則更新現有買單的大小（size）和 id
              sellQuotes.value[existingSellOrderIndex].size = size;
              sellQuotes.value[existingSellOrderIndex].id = price;
              sellQuotes.value[existingSellOrderIndex].isNew = false; // 不標記為新
            } else {
              sellQuotes.value = [...sellQuotes.value].sort((a, b) => a.price - b.price);
              if(size != 0){
                if (sellQuotes.value.length < maxQuotesPerDirection) {
                  // 如果不存在相同價格的買單且尚未達到最大限制，則添加新的賣單
                  sellQuotes.value.push({ id: price, price, size, textColor: '#FF5B5A', bgColor: 'rrgba(255, 90, 90, 0.12)', type: 'sell', isNew: true });
                } else {
                  // 如果超過最大限制，則取代最小的
                  const oldestBuyOrderIndex = sellQuotes.value.findIndex((quote) => quote.type === 'sell' && quote.id === Math.min(...sellQuotes.value.map((quote) => quote.id)));
                  if(oldestBuyOrderIndex == -1){
                    sellQuotes.value.push({ id: price, price, size, textColor: '#FF5B5A', bgColor: 'rgba(255, 90, 90, 0.12)', type: 'sell', isNew: true });
                    sellQuotes.value = [...sellQuotes.value].sort((a, b) => a.price - b.price);
                    if (sellQuotes.value.length > maxQuotesPerDirection) {
                      sellQuotes.value.pop();
                    }
                  }else{
                    sellQuotes.value.splice(oldestBuyOrderIndex, 1);
                    sellQuotes.value.push({ id: price, price, size, textColor: '#FF5B5A', bgColor: 'rgba(255, 90, 90, 0.12)', type: 'sell', isNew: true });
                  }
                }
              }else{
                const oldestBuyOrderIndex = sellQuotes.value.findIndex((quote)=>{
                  return quote.price == price
                })
                if(oldestBuyOrderIndex >=0){
                  sellQuotes.value.splice(oldestBuyOrderIndex, 1);
                }
              }
            }
          });
        }
      }

      // 檢查是否重新訂閱
      if(sellQuotes.value[sellQuotes.value.length-1].price < buyQuotes.value[0].price) {
        console.log("需要重新訂閱")
        ws1.value.close()
        sellQuotes.value = []
        buyQuotes.value = []
        createWs()
      }
    }
    function wsonError (){

    }

    
    const formatNumber = (number) => {
      if(number){
        return number.toLocaleString();
      }else{
        return 0;
      }
      
    };
    const sortedSellQuotes = (()=>{
      return [...sellQuotes.value].sort((a, b) => a.price - b.price)
    })
    // 計算按價格排序的報價
    const sortedQuotes = (() => {
      return [...buyQuotes.value].sort((a, b) => b.price - a.price)
    });




    onMounted(() => {

    });

    return {
      buyQuotes,
      formatNumber,
      sortedQuotes,
      sortedSellQuotes,
      realPrice,
      realPriceUp
    };
  },
};
</script>

<style>
.buy-quote {
  color: #00b15d;
}

.sell-quote {
  color: #FF5B5A;
}
.quote-table {
  width: 100%;
  background-color: #131B29;
  border: 1px solid #ccc;
  color: #F0F4F8;
}

.quote-header {
  display: flex;
  background-color: #131B29;
  font-weight: bold;
}

.quote-header-cell {
  flex: 1;
  padding: 8px;
  text-align: center;
}
.quote-colum_sell{
  display: flex;
  flex-direction: column-reverse;
}
.quote-row {
  display: flex;
  transition: background-color 0.3s;
  cursor: pointer;
}

.quote-cell {
  flex: 1;
  padding: 8px;
  text-align: center;
}

.highlightbuy{
  animation: highlightbuy 1s ease-in-out;
}
.highlightSell{
  animation: highlightSell 1s ease-in-out;
}
@keyframes highlightSell {
  0% {
    background-color: transparent;
  }
  50% {
    background-color: red; /* Change to red for sell quotes */
  }
  100% {
    background-color: transparent;
  }
  
}
@keyframes highlightbuy {
  0% {
    background-color: transparent;
  }
  50% {
    background-color: green; /* Change to red for sell quotes */
  }
  100% {
    background-color: transparent;
  }
}

.real-price{
  display: flex;
  justify-content: center;
  padding: 1rem 0;
}
.real-price_up{
  color: #00b15d;
  background-color: rgba(16, 186, 104, 0.12)
}
.real-price_down{
  color: #FF5B5A;
  background-color: rgba(255, 90, 90, 0.12);
}
</style>