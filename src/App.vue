<template>
  <v-app>
    <v-content>
      <v-container fluid>
        <v-card>
          <v-card-title>{{identity?identity.account:"Open MathWallet"}}</v-card-title>
          <v-card-actions>
            <v-btn class="primary" @click="login" v-if="!identity">Log in</v-btn>
            <v-btn class="warning" @click="logout" v-else>Log out</v-btn>
          </v-card-actions>
        </v-card>

        <v-card light class="mt-2" v-if="identity">
          <v-card-title>Transfer</v-card-title>
          <v-card-subtitle>
            <v-text-field
              label="Recipient"
              v-model="recipient"
              placeholder="gard18pgvd6dlhnt43l6t3ev0glyk4tflq2ld60tdy4"
              filled
            ></v-text-field>
            <v-text-field
              label="Amount"
              v-model="amount"
              placeholder="0.1"
              :hint="balance"
              persistent-hint
              filled
            ></v-text-field>
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
import { formatAmount, parseAmount } from "./utils/Format";
export default {
  name: "App",
  data() {
    return {
      rpcURL: "https://rest.hashgard.com/gard/api",
      provider: null,
      identity: null,
      balance: "",
      recipient: "gard18pgvd6dlhnt43l6t3ev0glyk4tflq2ld60tdy4",
      amount: "",
    };
  },
  methods: {
    login() {
      window.mathExtension
        .getIdentity({
          blockchain: "hashgard",
          chainId: "hashgard",
        })
        .then((identity) => {
          this.identity = identity;
          this.provider = window.mathExtension.httpProvider(this.rpcURL);
          this.getBalance();
        });
    },
    logout() {
      window.mathExtension.forgetIdentity().then(() => {
        this.identity = null;
      });
    },
    getBalance() {
      this.provider
        .get(`/bank/balances/${this.identity.account}`)
        .then((response) => {
          if (response.status == 200) {
            this.balance =
              response.result.result.length > 0
                ? formatAmount(response.result.result[0].amount, 6)
                : "";
          }
        });
    },
    async getAccountNumberAndSequence() {
      const res = await this.provider.get(
        `/auth/accounts/${this.identity.account}`
      );
      return {
        account_number: res.result.result.value.account_number,
        sequence: res.result.result.value.sequence,
      };
    },
    requestSignature() {
      this.getAccountNumberAndSequence().then(
        ({ account_number, sequence }) => {
          const transaction = {
            sequence: sequence,
            account_number: account_number,
            chain_id: "hashgard",
            fee: {
              amount: [
                {
                  amount: "1000",
                  denom: "ugard",
                },
              ],
              gas: "200000",
            },
            memo: "",
            msgs: [
              {
                type: "cosmos-sdk/MsgSend",
                value: {
                  amount: [
                    {
                      amount: parseAmount(this.amount, 6),
                      denom: "ugard",
                    },
                  ],
                  from_address: this.identity.account,
                  to_address: this.recipient,
                },
              },
            ],
          };
          // 请求插件签名
          window.mathExtension
            .requestSignature(transaction)
            .then((signature) => {
              // Broadcast
              const broadcatTx = {
                msg: transaction.msgs,
                fee: transaction.fee,
                memo: transaction.memo,
                signatures: [signature],
              };
              const opts = {
                data: { tx: broadcatTx, mode: "sync" },
                headers: {
                  "Content-Type": "text/plain",
                },
              };
              this.provider.post("/txs", null, opts).then((response2) => {
                console.log(response2);
              });
            });
        }
      );
    },
  },
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
