<template>
  <div>
    <div class="receiveContent_box">
      <div class="receiveAddress_box">
        <h3 v-html="addressTitle"></h3>
        <div class="receiveAddress_pwd">
          <input
            type="text"
            class="input-text input"
            readonly
            v-model="walletAddress"
            id="walletAdressHide"
          />
          <!-- <input type="text" v-model="walletAdress" id="walletAdressHide" class="inputOpacity"/> -->
        </div>
        <div class="receiveAddress_btn flex-c" id="receiveAddressBtn">
          <button class="btn blue flex-c" @click="qrcode(coinAddress)">
            <div class="icon">
              <img src="@/assets/image/QRcode.svg">
            </div>
            Show QR code
          </button>
          <button
            class="btn cyan flex-c"
            @click="copyAddress('walletAdressHide', 'receiveAddressBtn')"
            data-toggle="tooltip"
            data-placement="bottom"
            title="Copy clipboard"
          >
            <div class="icon"><img src="@/assets/image/copy.svg"></div>
            Copy clipboard
          </button>
        </div>
      </div>

      <div class="tableHistory_box">
        <hgroup class="tableHistory_title">
          <h3 class="title">History:</h3>
        </hgroup>
        <div class="tableHistory_table table-responsive">
          <table class="table table-bordered table-hover table-striped">
            <thead>
              <tr>
                <th>Status</th>
                <th>Coin</th>
                <th>Amount</th>
                <th>Date</th>
                <th>Information</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(item, index) in historyData" :key="index">
                <td>
                  <span v-html="item.status" :class="item.status !== 'Success' ? 'red' : ''"></span>
                </td>
                <td>
                  <span v-html="item.coin"></span>
                </td>
                <td>
                  <span v-html="item.value"></span>
                </td>
                <td>
                  <span v-html="item.date"></span>
                </td>
                <td>
                  <div class="moreInfo_box" @click="MoreContent">
                    <span v-html="item.txhax" :title="item.info" class="ellipsis moreInfo_hax"></span>
                    <ul class="list">
                      <li>TXid：{{item.txhax}}</li>
                      <li>Adress：{{item.from_address}}</li>
                    </ul>
                    <i class="arrow"></i>
                  </div>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div
        class="modal fade bs-example-modal-md"
        tabindex="-1"
        role="dialog"
        aria-labelledby="mySmallModalLabel"
        id="qrcodeBox"
      >
        <div class="modal-dialog modal-md modal-dialog-centered" role="document">
          <div class="modal-content">
            <div class="qrcodeCont_box">
              <div id="qrcode" class="flex-c"></div>
              <div class="qrcodeCont_title">
                <h3>Your Address</h3>
              </div>
            </div>
          </div>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import QRCode from "qrcodejs2"
export default {
  name: "receive",
  props: ["selectData"],
  data () {
    return {
      addressTitle: "",
      walletAddress: "",
      historyData: [],
      refreshHistory: null,
      coinAddress: "",
      web3: "",
      // count: 0
    }
  },
  watch: {
    selectData (cur, old) {
      if (cur.coin !== old.coin) {
        this.getInitData()
      }
    }
  },
  mounted () {
    this.walletAddress = this.$store.state.addressInfo
    this.pageRefresh()
    $(() => {
      $("[data-toggle='tooltip']").tooltip()
    })
    this.refreshHistory = setInterval(() => {
      this.getSendHistory()
    }, 30000)
    if (this.selectData.coin) {
      this.getInitData()
    }
  },
  methods: {
    getInitData () {
      this.titleChange(this.selectData.coin)
      this.coinAddress = this.selectData.address
      this.getSendHistory()
    },
    pageRefresh () {
      if (this.selectData.coin) {
        this.titleChange(this.selectData.coin)
      }
      if (location.href.indexOf("tranSend") === -1) {
        $(".transferBtn_btn").find("a:eq(0)").addClass("router-link-active")
      }
    },
    titleChange (bitType) {
      this.addressTitle = bitType + " Receiving Address"
    },
    qrcode (cont) {
      $("#qrcode").html("")
      let qrcodeInit = new QRCode("qrcode", {
        width: 300,
        height: 340, // 高度
        text: cont // 二维码内容
        // render: "canvas" // 设置渲染方式（有两种方式 table和canvas，默认是canvas）
        // background: "#f0f"
        // foreground: "#ff0"
      })
      $("#qrcodeBox").modal("show")
    },
    copyAddress (id, textId) {
      let copyText = $("#" + textId).find(".tooltip-inner").text()
      document.getElementById(id).select()
      document.execCommand("Copy")
      $("#" + textId)
        .find(".tooltip-inner")
        .text("Copied")
      setTimeout(function () {
        $("#" + textId)
          .find(".tooltip-inner")
          .text(copyText)
      }, 3000)
      this.$$.layerMsg("Copy Success")
    },
    setWeb3 () {
      this.$$.setWeb3(this)
    },
    MoreContent (e) {
      $(e.target.parentNode)
        .parents("tr")
        .siblings("tr")
        .find(".list")
        .hide()
      $(e.target.parentNode)
        .find(".list")
        .toggle()
    },
    getSendHistory () {
      if (!this.walletAddress || !this.selectData.coin) {
        return
      }
      $.ajax({
        url: this.$$.serverURL + "/transfer/receiveHistory",
        datatype: "json",
        type: "post",
        data: {
          to_address: this.walletAddress,
          coin: this.selectData.coin
        },
        success: res => {
          console.log(res)
          this.historyData = []
          if (res.msg === "success" && res.info.length > 0) {
            for (let i = 0; i < res.info.length; i++) {
              if (res.info[i].coin === this.selectData.coin && res.info[i].txhax) {
                res.info[i].date = this.$$.timeChange({date: res.info[i].date, type:"yyyy-mm-dd hh:mm"})
                res.info[i].value = this.$$.thousandBit(res.info[i].value, "no")
                res.info[i].status = this.getStatus(res.info[i])
                this.historyData.push(res.info[i])
              }
            }
          }
        },
        error: res => {
          console.log(res)
        }
      })
    },
    getStatus (txhax) {
      let statusFsn = ""
      if (txhax.txhax) {
        this.setWeb3()
        let receipt = this.web3.eth.getTransactionReceipt(txhax.txhax)
        if (receipt && receipt.status) {
          statusFsn = "Success"
        } else {
          statusFsn = "Pending"
        }
      } else {
        statusFsn = "Failure"
      }
      return statusFsn
    }
  },
  beforeDestroy () {
    clearInterval(this.refreshHistory)
    this.refreshHistory = null
  }
}
</script>
