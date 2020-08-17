<template>
  <v-app>
    <v-content>
      <v-container fluid>
        <v-card class="mx-auto">
          <v-card-title>{{account?account.account:"Open MathWallet"}}</v-card-title>
          <v-card-actions>
            <v-btn class="primary" @click="login" v-if="!account">Log in</v-btn>
            <v-btn class="warning" @click="logout" v-else>Log out</v-btn>
          </v-card-actions>
        </v-card>

        <v-card light class="mt-2" v-if="account">
          <v-card-title>Transfer</v-card-title>
          <v-card-subtitle>
            <v-text-field
              label="Recipient"
              v-model="recipient"
              placeholder="band19r4ta37cnd6yxcpu9qsyf37qhgjmhgsde46vnw"
              filled
            ></v-text-field>
            <v-text-field label="Amount" v-model="amount" placeholder="0.1" filled></v-text-field>
          </v-card-subtitle>
          <v-card-actions>
            <v-btn class="success" @click="requestSignature">Send</v-btn>
          </v-card-actions>
        </v-card>
      </v-container>
    </v-content>
  </v-app>
</template>

<script>
export default {
  name: "App",
  data() {
    return {
      network: {
        blockchain: "bandchain",
        chainId: "band-wenchang-mainnet"
      },
      provider: null,
      account: null,
      recipient: "band19r4ta37cnd6yxcpu9qsyf37qhgjmhgsde46vnw",
      amount: ""
    };
  },
  methods: {
    login() {
      window.mathExtension.getIdentity(this.network).then(account => {
        this.account = account;
        this.provider = window.mathExtension.httpProvider(
          "https://api-wm-lb.bandchain.org"
        );
      });
    },
    logout() {
      window.mathExtension.forgetIdentity().then(() => {
        this.account = null;
      });
    },
    requestSignature() {
      this.provider
        .get(`/auth/accounts/${this.account.account}`)
        .then(response => {
          console.log("response => ", response);

          if (response.status == 200) {
            const chainAccount = response.result.result.value;
            const transaction = {
              account_number: chainAccount.account_number,
              chain_id: this.network.chainId,
              fee: {
                amount: [
                  {
                    amount: "30",
                    denom: "uband"
                  }
                ],
                gas: "200000"
              },
              memo: "",
              msgs: [
                {
                  type: "cosmos-sdk/MsgSend",
                  value: {
                    amount: [
                      {
                        amount: `${
                          this.amount.length
                            ? parseFloat(this.amount) * 1000000
                            : 0
                        }`,
                        denom: "uband"
                      }
                    ],
                    from_address: this.account.account,
                    to_address: "band19r4ta37cnd6yxcpu9qsyf37qhgjmhgsde46vnw"
                  }
                }
              ],
              sequence: chainAccount.sequence
            };
            // 请求插件签名
            window.mathExtension
              .requestSignature(transaction)
              .then(signature => {
                // Broadcast
                const broadcatTx = {
                  msg: transaction.msgs,
                  fee: transaction.fee,
                  memo: transaction.memo,
                  signatures: [signature]
                };
                const opts = {
                  data: { tx: broadcatTx, mode: "sync" },
                  headers: {
                    "Content-Type": "text/plain"
                  }
                };
                this.provider
                  .post("/txs", null, opts)
                  .then(response2 => {
                    console.log(response2);
                  })
                  .catch(err2 => {
                    console.log(err2);
                  });
              })
              .catch(e => {
                console.log(e);
              });
          }
        })
        .catch(error => {
          console.log(error);
        });
    }
  }
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
