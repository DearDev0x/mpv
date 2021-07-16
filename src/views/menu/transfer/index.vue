<template>
  <div class="transfer-page">
    <layout
      :header="tokenByName.name"
      @onBack="$router.push('/')"
      :class="{ blur: showPin }"
      :receive="true"
      @receive="dialog = true"
    >
      <template v-slot:content>
        <v-col cols="12" xs="12" sm="8" md="6" lg="4">
          <v-row no-gutters class="pb-5">
            <v-col cols="12">
              <v-card class="box-detail" elevation="0">
                <v-toolbar color="#c71e2b" dark>
                  <div>
                    Balance :
                    {{ parseUtillETH(tokenByName.balance) }}
                  </div>
                  <v-spacer></v-spacer>
                  <div>
                    <v-img :src="tokenByName.logoURI" height="30px" width="30px" contain></v-img>
                  </div>
                </v-toolbar>
                <div class="pa-4 mt-4">
                  <v-form
                    ref="form"
                    v-model="valid"
                    lazy-validation
                    class="text-right"
                  >
                    <v-text-field
                      outlined
                      background-color="white"
                      v-model="form.to"
                      :rules="itemRules"
                      label="หมายเลขกระเป๋า"
                      required
                    >
                    </v-text-field>

                    <v-text-field
                      type="number"
                      outlined
                      background-color="white"
                      v-model="form.amount"
                      :rules="[
                        (v) => !!v || 'กรุณากรอกข้อมูล',
                        (v) => parseFloat(v) > 0 || 'จำนวนต้องมากกว่า 0',
                        (v) =>
                          parseFloat(v) <= parseFloat(tokenByName.balance) ||
                          'จำนวนไม่ถูกต้อง',
                      ]"
                      label="จำนวน"
                      required
                    >
                    </v-text-field>
                    <v-btn text class="mr-2" @click="reset"> เคลียร์ </v-btn>
                    <v-btn
                      :disabled="!valid"
                      color="#c71e2b"
                      class="ml-2"
                      @click="validate"
                    >
                      <span class="white--text"> โอนเหรียญ </span>
                    </v-btn>
                  </v-form>
                </div>
              </v-card>
            </v-col>
            <v-col cols="12" class="mt-30" v-show="showPage">
              <div class="d-flex align-center">
                <v-icon large> history </v-icon>
                <span class="ml-1">ประวัติการทำทำรายการ</span>
              </div>
              <v-card class="box-detail mt-2" elevation="0">
                <div v-if="historyByToken.length > 0">
                  <history-item
                    v-for="(history, i) in historyByToken"
                    :key="'historyByToken' + i"
                    :hash="history.hash"
                    :from="history.from"
                    :timeStamp="history.timeStamp"
                    :tokenSymbol="history.tokenSymbol"
                    :value="history.value"
                    :lastChild="i >= historyByToken.length - 1"
                  />
                </div>
                <div
                  v-else
                  class="d-flex align-center justify-center"
                  style="height: 100px"
                >
                  ไม่มีข้อมูล...
                </div>
              </v-card>
            </v-col>
          </v-row>
        </v-col>
      </template>
    </layout>
    <pin-pad
      v-if="showPin"
      header="กรุณาใส่รหัส"
      :verify="true"
      :backward="true"
      @goback="showPin = false"
      @verifyPin="sendToken"
    />
    <v-dialog max-width="350" v-model="dialog">
      <my-qrcode
        :value="
          JSON.stringify({
            type: 'transfer',
            to: ethereumAddress,
            contractAddress: tokenByName.address,
          })
        "
      />
    </v-dialog>
  </div>
</template>

<script>
import layout from "../../../components/layout.vue";
import pidPad from "../../../components/pinPad.vue";
import qrcode from "../../../components/qrcode.vue";
import historyItem from "../../../components/historyItem.vue";
export default {
  name: "Token",
  components: {
    layout: layout,
    "pin-pad": pidPad,
    "my-qrcode": qrcode,
    "history-item": historyItem,
  },
  data() {
    return {
      dialog: false,
      showPin: false,
      showPage: false,
      valid: true,
      itemRules: [
        (v) => !!v || "กรุณากรอกข้อมูล",
        (v) => /^0x[a-fA-F0-9]{40}$/g.test(v) || "กรุณากรอกข้อมูลให้ถูกต้อง",
      ],
      form: {
        to: null,
        amount: null,
        text: null,
      },
      transferInstance: null,
    };
  },
  computed: {
    tokenByName() {
      return this.$store.getters.tokenByName(this.$route.params.token);
    },
    historyByToken() {
      return this.$store.getters.historyByToken(this.tokenByName.address);
    },
  },
  methods: {
    async sendToken(verify) {
      if (verify) {
        this.showPin = false;
        try {
          const provider = await new this.$ethers.providers.JsonRpcProvider(
            "https://rpc.tbwg.io"
          );
          const walletSigner = await new this.$ethers.Wallet(
            this.$store.state.auth.me.privateKey,
            provider
          );
          let amount = await this.$ethers.utils.parseUnits(
            this.form.amount,
            18
          );
          let to_address = await this.form.to;
          let gas_limit = "0x100000";
          await this.app_loading(true);
          let currentGasPrice = await provider.getGasPrice();
          let gas_price = await this.$ethers.utils.hexlify(
            parseInt(currentGasPrice)
          );
          if (this.tokenByName.address != "mainnet") {
            try {
              // general token send
              let contract = await new this.$ethers.Contract(
                this.tokenByName.address,
                this.$abi,
                walletSigner
              );
              let transaction = await contract.transfer(to_address, amount);
              await transaction.wait();
              await this.transferSuccess();
            } catch (err) {
              throw err;
            }
          } else {
            // ether send
            try {
              const tx = {
                from: this.ethereumAddress,
                to: to_address,
                value: amount,
                nonce: provider.getTransactionCount(
                  this.ethereumAddress,
                  "latest"
                ),
                gasLimit: this.$ethers.utils.hexlify(gas_limit), // 100000
                gasPrice: gas_price,
              };
              let transaction = await walletSigner.sendTransaction(tx);
              await transaction.wait();
              await this.transferSuccess();
            } catch (err) {
              throw err;
            }
          }
        } catch (err) {
          console.log(err);
          this.transferError();
        }
      }
    },
    async validate() {
      let valid = await this.$refs.form.validate();
      if (valid) {
        this.showPin = true;
        // await this.sendToken();
      }
    },
    async transferSuccess() {
      await this.$store.dispatch("getHistory");
      await this.$store.dispatch("getBalance");
      await this.app_loading(false);
      this.reset();
      await this.alert_show({
        type: "success",
        header: "สำเร็จ",
        title: "โอนสำเร็จ",
      });
    },
    async transferError() {
      this.app_loading(false);
      this.alert_show({
        type: "error",
        title: "การทำรายการล้มเหลว",
        header: "ล้มเหลว !",
      });
    },
    reset() {
      this.$refs.form.reset();
      this.$refs.form.resetValidation();
    },
  },
  async mounted() {
    if (this.$route.query.to != null) {
      this.form.to = this.$route.query.to;
    }
  },
  async created() {
    this.app_loading(true);
    await this.$store.dispatch("getHistory");
    this.app_loading(false);
    this.showPage = true;
  },
};
</script>

<style lang="scss">
.transfer-page {
  .box-detail {
    background-color: #f7f7f7;
  }
}
</style>