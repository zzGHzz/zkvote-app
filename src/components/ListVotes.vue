<template>
  <div class="list-vote">
    <b-form-group
      id="auth-address"
      label="Enter authority account address"
      label-for="input-1"
      :state="state"
    >
      <b-form-input id="input-1" v-model="address" @keyup.enter="getVoteIDs()" :state="state" trim></b-form-input>
    </b-form-group>

    <b-container>
      <b-row>
        <b-col sm="2">Index</b-col>
        <b-col sm="9">Vote ID</b-col>
      </b-row>
      <b-row v-for="(id, index) in voteIDs" :key="id">
        <b-col sm="2">
          <code>{{ index }}</code>:
        </b-col>
        <b-col sm="9">{{id}}</b-col>
      </b-row>
    </b-container>

    <!-- <b-form-input v-model="address" @keyup.enter="getVoteIDs()"></b-form-input> -->
    <!-- <ul>
        <li v-for="(id, index) in voteIDs" :key="id">VoteID {{index+1}}: {{id}}</li>
    </ul>-->
  </div>
</template>

<script lang="ts">
import { Component, Vue } from "vue-property-decorator";

import { isAddress, getABI } from "myvetools/dist/utils";
import { contractCall } from "myvetools/dist/connexUtils";
import { abiVotingContract, addrVotingContract } from "../common";

@Component
export default class ListVotes extends Vue {
  private address: string = "";
  private lastValidAddress: string = "";
  private voteIDs: string[] = [];
  private nVote: number = 0;
  private abiVoterID = getABI(abiVotingContract, "voteID", "function");

  // get invalidFeedback() {
  //   if (this.address === "") {
  //     return "Emptyp address";
  //   }
  //   if (!isAddress(this.address)) {
  //     return "Invalid account address";
  //   }

  //   return "";
  // }

  get state() {
    return isAddress(this.address) ? true : false;
  }

  private lpad(hexStr: string, totalLen: number): string {
    const len = hexStr.length - 2; // hex string assumed to begin with '0x'

    if (len >= totalLen) {
      return hexStr;
    }

    return "0x" + "0".repeat(totalLen - len) + hexStr.slice(2);
  }

  private async getVoteIDs() {
    this.voteIDs = [];
    if (!isAddress(this.address)) {
      console.log("Invalid address: " + this.address);
      if (this.nVote > 0) {
        this.address = this.lastValidAddress;
      } else {
        this.address = "";
      }
      return;
    }

    if (this.lastValidAddress !== this.address) {
      this.voteIDs = [];
    }
    this.lastValidAddress = this.address;

    for (;;) {
      try {
        const out = await contractCall(
          connex,
          addrVotingContract,
          this.abiVoterID,
          this.address,
          this.voteIDs.length
        );
        if (out.data === "0x") {
          return;
        }
        this.voteIDs.push(out.data);
      } catch (e) {
        alert(this.voteIDs.length + " " + e);
        return;
      }
    }
  }

  private async checkNewVote(address: string, n: number): Promise<boolean> {
    try {
      const out = await contractCall(
        connex,
        addrVotingContract,
        this.abiVoterID,
        this.address,
        this.voteIDs.length
      );
      if (out.data === "0x") {
        return new Promise((resolve) => resolve(false));
      }
    } catch (e) {
      console.log(this.voteIDs.length + " " + e);
      return new Promise((resolve) => resolve(false));
    }

    return new Promise((resolve) => resolve(true));
  }

  public async created() {
    const ticker = connex.thor.ticker();
    for (;;) {
      if (
        this.address === this.lastValidAddress &&
        this.lastValidAddress !== ""
      ) {
        const check = await this.checkNewVote(
          this.lastValidAddress,
          this.voteIDs.length
        );
        if (check) {
          await this.getVoteIDs();
        }
      }
      await ticker.next();
    }
  }
}
</script>

<style>
</style>