<template>
  <div>
    <div class="pwdSure_box">

      <!-- <div class="newWallet_title flex-c">
        <h3 class="title">Import wallet</h3>
      </div> -->

      <div class="flex-bc selectType_box">
        <div class="selectType_type flex-ai-c">
          <ul>
            <li>
              <input
                type="radio"
                name="selectType"
                id="aria1"
                class="input"
                checked
              >
              <label class="flex-sc labelStyle" for="aria1" data-show="keystore">
                <div class="radioBox flex-c"><i class="radio"></i></div>
                <p class="p">Keystore / JSON File</p>
              </label>
            </li>
            <li>
              <input
                type="radio"
                name="selectType"
                id="aria2"
                class="input"
              >
              <label class="flex-sc labelStyle" for="aria2" data-show="private">
                <div class="radioBox flex-c"><i class="radio"></i></div>
                <p class="p">Private Key</p>
              </label>
            </li>
          </ul>
        </div>

        <div class="selectType_content">
          <div class="selectType_keystore" id="keystore" data-view="keystore" style="display:block">
            <hgroup class="selectType_contTitle">
              <h3 class="title">Select Your Wallet File</h3>
            </hgroup>
            <div class="selectType_contentBox">
              <div class="selectType_KSbtn">
                <p class="p" id="fileName">SELECT WALLET FILE...</p>
                <input
                  type="file"
                  class="file"
                  id="fileUpload"
                >
              </div>
              <div class="selectType_contTnput" v-if="showPwd">
                <input
                  type="password"
                  placeholder="Enter a password"
                  class="input-text input"
                  v-model="password"
                  @keyup="changePwd"
                >
              </div>

              <div class="createInfo_btn flex-c" v-if="showPwdBtn">
                <button class="btn" @click="inputFileBtn" id="UnlockBtn">Unlock</button>
              </div>
            </div>
          </div>

          <div class="selectType_keystore" id="private" data-view="private">
            <hgroup class="selectType_contTitle">
              <h3 class="title">Paste Your Private Key</h3>
            </hgroup>
            <div class="selectType_contentBox">
              <div class="selectType_contTnput">
                <input
                  type="password"
                  placeholder="Enter a Private Key"
                  class="input-text input pwdChange"
                  v-model="privateKey"
                  @keyup="changePrv"
                >
              </div>

              <div class="createInfo_btn flex-c">
                <button class="btn" @click="inputPwdBtn" v-if="showPwdBtn">Unlock</button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- <div class="createInfo_tip pb-20">
        <h3 class="h3">Fusion DCRM Wallet does not hold your keys for you. We cannot access accounts, recover keys, reset passwords,<br/>nor reverse transactions. Protect your keys & always check that you are on correct URL. You are responsible for your security.</h3>
      </div> -->

    </div>
  </div>
</template>

<style>

</style>

