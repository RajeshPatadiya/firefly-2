// 资产详情
<template>
  <div class="page">
    <toolbar :title="$t(title)" 
      :showmenuicon="showmenuicon" 
      :showbackicon="showbackicon"
      @goback="back"
      :shadow=false  
      ref="toolbar"
      >
      <span slot="switch_password">{{$t('Account.Password')}}</span>
    </toolbar>

    <swiper :options="swiperOptionTop" class="gallery-top assets-wrapper" ref="swiperTop">
      <swiper-slide 
        v-for="(item,index) in this.balances"
        v-bind:item="item"
        v-bind:index="index"
        v-bind:key="item.code +'_'+item.issuer"
      >
        <div class="assets-code">{{item.code}}</div>
        <div class="assets-issuer" v-if="assethosts[item.code]">{{assethosts[item.code]}}</div>
        <div class="assets-issuer" v-else-if="assethosts[item.issuer]">{{assethosts[item.issuer]}}</div>
        <div class="assets-issuer" v-else>{{item.issuer|miniaddress}}</div>
      </swiper-slide>
    </swiper>

    <div class="content">
      <swiper :options="swiperOptionContent" class="gallery-content" ref="swiperContent">
        <swiper-slide 
          v-for="(item,index) in this.balances"
          v-bind:item="item"
          v-bind:index="index"
          v-bind:key="item.code +'_'+item.issuer"
          class="infocard"
        >
          <v-layout row wrap class="asset-amount">
            <v-flex xs12 class="row" d-flex justify-center align-center>
                <v-flex xs2 class="label">{{$t('Total')}}</v-flex>
                <v-flex class="amount">{{item.balance}}</v-flex>
            </v-flex>
            <v-flex d-flex justify-center align-center xs12 class="row" v-if="item.code=='XLM'"> 
                <v-flex xs2 class="label">{{$t('Available')}}</v-flex>
                <v-flex class="available">{{item.balance - reserve}}</v-flex  >
                <v-spacer></v-spacer>
                <v-flex xs2 class="label-reserve">{{$t('Reserve')}}</v-flex>
                <v-flex xs1 class="reserve">{{reserve}}</v-flex>
            </v-flex>
          </v-layout>
            <v-flex d-flex xs12 class="row btns">
                <v-spacer></v-spacer>
                <v-btn   class="primary btn" @click.stop="receive">{{$t('Receive')}}</v-btn>
                <v-spacer></v-spacer>
                <v-btn   class="error btn" @click.stop="send">{{$t('Send')}}</v-btn>
                <v-spacer></v-spacer>
              </v-flex> 
        </swiper-slide>
      </swiper>


      <h4 class="subtitle">{{$t('History')}}</h4>
       <card padding="10px 10px" class="infocard">
        <div class="history" slot="card-content">
          <v-layout class="history-li" row wrap v-for="item in history" :key="item.id" @click.stop="toTranscation(item)">
              <v-flex xs4 class="history-wrapper">
                <div class="history">
                  <div class="history-flag">{{$t(item.flag)}}</div>
                  <div class="history-time">{{item.date}}</div>
                </div>
              </v-flex>
              <v-flex xs8 class="history-wrapper">
                <div :class="'history-amount' + (item.isInbound ? ' add ':' minus ') ">
                    <span class="inbound" v-if="item.isInbound">+</span>
                    <span class="inbound" v-else>-</span>
                    <span class="amount">{{item.amount}}</span>
                    <span class="code">{{item.asset.code}}</span>
                </div>
              </v-flex>
            </v-layout>
        </div>
      </card>
    </div>

  </div>
</template>

<script>
import Card from '../components/Card'
import Toolbar from '../components/Toolbar'
import { mapState, mapActions, mapGetters} from 'vuex'

import { swiper, swiperSlide } from 'vue-awesome-swiper'
import Loading from '@/components/Loading'

