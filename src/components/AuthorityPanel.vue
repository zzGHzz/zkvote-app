<template>
  <b-form-group
    id="auth-address"
    label="Enter vote ID"
    label-for="input-1"
    :invalid-feedback="invalidFeedback"
    :state="state"
  >
    <b-form-input id="input-1" v-model="voteID" :state="state" trim></b-form-input>
  </b-form-group>
</template>

<script lang="ts">
import { Component, Vue } from "vue-property-decorator";
import { isHex, getABI } from "myvetools/dist/utils";
import { contractCall } from 'myvetools/dist/connexUtils'

import { abiVotingContract, addrVotingContract } from "../common";

@Component
export default class AuthorityPanel extends Vue {
  private voteID: string = "";
  private stage: number = 0;
  private authAddress: string = "";
  private txID: string = "";

  get invalidFeedback() {
    if (this.voteID === "") {
      return "Emptyp vote ID";
    }
    if (this.voteID.length != 66 || !isHex(this.voteID)) {
      return "Invalid vote ID";
    }

    return "";
  }

  get state() {
    return this.voteID.length == 66 && isHex(this.voteID);
  }

  private async checkOwnership() {
    
  }
}
</script>