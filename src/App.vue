<template>
  <div class="m-1" id="app">
    <!-- <img alt="Vue logo" src="./assets/logo.png" /> -->
    <!-- <HelloWorld msg="Welcome to Your Vue.js + TypeScript App" /> -->
    <h1 v-if="!isTestNet">VeChainThor TestNet Required</h1>
    <h1 v-if="isTestNet">ZKVote APP</h1>
    <NewVote v-if="isTestNet"/>
    <Cast v-if="isTestNet"/>
    <ListBallots v-if="isTestNet"/>
    <ListVotes v-if="isTestNet"/>
    <!-- <AuthorityPanel v-if="isTestNet" /> -->
  </div>
</template>

<script lang="ts">
import { Component, Vue } from "vue-property-decorator";
// import HelloWorld from "./components/HelloWorld.vue";
// import AuthorityPanel from "./components/AuthorityPanel.vue";
import NewVote from "./components/NewVote.vue"
import Cast from "@/components/Cast.vue"
import ListVotes from "./components/ListVotes.vue";
import ListBallots from "@/components/ListBallots.vue"

import "@vechain/connex";

@Component({
  components: {
    ListVotes,
    ListBallots,
    NewVote,
    Cast,
  },
})
export default class App extends Vue {
  private isTestNet = true;

  public async created() {
    this.isTestNet = await this.checkNet();
  }

  private async checkNet() {
    const block = connex.thor.block(0);
    const firstBlock = await block.get();
    return (
      firstBlock!.id ===
      "0x000000000b2bce3c70bc649a02749e8687721b09ed2e15997f466536b20bb127"
    );
  }
}
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
