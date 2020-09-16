<template>
  <div class="border rounded d-flex m-2 p-2 flex-column">
    <h3 class="text-left font-weight-bold m-2">Authority Panel</h3>

    <b-input-group class="m-2" prepend="Vote ID">
      <b-form-input v-model="voteID" :state="stateVoteID" :disabled="ifCheckedVote"></b-form-input>
      <b-button
        class="mr-3"
        :disabled="!stateVoteID"
        @click="check"
      >{{ !ifCheckedVote ? "Check & Lock" : "UnLock" }}</b-button>
    </b-input-group>

    <h5 class="text-left m-2">Stage: {{stageText[stage]}}</h5>

    <b-button
      class="m-2"
      :disabled="!ifCheckedVote || isAuth"
      @click="confirm"
    >Confirm Ownership of {{auth}}</b-button>

    <b-input-group class="m-2" prepend="Authority Key">
      <b-form-input v-model="k" :state="stateKey" :disabled="!isAuth || stage > 0"></b-form-input>
      <b-button :disabled="!isAuth || stage > 0" variant="primary" @click="genKey">Generate</b-button>
      <b-button
        class="mr-3"
        :disabled="!isAuth || !stateKey || stage > 0"
        @click="setPubKey"
      >Set Public Key</b-button>
    </b-input-group>

    <b-butter-group class="d-flex">
      <b-button class="b1 m-2" :disabled="!isAuth || stage != 1" @click="beginTally">Begin Tally</b-button>
      <b-button class="b1 m-2" :disabled="!isAuth || stage != 2" @click="tally">Tally</b-button>
      <b-button class="b1 m-2" :disabled="!isAuth || stage != 2" @click="endTally">End Tally</b-button>
      <b-button class="b1 m-2" :disabled="!isAuth || stage != 3" @click="verify">Verify</b-button>
    </b-butter-group>

    <div class="container-fluid m-2 p-2">
      <b-row>
        <b-col sm="3">Total Number of Ballots</b-col>
        <b-col sm="3">Number Invalid Ballots</b-col>
        <b-col sm="3">Number of Yes Votes</b-col>
        <b-col sm="3">Verification Status</b-col>
      </b-row>
      <b-row>
        <b-col sm="3"></b-col>
        <b-col sm="3"></b-col>
        <b-col sm="3"></b-col>
        <b-col sm="3">{{statusTexts[status]}}</b-col>
      </b-row>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from "vue-property-decorator";
import { isHex, getABI } from "myvetools/dist/utils";
import { contractCall, encodeABI } from "myvetools/dist/connexUtils";

import { abiVotingContract, addrVotingContract } from "@/common";
import { randAddress, randPower, toHex } from "@/zkvote/utils";
import { g } from "@/zkvote/ec";

import BN from "bn.js";

@Component
export default class AuthorityPanel extends Vue {
  private voteID: string = "";
  private lastValidVoteID: string = "";
  private ifCheckedVote: boolean = false;

  private isAuth: boolean = false;
  private auth: string = "N/A";

  private k: string = "";
  private gk: string[] = ["0x", "0x"];

  private stage: number = 4;
  private stageText: string[] = ["Init", "Cast", "Tally", "End", "N/A"];

  private status = 3;
  private statusTexts = ["Unverified", "Valid", "Invalid", ""];

  private gas = {
    setPubKey: 100000,
    startTally: 40000,
    tally: 400000,
    endTally: 40000,
  };

  get stateVoteID() {
    return this.voteID.length == 66 && isHex(this.voteID);
  }

  get stateKey() {
    return this.k.length == 66 && isHex(this.k);
  }

  private reset() {
    this.stage = 4;
    this.status = 3;
    this.auth = "N/A";
    this.k = "";
    this.gk = ["0x", "0x"];

    this.ifCheckedVote = false;
    this.isAuth = false;
  }

