<template>
  <div class="border rounded d-flex m-2 p-2 flex-column">
    <h3 class="text-left font-weight-bold m-2">Create a new vote</h3>
    <b-button class="m-2" variant="primary" v-on:click="newVote()">New Vote</b-button>
    <b-container class="m-2" fluid>
      <b-row>
        <b-col sm="3">Authority Account</b-col>
        <b-col sm="2">TxID</b-col>
        <b-col sm="1">Status</b-col>
        <b-col sm="5">Vote ID</b-col>
      </b-row>
      <b-row v-for="vote in votes" :key="vote.txid">
        <b-col sm="3">{{ vote.authAddr }}</b-col>
        <b-col sm="2">{{ shortHex(vote.txID) }}</b-col>
        <b-col sm="1">{{ vote.status }}</b-col>
        <b-col sm="5">{{ vote.voteID }}</b-col>
      </b-row>
    </b-container>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from "vue-property-decorator";

import { getABI } from "myvetools/dist/utils";
import { encodeABI } from "myvetools/dist/connexUtils";
import {
  abiVoteCreator,
  addrVoteCreator,
  abiVotingContract,
  addrVotingContract,
} from "../common";

@Component
export default class NewVote extends Vue {
  private txs: Set<{ signer: string; txid: string }> = new Set();

  private votes: {
    authAddr: string;
    voteID: string;
    txID: string;
    status: "success" | "reverted";
  }[] = [];

  private shortHex(h: string): string {
    if (h.length <= 10) {
      return h;
    }
    return h.slice(0, 6) + "..." + h.slice(h.length - 4);
  }

  private async newVote() {
    const signingService = connex.vendor.sign("tx");
    signingService.gas(5000000);
    const abi = getABI(abiVoteCreator, "newBinaryVote", "function");
    const data = encodeABI(abi);
    try {
      const resp = await signingService.request([
        {
          to: addrVoteCreator,
          value: "0x0",
          data: data,
        },
      ]);
      this.txs.add(resp);
    } catch (e) {
      alert(e);
    }
  }

  public async created() {
    const ticker = connex.thor.ticker();
    for (;;) {
      for (let tx of this.txs) {
        const receipt = await connex.thor.transaction(tx.txid).getReceipt();
        if (receipt) {
          this.txs.delete(tx);
          if (receipt.reverted) {
            this.votes.push({
              authAddr: tx.signer,
              voteID: "",
              txID: tx.txid,
              status: "reverted",
            });
          } else {
            this.votes.push({
              authAddr: tx.signer,
              voteID: receipt.outputs[0].events[1].topics[1],
              txID: tx.txid,
              status: "success",
            });
          }
        }
      }
      await ticker.next();
    }
  }
}
</script>