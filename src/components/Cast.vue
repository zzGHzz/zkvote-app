<template>
  <div class="border rounded d-flex m-2 p-2 flex-column">
    <h3 class="text-left font-weight-bold m-2">Cast a ballot</h3>
    <b-input-group class="m-2" id="vote-id" prepend="VoteID">
      <b-form-input id="input-1" v-model="voteID" :state="state" :disabled="ifCheckedVote" trim></b-form-input>
      <b-button
        class="mr-3"
        :disabled="!state"
        @click="check"
      >{{ !ifCheckedVote ? "Check & Lock" : "UnLock" }}</b-button>
    </b-input-group>

    <b-button
      class="m-2"
      :disabled="!ifCheckedVote"
      @click="select"
    >{{ !ifSelectedAccount? "Select an account" : "Select another account" }}</b-button>
    <label class="m-2">Selected account: {{signer}}</label>

    <b-form-group
      class="m-2"
      label="Choose YES or NO"
      :disabled="!ifCheckedVote || !ifSelectedAccount"
    >
      <b-form-radio v-model="selected" value="yes">YES</b-form-radio>
      <b-form-radio v-model="selected" value="no">NO</b-form-radio>
    </b-form-group>

    <b-button
      class="m-2"
      :disabled="!ifCheckedVote || !ifSelectedAccount || (selected !== `yes` && selected !== `no`)"
      @click="cast"
    >Cast</b-button>

    <b-container class="m-2" fluid>
      <b-row>
        <b-col sm="3">Casted By</b-col>
        <b-col sm="2">TxID</b-col>
        <b-col sm="1">Status</b-col>
        <b-col sm="5">Ballot Hash</b-col>
      </b-row>
      <b-row v-for="c in casted" :key="c.txID">
        <b-col sm="3">{{ c.signer }}</b-col>
        <b-col sm="2">{{ shortHex(c.txID) }}</b-col>
        <b-col sm="1">{{ c.status }}</b-col>
        <b-col sm="5">{{ c.ballotHash }}</b-col>
      </b-row>
    </b-container>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from "vue-property-decorator";
import BN from "bn.js";

import { isHex, getABI } from "myvetools/dist/utils";
import { contractCall, encodeABI } from "myvetools/dist/connexUtils";
import { abiVotingContract, addrVotingContract } from "../common";
import { Ballot, generateBallot, compressBallot } from "@/zkvote/binary-ballot";
import { randPower, toHex } from "@/zkvote/utils";
import { point } from "@/zkvote/ec";

@Component
export default class Cast extends Vue {
  private voteID: string = "";
  private ifCheckedVote: boolean = false;
  private gk: string[] = [];

  private signer: string = "N/A";
  private ifSelectedAccount: boolean = false;

  private txs: Set<{ signer: string; txid: string }> = new Set();
  private selected: string = "";
  private casted: {
    signer: string;
    txID: string;
    status: "success" | "reverted";
    ballotHash: string;
  }[] = [];

  private gas = 350000;

  get state() {
    return isHex(this.voteID) && this.voteID.length == 66;
  }

  private shortHex(h: string): string {
    if (h.length <= 10) {
      return h;
    }
    return h.slice(0, 6) + "..." + h.slice(h.length - 4);
  }

  // Users have to commit to using a certain account before casting
  public async select() {
    const signSvc = connex.vendor.sign("cert");
    let result: Connex.Vendor.CertResponse;
    const msg: Connex.Vendor.CertMessage = {
      purpose: "identification",
      payload: {
        type: "text",
        content: "Select account to sign tx",
      },
    };
    try {
      result = await signSvc.request(msg);
      this.signer = result.annex.signer;
      this.ifSelectedAccount = true;
    } catch (error) {
      this.ifSelectedAccount = false;
      this.signer = "N/A";
      return;
    }
  }

  private async check() {
    if (this.ifCheckedVote) {
      this.ifCheckedVote = false;
      this.ifSelectedAccount = false;
      this.signer = "N/A";
      return;
    } else if (!isHex(this.voteID) || this.voteID.length != 66) {
      return;
    }

    // Check the existance of voteID
    let out = await contractCall(
      connex,
      addrVotingContract,
      getABI(abiVotingContract, "voteAddr", "function"),
      this.voteID
    );
    if (parseInt(out.data.slice(2), 16) == 0) {
      alert("VoteID does not exist");
      return;
    }

    this.ifCheckedVote = true;

    // Get the auth public key
    out = await contractCall(
      connex,
      addrVotingContract,
      getABI(abiVotingContract, "getAuthPubKey", "function"),
      this.voteID
    );

    if (out.reverted) {
      alert("Authority public key not set!");
      this.ifCheckedVote = false;
      this.ifSelectedAccount = false;
      this.signer = "N/A";
      return;
    }

    this.gk = ["0x" + out.data.slice(2, 66), "0x" + out.data.slice(66, 130)];
  }

  private async cast() {
    const gk = point(
      new BN(this.gk[0].slice(2), "hex"),
      new BN(this.gk[1].slice(2), "hex")
    );

    // Generate a ballot
    const a = randPower();
    const val = this.selected == "yes";
    const ballot = generateBallot({
      a: a,
      gk: gk,
      v: val,
      address: this.signer,
    });
    const b = compressBallot(ballot);

    const signingService = connex.vendor.sign("tx");
    signingService.gas(this.gas).signer(this.signer);
    const abi = getABI(abiVotingContract, "cast", "function");
    const data = encodeABI(abi, this.voteID, b.h, b.y, b.zkp, b.prefix);
    try {
      const resp = await signingService.request([
        {
          to: addrVotingContract,
          value: "0x0",
          data: data,
        },
      ]);
      this.txs.add(resp);

      console.log(`Cast add to the unconfirmed tx list: 
  txid: ${resp.txid}`);
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

          console.log(`Cast remove from the unconfirmed tx list:
  signer: ${tx.signer}
  txid: ${tx.txid}`);

          if (receipt.reverted) {
            this.casted.push({
              signer: tx.signer,
              txID: tx.txid,
              status: "reverted",
              ballotHash: "N/A",
            });

            console.log(`Cast tx reverted:
  signer: ${tx.signer}
  txid: ${tx.txid}`);
          } else {
            this.casted.push({
              signer: tx.signer,
              ballotHash: receipt.outputs[0].events[0].data,
              txID: tx.txid,
              status: "success",
            });

            console.log(`Cast tx success:
  signer: ${tx.signer}
  txid: ${tx.txid}
  hash: ${receipt.outputs[0].events[0].data}`);
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