<template>
  <div class="body-container">
    <div style="margin-bottom: 0%;">
    <v-text-field
      :value="roomName"
      @change="setRoomName"
      append-icon="mdi-cached"
      @click:append="randomRoomName"
      :error-messages="roomNameInputErrorMessages"
      label="Name your room"
      color="primary"
      outlined
      rounded
      autofocus
      @keydown.enter="startShare"
    ></v-text-field>

      <v-checkbox 
      style="margin-bottom: 0%; margin-top: -5%; margin-left: 5%;"
      label="Share Screen"
      color="primary"
      @change="toggleScreenShare"
      >
      </v-checkbox>
      
      <v-switch
      label="60 FPS"
      style="margin-bottom: 0%; margin-left: 5%; margin-top: -5%;"
      color="primary"
      @change="toggle60FpsVideo"
      >
      </v-switch>

      <v-checkbox 
      style="margin-bottom: 0%; margin-top: -5%; margin-left: 5%;"
      label="Distributed Streaming (Experimental)"
      color="red"
      @change="toggleDistributedStreaming"
      >
      </v-checkbox>
    </div>
      
    <v-btn
    :disabled="!roomNameValid"
    @click="startShare"
    class="btn-share"
    height="42"
    outlined
    color="primary"
    rounded
    style="margin-left: 10%;"
    >
    <v-icon left>$vuetify.icons.toonin</v-icon>Share
    </v-btn>
  </div>
</template>

<script>
import { mapState, mapActions, mapMutations } from "vuex";

export default {
  name: "home-view",
  computed: {
    ...mapState(["roomName", "roomNameValid", "roomNameInputErrorMessages"])
  },
  methods: {
    ...mapActions(["randomRoomName", "startShare", "toggleScreenShare", "toggleDistributedStreaming", "toggle60FpsVideo"]),
    ...mapMutations(["setRoomName"])
  }
};
</script>

<style>
</style>