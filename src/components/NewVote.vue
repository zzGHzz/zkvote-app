<template>
  <div class="border rounded d-flex m-2 p-2 flex-column">
    <p class="text-left font-weight-bold m-2">Create a new vote here: </p>
    <b-button class="m-2" variant="primary" v-on:click="newVote()">New Vote</b-button>
    <b-container class="m-2" fluid>
      <b-row>
        <b-col sm="3">
          Authority Account
        </b-col>
        <b-col sm="2">
          TxID
        </b-col>
        <b-col sm="1">
          Status
        </b-col>
        <b-col sm="5">
          Vote ID
        </b-col>
      </b-row>
      <b-row v-for="vote in votes" :key="vote.txid">
        <b-col sm="3">
          {{ vote.authAddr }}
        </b-col>
        <b-col sm="2">
          {{ shortHex(vote.txID) }}
        </b-col>
        <b-col sm="1">
          {{ vote.status }}
        </b-col>
        <b-col sm="5">
          {{ vote.voteID }}
        </b-col>
      </b-row>
    </b-container>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from "vue-property-decorator";

import { getABI } from "myvetools/dist/utils";
import { encodeABI } from "myvetools/dist/connexUtils";
import { abiVoteCreator, addrVoteCreator, shortHex } from "../common";

@Component
export default class NewVote extends Vue {
  private auths: {
    address: string;
    unconfirmed_txids: string[];
  }[] = [];

  private votes: {
    authAddr: string;
    voteID: string;
    txID: string;
    status: "success" | "reverted";
  }[] = [];

  private async newVote() {
    const signingService = connex.vendor.sign("tx");
    signingService.gas(5000000);
    const abi = getABI(abiVoteCreator, "newBinaryVote", "function");
    const data = encodeABI(abi);
    const resp = await signingService.request([
      {
        to: addrVoteCreator,
        value: "0x0",
        data: data,
      },
    ]);

    let p = this.auths.findIndex((auth) => auth.address === resp.signer);
    if (p == -1) {
      this.auths.push({
        address: resp.signer,
        unconfirmed_txids: [],
      });
    }

    p = this.auths.findIndex((auth) => auth.address === resp.signer);
    this.auths[p].unconfirmed_txids.push(resp.txid);
  }

  public async created() {
    const ticker = connex.thor.ticker();
    for (;;) {
      // Loop that checks all the unconfirmed txs
      // If reverted, remove txid from unconfirmed_txids
      // If success, 
      // 1. remove txid from auth.unconfirmed_txids, and
      // 2. add a new vote to votes
      for (let i in this.auths) {
        for (let txid of this.auths[i].unconfirmed_txids) {
          const receipt = await connex.thor.transaction(txid).getReceipt();
          if (receipt) {
            const p = this.auths[i].unconfirmed_txids.findIndex(
              (id) => id === txid
            );
            this.auths[i].unconfirmed_txids.splice(p, 1);

            if (receipt.reverted) {
              this.votes.push({
                authAddr: this.auths[i].address,
                voteID: "",
                txID: txid,
                status: "reverted",
              });
            } else {
              this.votes.push({
                authAddr: this.auths[i].address,
                voteID: receipt.outputs[0].events[1].topics[1],
                txID: txid,
                status: "success",
              });
            }
          }
        }
      }
      await ticker.next();
    }
  }
}
</script>