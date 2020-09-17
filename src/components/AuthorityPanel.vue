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
    >Confirm Ownership of {{authAddr}}</b-button>

    <b-input-group class="m-2" prepend="Authority Key">
      <b-form-input v-model="authPriv" :state="stateKey" :disabled="!isAuth || ifSetKey"></b-form-input>
      <b-button :disabled="!isAuth || stage > 0" variant="primary" @click="genKey">Generate</b-button>
      <b-button class="mr-3" :disabled="!isAuth || stage == 3 || !stateKey " @click="setKey">Set Key</b-button>
    </b-input-group>

    <b-button-group>
      <b-button class="b1 m-2" :disabled="!isAuth || stage != 1" @click="beginTally">Begin Tally</b-button>
      <b-button class="b1 m-2" :disabled="!isAuth || stage != 2 || !ifSetKey" @click="tally">Tally</b-button>
      <b-button class="b1 m-2" :disabled="!isAuth || stage != 2" @click="endTally">End Tally</b-button>
      <b-button class="b1 m-2" :disabled="stage != 3 || res.status > 0" @click="verify">Verify</b-button>
    </b-button-group>

    <div class="container-fluid m-2 p-2">
      <b-row>
        <b-col sm="3">Total Number of Ballots</b-col>
        <b-col sm="3">Number Invalid Ballots</b-col>
        <b-col sm="3">Number of Yes Votes</b-col>
        <b-col sm="3">Verification Status</b-col>
      </b-row>
      <b-row>
        <b-col sm="3" v-if="ifCheckedVote && stage == 3">{{res.total}}</b-col>
        <b-col sm="3" v-if="ifCheckedVote && stage == 3">{{res.invalid}}</b-col>
        <b-col sm="3" v-if="ifCheckedVote && stage == 3">{{res.yes}}</b-col>
        <b-col sm="3" v-if="ifCheckedVote && stage == 3">{{statusText[res.status]}}</b-col>
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
import {
  CompressedBallot,
  uncompressBallot,
  verifyBallot,
  print,
} from "@/zkvote/binary-ballot";
import { Tally, prepTallyRes, ResForTally } from "@/zkvote/binary-tally";

import BN from "bn.js";

@Component
export default class AuthorityPanel extends Vue {
  private ifCheckedVote: boolean = false;
  private isAuth: boolean = false;
  private ifSetKey: boolean = false;

  private stageText: string[] = ["Init", "Cast", "Tally", "End", "N/A"];
  private statusText = ["Unverified", "Valid", "Invalid"];

  private voteID: string = "";

  // authority key pair
  private authAddr: string = "N/A";
  private authPriv: string = "";
  private authPub: string[] = ["0x", "0x"];

  // State of the voting process
  private stage: number = 4;

  // estimate gas usage
  private gas = {
    setPubKey: 100000,
    startTally: 40000,
    tally: 400000,
    endTally: 40000,
  };

  // tally result
  private res: {
    total: number;
    invalid: number;
    yes: number;
    status: number;
  } = { total: -1, invalid: -1, yes: -1, status: -1 };

  private valids: string[] = [];
  private invalids: string[] = [];

  // check tx that submits tally result
  private txTally: string = "";

  get stateVoteID() {
    return this.voteID.length == 66 && isHex(this.voteID);
  }

  get stateKey() {
    return this.authPriv.length == 66 && isHex(this.authPriv);
  }