export default {
  data(){
    return {
      title: 'Title.MyAssets',
      showmenuicon: false,
      showbackicon: true,

      swiperOptionContent: {
          notNextTick: true,
          spaceBetween: 10
      },
      swiperOptionTop: {
          notNextTick: true,
          spaceBetween: 10,
          centeredSlides: true,
          slidesPerView: 'auto',
          touchRatio: 0.2,
          slideToClickedSlide: true
      }
    }
  },
   computed:{
    ...mapState({
      account: state => state.accounts.selectedAccount,
      accountData: state => state.accounts.accountData,
      selectedAsset: state => state.asset.selected,
      assethosts: state => state.asset.assethosts,
    }),
    ...mapGetters([
      'balances',
      'paymentsRecords',
      'reserve',
    ]),
    swiperTop() {
      return this.$refs.swiperTop.swiper
    },
    swiperContent(){
      return this.$refs.swiperContent.swiper
    },
    swiperIndex(){
      for (let basset of this.balances){
        if (basset.code == this.selectedAsset.code 
            && basset.isssuer == this.selectedAsset.isssuer){
          return this.balances.indexOf(basset)
        }
      }
      return 0
    },

    history(){
      console.log('--------payments records')
      console.log(this.paymentsRecords)
      let data = []
      if(!this.paymentsRecords){
        return data;
      }
      this.paymentsRecords.forEach((ele) => {
        let asset = ele.asset
        let sasset = this.selectedAsset.code ? this.selectedAsset : this.balances[this.swiperIndex]
        if(('XLM' === asset.code && 'XLM' === sasset.code)||
          (asset.code === sasset.code && asset.issuer === sasset.issuer)){
            if(ele.type==='payment' || ele.type==='path_payment'){
              if(ele.isInbound){
                ele.flag = 'Receive'
              }else{
                ele.flag = 'Send'
              }
            }else{
              ele.flag = ele.type
            }
            data.push(ele)
        }
       
      });
      return data
    },
  
  },
  updated(){
    this.$nextTick(()=>{
      if(this.paymentsRecords === null || this.paymentsRecords.length ===0 ){
        this.getPayments(this.account.address)
          .then(data=>{        })
          .catch(err=>{
            //this.$toasted.error(this.$t(''))
          })
      }
    })
  },
  mounted(){
    this.swiperTop.params.control = this.swiperContent
    this.swiperContent.params.control = this.swiperTop
    this.swiperTop.on('SlideChangeEnd', this.swipeAsset)
    this.swiperTop.slideTo(this.swiperIndex,0,true)
    //如果历史记录为空,则查询历史记录
    
  },
  methods: {
    ...mapActions({
      selectAsset:'selectAsset',
      delTrust:'delTrust',
      selectPayment: 'selectPayment',
      getPayments: 'getPayments',
    }),
    swipeAsset () {
      this.selectAsset(this.balances[this.swiperTop.activeIndex])
    },
    back(){
      this.$router.push({name:"MyAssets"})
      // this.$router.push(`/myassets`)
    },
    switchAsset(item){
      this.selectAsset(item)
    },
    // 发送资产
    send(){
      if(!this.accountData.seed){
        this.$toasted.error(this.$t('Error.NoPassword'))
        this.$refs.toolbar.showPasswordLogin()
        return
      }
      this.$router.push(`/sendasset`)
    },
    // 接收资产
    receive(){
      this.$router.push(`/receiveasset`)
    },
    toTranscation(item){
      this.selectPayment(item)
      this.$router.push(`/transaction`)
    },
   
  },
  components: {
    Card,
    Toolbar,
    swiper,
    swiperSlide,
    Loading
    
  }


}
</script>

<style lang="stylus" scoped>
@require '../stylus/color.styl'
.page
  background: $primarycolor.gray
  color: $primarycolor.font
  .content
    padding: 8px 8px

// .mytoolbar
//   background: $primarycolor.green
//   color: $primarycolor.font
//   width: 100%
//   height:56px
//   line-height: 56px
//   .back-icon
//     float: left
//     padding-left: 10px
//     .material-icons
//       font-size: 30px
//       margin-top: 10px
//   .mytitle
//     width: 100%
//     height:56px
//     line-height: 56px
//     text-align: center
//     font-size: 20px

// .gallery-top
    
.assets-wrapper
  background: $primarycolor.green
  height: 42px
  margin: 0 !important
  min-height:42px !important
  .swiper-slide 
    width: 100px;
    height: 100%;
    opacity: 0.6;
  .swiper-slide-active 
    opacity: 1;
    font-weight 600

.assets-code
  font-size: 16px
  text-align: center
.assets-issuer
  text-align center

.gallery-content 
   width: 100%;

.infocard
   min-height 10em
   max-width: 100%

.asset-amount
  padding: 10px
  min-height 10vh
  .row
    .label
    .label-reserve
      color: $secondarycolor.font
    .amount
      font-size: 20px
      padding-left: 10px
      padding-right: 20px
    .available
      font-size: 18px
      padding-left: 10px
      padding-right: 20px
    .reserve
      font-size: 18px
      padding-left: 10px
      padding-right: 20px
    .label-reserve
      padding-left: 10px
.btns
  padding-top: 20px
  padding-bottom: 10px
  .btn
    width 45%

.subtitle
  padding-left: 10px
  padding-top: 10px
  padding-bottom: 5px
  margin-bottom: 0px
  color: $secondarycolor.font
  font-size: 16px
.history-li
  padding-top: 5px
  padding-bottom: 5px
  font-size: 16px
.history-amount
  text-align: right
.history-amount.add
  color: $primarycolor.green
.history-amount.minus
  color: $primarycolor.red
</style>
