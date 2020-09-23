<template>
  <div class="border rounded d-flex m-2 p-2 flex-column">
    <h3 class="text-left m-2">List all the ballots</h3>
    <b-input-group class="m-2" prepend="Vote ID">
      <b-form-input v-model="voteID" :state="state" :disabled="ifCheckedVote"></b-form-input>
      <b-button
        class="mr-3"
        :disabled="!state"
        @click="check"
      >{{ !ifCheckedVote ? "Check & Lock" : "UnLock" }}</b-button>
    </b-input-group>

    <b-container class="m-2" fluid>
      <b-row>
        <b-col sm="1">Index</b-col>
        <b-col sm="3">Cast By</b-col>
        <b-col sm="2">Status</b-col>
        <b-col sm="3">Ballot Hash</b-col>
      </b-row>
      <b-row v-for="(b, index) in ballots" :key="b.hash">
        <b-col sm="1">
          <code>{{ index }}</code>:
        </b-col>
        <b-col sm="3">{{shortHex(b.signer)}}</b-col>
        <b-col sm="2">{{statusTexts[b.status]}}</b-col>
        <b-col sm="3">{{shortHex(b.hash)}}</b-col>
        <b-col>
          <b-button class="mb-2" :disabled="b.status > 0" @click="verify(index)">Verify</b-button>
        </b-col>
      </b-row>
    </b-container>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from "vue-property-decorator";

import { isHex, getABI } from "myvetools/dist/utils";
import { contractCall } from "myvetools/dist/connexUtils";
import { abiVotingContract, addrVotingContract } from "../common";

import { cry } from "thor-devkit";
import BN from "bn.js";

import LRU from "lru-cache";

@Component
export default class ListBallots extends Vue {
  private voteID: string = "";
  private lastValidVoteID: string = "";
  private ifCheckedVote: boolean = false;

  private ballots: {
    signer: string;
    hash: string;
    status: number; // 0-unverified, 1-verfied/true, 2-verified/false
  }[] = [];

  private statusTexts = ["Unverified", "Valid", "Invalid"];

  get state() {
    return isHex(this.voteID) && this.voteID.length == 66;
  }

  private async verify(index: number) {
    console.log(`verify ballot:
  voteID: ${this.voteID}
  signer: ${this.ballots[index].signer}`);

    const out = await contractCall(
      connex,
      addrVotingContract,
      getABI(abiVotingContract, "verifyBallot", "function"),
      this.voteID,
      this.ballots[index].signer
    );

    if (out.data.charAt(out.data.length - 1) === "1") {
      this.ballots[index].status = 1;
    } else {
      this.ballots[index].status = 2;
    }
  }

  private shortHex(h: string): string {
    if (h.length <= 10) {
      return h;
    }
    return h.slice(0, 6) + "..." + h.slice(h.length - 4);
  }

  private calBallotHash(hx: Buffer, yx: Buffer, prefix: string): string {
    const preh = Buffer.from("0".repeat(62) + prefix.slice(2, 4), "hex");
    const prey = Buffer.from("0".repeat(62) + prefix.slice(4, 6), "hex");

    return "0x" + cry.keccak256(hx, preh, yx, prey).toString("hex");
  }

  private async getBallots() {
    let out: Connex.Thor.VMOutput;

    // Get the number of ballots
    out = await contractCall(
      connex,
      addrVotingContract,
      getABI(abiVotingContract, "getNumVoter", "function"),
      this.voteID
    );
    const n = parseInt(out.data, 16);
    console.log(`n = ${n}`);

    for (let i = 0; i < n; i++) {
      // Get the account that cast the i'th ballot
      out = await contractCall(
        connex,
        addrVotingContract,
        getABI(abiVotingContract, "getVoter", "function"),
        this.voteID,
        i
      );
      const signer = "0x" + out.data.slice(66 - 40, 66);

      // Get the i'th ballot
      out = await contractCall(
        connex,
        addrVotingContract,
        getABI(abiVotingContract, "getBallot", "function"),
        this.voteID,
        signer
      );

      // Calculate hash
      const hx = Buffer.from(out.data.slice(2, 2 + 64), "hex"); // 1st uint256
      const yx = Buffer.from(out.data.slice(2 + 64, 2 + 64 * 2), "hex"); // 2nd uint 256
      const prefix = // last 12bits of the 11th uint256
        "0x" + out.data.slice(2 + 64 * 10, 2 + 64 * 11).slice(64 - 6 * 2, 64);

      const h = this.calBallotHash(hx, yx, prefix);
      this.ballots.push({ signer: signer, status: 0, hash: h });
    }
  }

  private async check() {
    // unlock the input txt bar
    if (this.ifCheckedVote) {
      this.ifCheckedVote = false;
      return;
    }

    // if voteID doesn't change, do nothing
    if (this.voteID === this.lastValidVoteID) {
      this.ifCheckedVote = true;
      return;
    }

    // check the existence of voteID
    let out = await contractCall(
      connex,
      addrVotingContract,
      getABI(abiVotingContract, "voteAuth", "function"),
      this.voteID
    );
    if (parseInt(out.data.slice(2), 16) == 0) {
      alert("VoteID does not exist");
      this.voteID = this.lastValidVoteID;
      return;
    }
    console.log(`VoteID ${this.voteID} exists`);

    this.ballots = [];
    await this.getBallots();

    this.ifCheckedVote = true;
    this.lastValidVoteID = this.voteID;
  }

  public async created() {
    const castEvent = connex.thor
      .account(addrVotingContract)
      .event(getABI(abiVotingContract, "CastBinaryBallot", "event"));
    const ticker = connex.thor.ticker();
    for (;;) {
      const blk = await connex.thor.block().get();
      if (blk === null) {
        continue;
      }

      //only update when the input text bar is disabled
      if (this.ifCheckedVote) {
        // Set event conditions: id = this.voteID and only appearing in the latest block
        const f = castEvent.filter([{ id: this.voteID }]).range({
          unit: "block",
          from: blk!.number,
          to: blk!.number,
        });

        // Get events
        const evs = await f.apply(0, 256);

        for (let ev of evs) {
          // Get signer and ballot hash
          const signer = "0x" + ev.topics[2].slice(66 - 40, 66);
          const hash = ev.data;

          console.log(`Find cast event:
  voteID: ${this.voteID}
  signer: ${signer}
  hash: ${hash}`);

          const i = this.ballots.findIndex((b) => b.signer === signer);
          if (i == -1) {
            // cast by a new address, add the info
            this.ballots.push({ signer: signer, hash: hash, status: 0 });

            console.log(`Add ballot: 
signer: ${signer}
hash: ${hash}
status: 0`);
          } else {
            // cast by an existing address, then update info if the ballot hash changes
            if (this.ballots[i].hash !== hash) {
              this.ballots[i].hash = hash;
              this.ballots[i].status = 0;

              console.log(`Modify ballot hash:
signer: ${this.ballots[i].signer}
hash: from ${this.ballots[i].hash} to ${hash}
status: from ${this.ballots[i].status} to 0`);
            }
          }
        }
      }
      await ticker.next();
    }
  }
}
</script>

<style>
</style>