<script>
import wallet from "@/assets/js/wallet"
import Lilo from "@/assets/js/lilo"
export default {
  name: "createWallet",
  props: ["sendDataPage"],
  data () {
    return {
      password: "",
      fileJSON: "",
      showPwd: false,
      showPwdBtn: false,
      fileData: "",
      privateKey: "",
      checkAddress: "",
      web3: "",
      newWeb3: "",
      dcrmAddress: ""
    }
  },
  mounted () {
    // const that = this
    $(".selectType_type").on("click", "label", event => {
      let showID = $(event.currentTarget).attr("data-show")
      $("[data-view]").hide()
      $("[data-view=" + showID + "]").show()
      this.showPwd = false
      this.showPwdBtn = false
      this.password = ""
      $("#fileUpload").val("")
      this.privateKey = ""
      this.checkAddress = ""
      $("#fileName").text("SELECT WALLET FILE...")
    })
    $("#fileUpload").change(event => {
      let reader = new FileReader()
      this.password = ""
      // let _this = this
      let fileName = $(event.currentTarget)[0].files[0].name
      reader.onload = onLoadEvent => {
        this.fileData = onLoadEvent.currentTarget.result
        this.showPwd = this.walletRequirePass(this.fileData)
        if (this.showPwd) {
          $("#fileName").text(fileName)
        } else {
          $("#fileName").text("SELECT WALLET FILE...")
        }
      }
      reader.readAsText($(event.currentTarget)[0].files[0])
    })
    $(".pwdChange").on("change", () => {
      this.showPwdBtn = true
    })
  },
  methods: {
    setWeb3 () {
      this.$$.setWeb3(this)
      this.newWeb3 = new Lilo(this.$$.baseUrl)
    },
    inputFileBtn () {
      let walletData
      try{
        walletData = wallet.getWalletFromPrivKeyFile(this.fileData, this.password)
        this.privateKey = walletData.getPrivateKeyString()
        this.checkAddress = walletData.getChecksumAddressString()
        if (this.checkAddress.toLowerCase() !== this.sendDataPage.from.toLowerCase()) {
          this.$$.layerMsg({
            tip: "Account error!",
            time: 3000,
            bgColor: "#ea4b40",
            icon: this.$$.promptSvg
          })
          $("#sendInfo").modal("hide")
          return
        }
        if (this.sendDataPage.sendType === "LOCKOUT" || this.sendDataPage.sendType === "LOCKIN" || this.sendDataPage.sendType === "SENDDCRM") {
          this.getDcrmAddress()
        } else if (this.sendDataPage.sendType === "MYWALLET") {
          this.getDecrAddress(this.privateKey)
        } else {
          this.signSendData()
        }
      } catch (e) {
        this.$$.layerMsg({
          tip: e,
          time: 5000,
          bgColor: "#ea4b40",
          icon: this.$$.promptSvg
        })
      }
    },
    
    inputPwdBtn () {
      let walletData
      try {
        walletData = new wallet(
          new Buffer(this.fixPkey(this.privateKey), "hex")
        )
        this.privateKey = walletData.getPrivateKeyString()
        this.checkAddress = walletData.getChecksumAddressString()
        if (this.checkAddress.toLowerCase() !== this.sendDataPage.from.toLowerCase()) {
          this.$$.layerMsg({
            tip: "Account error!",
            time: 3000,
            bgColor: "#ea4b40",
            icon: this.$$.promptSvg
          })
          $("#sendInfo").modal("hide")
          return
        }
        if (this.sendDataPage.sendType === "LOCKOUT" || this.sendDataPage.sendType === "LOCKIN" || this.sendDataPage.sendType === "SENDDCRM") {
          this.getDcrmAddress()
        } else if (this.sendDataPage.sendType === "MYWALLET") {
          this.getDecrAddress(this.privateKey)
        }else {
          this.signSendData()
        }
      } catch (e) {
        console.log(e)
        this.$$.layerMsg({
          tip: e,
          time: 5000,
          bgColor: "#ea4b40",
          icon: this.$$.promptSvg
        })
      }
    },
    getDcrmAddress () {
      this.setWeb3()
      this.newWeb3.lilo.dcrmGetAddr(this.$store.state.addressInfo, this.sendDataPage.coin).then((val) => {
        this.dcrmAddress = val
        this.signSendData()
      })
    },
    getDecrAddress (pwd) {
      this.setWeb3()
      let gasPriceNum
      try {
        gasPriceNum = this.web3.eth.gasPrice.toString(10)
      } catch (error) {
        gasPriceNum = this.$$.getWeb3({
          method: "eth_gasPrice",
          params: []
        }).result.toString(10)
      }

      let getGasLimit
      try {
        getGasLimit = this.web3.eth.estimateGas({to: this.toAddress})
      } catch (error) {
        try {
          getGasLimit = this.$$.getWeb3({
            method: "eth_estimateGas",
            params: [{to: this.toAddress}]
          })
          if (getGasLimit.error) {
            getGasLimit = "Error"
          } else {
            getGasLimit = getGasLimit.result
          }
        } catch (error) {
          getGasLimit = error
        }
      }
      if (getGasLimit && getGasLimit.toString().indexOf("Error") !== -1) {
        this.$$.layerMsg({
          tip: getGasLimit,
          time: 5000,
          bgColor: "#ea4b40",
          icon: this.$$.promptSvg
        })
        throw getGasLimit
      }
      let rawTx = {
        nonce: this.sendDataPage.nonce,
        gasPrice: Number(gasPriceNum),
        gasLimit: Number(getGasLimit),
        // gasLimit: 21000,
        from: this.sendDataPage.from,
        to: "0x00000000000000000000000000000000000000dc",
        value: 0,
        data: ""
      }
      let sendBack = {
        coin: this.sendDataPage.coin,
        serializedTx: "",
        nowFlag: false,
        dcrmAddress: ""
      }
      let Tx = require("ethereumjs-tx")
      pwd = pwd.indexOf("0x") === 0 ? pwd.substr(2) : pwd
      let privateKey = new Buffer(pwd, "hex")
      this.newWeb3.lilo.dcrmReqAddr(this.sendDataPage.from, this.sendDataPage.coin, pwd).then((val) => {
        let dcrmAddressVal
        if ((typeof val).toLowerCase() === "object") {
          if (val.result && val.result.indexOf("the account has request dcrm address already.the dcrm address is") === 0) {
            dcrmAddressVal = val.result.split(":")[1]
            sendBack.nowFlag = false
          } else if (val.error && val.error.message.indexOf("the account has request dcrm address already.the dcrm address is") === 0) {
            dcrmAddressVal = val.error.message.split(":")[1]
            sendBack.nowFlag = false
          } else if (val.result && val.result.indexOf("the account has confirm dcrm address already.the dcrm address is") === 0) {
            this.$$.layerMsg({
              tip: "The address has been requested. Please refresh later. DCRM:" + val.result.split(":")[1],
              time: 8000,
              bgColor: "#5dba5a",
              icon: this.$$.promptSvg
            })
            $("#privateSure").modal("hide")
            return
          } else {
            this.$$.layerMsg({
              tip: val,
              time: 8000,
              bgColor: "#ea4b40",
              icon: this.$$.promptSvg
            })
            return
          }
        } else {
          dcrmAddressVal = val
          sendBack.nowFlag = true
        }
        sendBack.dcrmAddress = dcrmAddressVal
        rawTx.data = "DCRMCONFIRMADDR:" + dcrmAddressVal + ":" + this.sendDataPage.coin
        let tx = new Tx(rawTx)
        tx.sign(privateKey)
        let serializedTx = tx.serialize()
        let serializedTxString = serializedTx.toString("hex")
        serializedTxString = serializedTxString.indexOf("0x") === 0 ? serializedTxString : ("0x" + serializedTxString)
        sendBack.serializedTx = serializedTxString
        this.sendSignData(sendBack)
      })
    },
    signSendData () {
      if (this.checkAddress.toLowerCase() === this.sendDataPage.from.toLowerCase()) {
        let rawTx = {
          nonce: this.sendDataPage.nonce,
          gasPrice: Number(this.sendDataPage.gasPrice),//Number类型 
          gasLimit: Number(this.sendDataPage.gasLimit),
          from: this.sendDataPage.from,
          to: this.sendDataPage.to,
          value: Number(this.sendDataPage.value),//Number类型
          data: this.sendDataPage.data
        }
        let Tx = require("ethereumjs-tx")
        let privateKey = new Buffer(this.fixPkey(this.privateKey), "hex")
        let tx = new Tx(rawTx)
        tx.sign(privateKey)
        let serializedTx = tx.serialize()
        let serializedTxString = serializedTx.toString("hex")
        serializedTxString = serializedTxString.indexOf("0x") === 0 ? serializedTxString : ("0x" + serializedTxString)
        this.sendSignData(serializedTxString)
      } else {    
        this.$$.layerMsg({
          tip: "Account error!",
          time: 3000,
          bgColor: "#ea4b40",
          icon: this.$$.promptSvg
        })
        $("#sendInfo").modal("hide")
        return
      }
    },
    sendSignData (data) {
      this.$emit("sendSignData", data)
      this.password = ""
      $("#fileUpload").val("")
      this.privateKey = ""
      this.checkAddress = ""
      this.showPwdBtn = false
    },
    changePrv (e) {
      this.showPwdBtn = true
      if (e.which === 13) {
        this.inputPwdBtn()
      }
    },
    changePwd (e) {
      if (e.which === 13) {
        this.inputFileBtn()
      }
      this.showPwdBtn = true
    },
    walletRequirePass (ethjson) {
      let jsonArr
      try {
          jsonArr = JSON.parse(ethjson)
      } catch (err) {
          throw "This is not a valid wallet file. "
      }
      if (jsonArr.encseed != null) {
        return true
      } else if (jsonArr.Crypto != null || jsonArr.crypto != null) {
        return true
      } else if (jsonArr.hash != null && jsonArr.locked) {
        return true
      } else if (jsonArr.hash != null && !jsonArr.locked) {
        return false
      } else if (jsonArr.publisher == "MyEtherWallet" && !jsonArr.encrypted) {
        return false
      } else {
        throw "Sorry! We don\"t recognize this type of wallet file. "
      }
    },
    fixPkey (key) {
      if (key.indexOf("0x") === 0) {
        return key.slice(2);
      }
      return key
    }
  }
}
</script>