  private async getState() {
    const out = await contractCall(
      connex,
      addrVotingContract,
      getABI(abiVotingContract, "getState", "function"),
      this.voteID
    );
    this.stage = parseInt(out.data.slice(2), 16);

    return true;
  }

  private async check() {
    // unlock the input txt bar and reset internal variables
    if (this.ifCheckedVote) {
      this.reset();
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

    // Get authority account address
    this.auth = "0x" + out.data.slice(66 - 40, 66);

    // Get the current state of the vote
    this.getState();

    this.ifCheckedVote = true;
  }

  // confirm the ownership of the authority account
  private async confirm() {
    const signSvc = connex.vendor.sign("cert");
    let result: Connex.Vendor.CertResponse;
    const msg: Connex.Vendor.CertMessage = {
      purpose: "identification",
      payload: {
        type: "text",
        content: "Confirm ownership",
      },
    };
    try {
      await signSvc.signer(this.auth).request(msg);
      this.isAuth = true;
    } catch (err) {
      alert(err);
      this.isAuth = false;
      return;
    }
  }

  // Generate random private key
  private genKey() {
    this.k = toHex(randPower());
  }

  // Set public key
  private async setPubKey() {
    // Compute public key
    const pt = g.mul(new BN(this.k.slice(2), "hex"));
    this.gk = [toHex(pt.getX()), toHex(pt.getY())];

    const signingService = connex.vendor
      .sign("tx")
      .signer(this.auth)
      .gas(this.gas.setPubKey);
    const abi = getABI(abiVotingContract, "setAuthPubKey", "function");
    const data = encodeABI(
      abi,
      this.voteID,
      toHex(pt.getX()),
      pt.getY().isEven() ? "0x02" : "0x03"
    );
    try {
      const resp = await signingService.request([
        {
          to: addrVotingContract,
          value: "0x0",
          data: data,
        },
      ]);
      console.log(`AuthPanel set authority public key: 
txid: ${resp.txid}
signer: ${resp.signer}
voteID: ${this.voteID}`);
    } catch (err) {
      alert(err);
    }
  }

  private async beginTally() {
    const signingService = connex.vendor
      .sign("tx")
      .signer(this.auth)
      .gas(this.gas.startTally);
    const abi = getABI(abiVotingContract, "beginTally", "function");
    const data = encodeABI(abi, this.voteID);
    try {
      const resp = await signingService.request([
        {
          to: addrVotingContract,
          value: "0x0",
          data: data,
        },
      ]);

      console.log(`AuthPanel begin tally: 
txid: ${resp.txid}`);
    } catch (err) {
      alert(err);
    }
  }

  private async tally() {}

  private async endTally() {
    const signingService = connex.vendor
      .sign("tx")
      .signer(this.auth)
      .gas(this.gas.endTally);
    const abi = getABI(abiVotingContract, "endTally", "function");
    const data = encodeABI(abi, this.voteID);
    try {
      const resp = await signingService.request([
        {
          to: addrVotingContract,
          value: "0x0",
          data: data,
        },
      ]);

      console.log(`AuthPanel end tally: 
txid: ${resp.txid}`);
    } catch (err) {
      alert(err);
    }
  }

  private async verify() {}

  private async getPubKey() {
    const out = await contractCall(
      connex,
      addrVotingContract,
      getABI(abiVotingContract, "getAuthPubKey", "function"),
      this.voteID
    );
    console.log("out.data = " + out.data);
    this.gk[0] = "0x" + out.data.slice(2, 2 + 64);
    this.gk[1] = "0x" + out.data.slice(2 + 64, 2 + 64 * 2);

    return true;
  }

  public async created() {
    const ticker = connex.thor.ticker();
    for (;;) {
      await ticker.next();
      if (this.ifCheckedVote) {
        await this.getState();
        if (this.stage > 0 && this.gk[0] === "0x") {
          await this.getPubKey();
          console.log("Get pubkey: " + this.gk[0] + this.gk[1]);
        }
      }
    }
  }
}
</script>

<style>
.b1 {
  width: 30%;
}
</style>