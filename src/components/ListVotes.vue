<template>
  <div class="border rounded d-flex m-2 p-2 flex-column">
    <h3 class="text-left font-weight-bold m-2">List all the votes created by</h3>
    <b-input-group class="m-2" prepend="Authority Account">
      <b-form-input class="mr-3" v-model="address" @keyup.enter="getVoteIDs()" :state="state" trim></b-form-input>
    </b-input-group>

    <b-container class="m-2" fluid>
      <b-row>
        <b-col sm="3">Index</b-col>
        <b-col sm="7">Vote ID</b-col>
      </b-row>
      <b-row v-for="(id, index) in voteIDs" :key="id">
        <b-col sm="3">
          <code>{{ index }}</code>:
        </b-col>
        <b-col sm="7">{{id}}</b-col>
      </b-row>
    </b-container>
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

  get state() {
    return isAddress(this.address);
  }

  private async getVoteIDs() {
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
        if (out.data === "0x" + "0".repeat(64)) {
          return;
        }
        this.voteIDs.push(out.data);
      } catch (e) {
        alert(e);
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