  private reset() {
    this.stage = 4;
    this.authAddr = "N/A";
    this.authPriv = "";
    this.authPub = ["0x", "0x"];
    this.res = { total: -1, invalid: -1, yes: -1, status: -1 };
    this.txTally = "";

    this.valids = [];
    this.invalids = [];

    this.ifCheckedVote = false;
    this.isAuth = false;
    this.ifSetKey = false;
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
    if (out.data === "0x") {
      alert("VoteID does not exist");
      return;
    }
    console.log(`VoteID ${this.voteID} exists`);

    // Get authority account address
    this.authAddr = "0x" + out.data.slice(66 - 40, 66);

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
      await signSvc.signer(this.authAddr).request(msg);

      this.isAuth = true;
    } catch (err) {
      alert(err);
      this.isAuth = false;
      return;
    }
  }

  // Generate random private key
  private genKey() {
    this.authPriv = toHex(randPower());
  }

  // Set public key
  private async setKey() {
    // Compute public key
    const k = new BN(this.authPriv.slice(2), "hex");
    const gk = g.mul(k);

    if (this.stage > 0) {
      const _gk = await this.getPubKey();
      if (_gk[0] !== toHex(gk.getX()) || _gk[1] !== toHex(gk.getY())) {
        alert("Input key does not match the registered authority public key");
      } else {
        this.ifSetKey = true;
      }
      return;
    }

    const signingService = connex.vendor
      .sign("tx")
      .signer(this.authAddr)
      .gas(this.gas.setPubKey);
    const abi = getABI(abiVotingContract, "setAuthPubKey", "function");
    const data = encodeABI(
      abi,
      this.voteID,
      toHex(gk.getX()),
      gk.getY().isEven() ? "0x02" : "0x03"
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

      this.ifSetKey = true;

      this.authPub = [toHex(gk.getX()), toHex(gk.getY())];
    } catch (err) {
      alert(err);
    }
  }

  private async beginTally() {
    const signingService = connex.vendor
      .sign("tx")
      .signer(this.authAddr)
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

  private getCompressedBallot(out: Connex.Thor.VMOutput): CompressedBallot {
    return {
      h: toHex(out.decoded[0], 10),
      y: toHex(out.decoded[1], 10),
      zkp: [
        toHex(out.decoded[2][0], 10),
        toHex(out.decoded[2][1], 10),
        toHex(out.decoded[2][2], 10),
        toHex(out.decoded[2][3], 10),
        toHex(out.decoded[2][4], 10),
        toHex(out.decoded[2][5], 10),
        toHex(out.decoded[2][6], 10),
        toHex(out.decoded[2][7], 10),
      ],
      prefix: toHex(out.decoded[3], 10),
    };
  }

  private async tally() {
    this.valids = [];
    this.invalids = [];

    const k = new BN(this.authPriv.slice(2), "hex");
    const gk = g.mul(k);
    // console.log(`k: ${toHex(k)}
    // gkx: ${toHex(gk.getX())}
    // gky: ${toHex(gk.getY())}`);

    // Get the number of voter
    let out = await contractCall(
      connex,
      addrVotingContract,
      getABI(abiVotingContract, "getNumVoter", "function"),
      this.voteID
    );
    const n = parseInt(out.decoded[0]);

    let r: ResForTally;

    if (n > 0) {
      // New tally
      const tally = new Tally(k, this.authAddr);

      // Download, uncompress and count ballot
      for (let i = 0; i < n; i++) {
        // Get the address of the account that cast a ballot
        out = await contractCall(
          connex,
          addrVotingContract,
          getABI(abiVotingContract, "getVoter", "function"),
          this.voteID,
          i
        );
        const addr = out.decoded[0];

        // download compressed ballot
        out = await contractCall(
          connex,
          addrVotingContract,
          getABI(abiVotingContract, "getBallot", "function"),
          this.voteID,
          addr
        );

        // Get the compressed ballot
        const cb = this.getCompressedBallot(out);
        // console.log(cb)

        // Uncompress
        const b = uncompressBallot(cb, addr, gk);
        // console.log(print(b));

        if (verifyBallot(b)) {
          this.valids.push(addr);
        } else {
          this.invalids.push(addr);
        }

        // Count
        tally.count(b);
      }

      try {
        // Get the tally result in the right format for submitting
        r = prepTallyRes(tally.getRes());
      } catch (err) {
        alert(err);
        r = {
          V: 0,
          X: "0x0",
          Y: "0x0",
          zkp: ["0x0", "0x0", "0x0"],
          prefix: "0x0",
          invalidBallots: [],
        };
      }
    } else {
      // zero-ballot case
      r = {
        V: 0,
        X: "0x0",
        Y: "0x0",
        zkp: ["0x0", "0x0", "0x0"],
        prefix: "0x0",
        invalidBallots: [],
      };
    }

    console.log(`Tally res:
  #total: ${n}
  #invalid: ${this.invalids.length}
  #yes ${r.V}`)
    console.log(`valid ballots: ${this.valids}`);
    console.log(`invalid ballots: ${this.invalids}`);

    // Submit the tally result
    const signingService = connex.vendor
      .sign("tx")
      .signer(this.authAddr)
      .gas(this.gas.tally);
    const abi = getABI(abiVotingContract, "setTallyRes", "function");
    const data = encodeABI(
      abi,
      this.voteID,
      r.invalidBallots,
      r.V,
      r.X,
      r.Y,
      r.zkp,
      r.prefix
    );
    try {
      const resp = await signingService.request([
        {
          to: addrVotingContract,
          value: "0x0",
          data: data,
        },
      ]);

      console.log(`AuthPanel submit rally result:
    txid: ${resp.txid}`);

      this.txTally = resp.txid;
    } catch (err) {
      alert(err);
    }
  }

  private async endTally() {
    const signingService = connex.vendor
      .sign("tx")
      .signer(this.authAddr)
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

  // Verify tally result
  private async verify() {
    const out = await contractCall(
      connex,
      addrVotingContract,
      getABI(abiVotingContract, "verifyTallyRes", "function"),
      this.voteID
    );
    if (!out.reverted) {
      if (parseInt(out.data.slice(2)) == 0) {
        this.res.status = 0;
      } else {
        this.res.status = 1;
      }
    }
  }

  private async getPubKey(): Promise<string[]> {
    const out = await contractCall(
      connex,
      addrVotingContract,
      getABI(abiVotingContract, "getAuthPubKey", "function"),
      this.voteID
    );

    return out.reverted
      ? ["0x", "0x"]
      : [
          "0x" + out.data.slice(2, 2 + 64),
          "0x" + out.data.slice(2 + 64, 2 + 64 * 2),
        ];
  }

  private async getRes() {
    let out: Connex.Thor.VMOutput;
    out = await contractCall(
      connex,
      addrVotingContract,
      getABI(abiVotingContract, "getTallyRes", "function"),
      this.voteID
    );
    if (!out.reverted) {
      this.res.total = parseInt(out.data.slice(2, 2 + 64), 16);
      this.res.invalid = parseInt(out.data.slice(2 + 64, 2 + 64 * 2), 16);
      this.res.yes = parseInt(out.data.slice(2 + 64 * 2, 2 + 64 * 3), 16);
      if (this.res.status < 0) {
        this.res.status = 0;
      }
    }
  }

  public async created() {
    const ticker = connex.thor.ticker();
    for (;;) {
      await ticker.next();
      if (this.ifCheckedVote) {
        // check state of the voting process
        await this.getState();

        // check latest submitted tally result
        if (this.stage == 3) {
          await this.getRes();
        }

        // check confirmation of the tx that submit the tally result
        if (this.stage == 2 && this.txTally !== "") {
          const rec = await connex.thor.transaction(this.txTally).getReceipt();
          if (rec) {
            const txid = this.txTally;
            this.txTally = "";
            if (rec.reverted) {
              alert(`Tally tx ${txid} reverted`);
            } else {
              alert(`Tally tx ${txid} confirmed`);
            }
          }
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