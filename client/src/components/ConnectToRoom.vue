<template>
  <v-card class="mx-auto" max-width="400" max-height="600px" flat rounded :elevation="8">
    <v-card-title
      class="toonin-title"
    >{{cardTitle}}</v-card-title>
    <v-img
      max-height="240px"
      contain
      src="../assets/icon.png"
      style="margin-top: 1%; padding-top: 20px"
    />
    <v-card-text class="text--primary">
      <v-text-field
        v-show="connectedStatus=='disconnected' || connectedStatus=='failed'"
        v-model="roomName"
        style="color: white;"
        autofocus
        @keydown.enter="toonin"
        placeholder="Room Key"
        outlined
        rounded
      />
    </v-card-text>
    <v-card-actions>
      <v-spacer></v-spacer>
      <v-btn
        @click="handleTooninButtonClick"
        class="btn-share pr-4"
        height="42"
        outlined
        color="primary"
        rounded
      >
        <v-icon v-if="connectedStatus === 'connected'" left>mdi-stop</v-icon>
        <v-icon v-else left>$vuetify.icons.toonin</v-icon>
        {{buttonStatus}}
      </v-btn>
      <v-spacer></v-spacer>
    </v-card-actions>
  </v-card>
</template>

<script>
import { mapState } from "vuex";
import { startShare } from "../host";
const SUCCESSFUL = "connected";
const DISCONNECTED = "disconnected";
const FAILED = "failed";
const servers = {
  iceServers: [
    {
      urls: [
        "stun:stun.l.google.com:19302",
        "stun:stun2.l.google.com:19302",
        "stun:stun3.l.google.com:19302",
        "stun:stun4.l.google.com:19302"
      ]
    }
  ]
};

/* eslint no-console: ["error", { allow: ["log"] }] */

export default {
  name: "connect-to-room",
  props: {},
  data() {
    return {
      roomName: null,
      targetHost: "",
      failedHosts: [],
      peers: {}
    };
  },
  methods: {
    handleTooninButtonClick() {
      if (this.connectedStatus == "connected") {
        this.disconnect();
      } else this.toonin();
    },
    toonin() {
      this.connectToRoom();
    },
    connectToRoom(reconnecting) {
      this.$store.dispatch("UPDATE_PEERID", this.$socket.client.id);
      this.$store.dispatch("UPDATE_ROOM", this.roomName);
      if(!reconnecting) { this.setSocketListeners(); }
      this.$socket.client.emit("new peer", this.roomName);
    },
    evaluateHosts(hostPool) {
      for(var i = 0; i < hostPool.length; i++) {
        if(!this.failedHosts.includes(hostPool[i].socketID)) {
          return { hostFound: true, selectedHost: hostPool[i].socketID };
        }
      }
    },
    setSocketListeners() {
      this.$socket.$subscribe("room null", () => {
        this.roomName = "";
        this.$store.dispatch("UPDATE_ROOM", "");
      });

      this.$socket.$subscribe("host pool", (hostPool) => {
        console.log("recieved host pool to evaluate");
        const evalResult = this.evaluateHosts(hostPool.potentialHosts);
        evalResult.room = hostPool.roomID;
        console.log(evalResult);
        if(evalResult.hostFound) {
            console.log("sending eval result");
            this.$socket.client.emit("host eval res", { evalResult: evalResult });
            this.targetHost = evalResult.selectedHost;
        }
      });

      this.$socket.$subscribe("src ice", iceData => {
        if (iceData.room !== this.room || iceData.id !== this.peerID) {
          return;
        }

        this.rtcConn.addIceCandidate(new RTCIceCandidate(iceData.candidate));
      });

      this.$socket.$subscribe("src desc", descData => {
        if (descData.room !== this.room || descData.id !== this.peerID) {
          return;
        }
        const rtc = new RTCPeerConnection(servers, {
          optional: [{ RtpDataChannels: true }]
        });
        this.$store.dispatch("UPDATE_RTCCONN", rtc);
        this.attachRTCliteners();
        this.rtcConn
          .setRemoteDescription(new RTCSessionDescription(descData.desc))
          .then(() => {
            this.createAnswer();
          });
      });

      this.$socket.$subscribe("title", title => {
        this.$store.dispatch("UPDATE_STREAM_TITLE", title);
      });


      /* Listeners to convert this client into host for new peers */


      this.$socket.$subscribe("peer joined", peerData => {
        if(peerData.hostID !== this.$socket.client.id) {
          console.log("peer not for me");
          return;
        }


        this.peers[peerData.id] = {
          id: peerData.id,
          room: peerData.room,
          iceCandidates: []
        };

        startShare(peerData.id, this);
      });

      this.$socket.$subscribe("peer ice", iceData => {
        console.log("Ice Candidate from peer: " + iceData.id + " in room: " + iceData.room);
        console.log("Ice Candidate: " + iceData.candidate);

        // check if this ice data is for us or someone else in the room
        if (this.$store.getters.ROOM != iceData.room ||
          !(iceData.id in this.peers) || (iceData.hostID !== this.$socket.client.id)) {
          console.log("Ice Candidate not for me");
          return;
        }
        this.peers[iceData.id].rtcConn.addIceCandidate(new RTCIceCandidate(iceData.candidate))
          .then(console.log("Ice Candidate added successfully for peer: " + iceData.id))
          .catch(function (err) {
              console.log("Error on addIceCandidate: " + err);
          });
      });

      this.$socket.$subscribe("peer desc", descData => {
        console.log("Answer description from peer: " + descData.id + " in room: " + descData.room);
        console.log("Answer description: " + descData.desc);
        if (this.$store.getters.ROOM !== descData.room || !(descData.id in this.peers)) {
            console.log("Answer Description not for me");
            return;
        }
        this.peers[descData.id].rtcConn.setRemoteDescription(new RTCSessionDescription(descData.desc)).then(function () {
            console.log("Remote description set successfully for peer: " + descData.id);
        }).catch(function (err) {
            console.log("Error on setRemoteDescription: " + err);
        });
      });

      this.$socket.$subscribe('reconnect', req => {
        if(req.socketIDs.includes(this.$socket.client.id)) { this.reconnect(); }
      });

    },
    createAnswer() {
      this.rtcConn.createAnswer().then(desc => {
        this.rtcConn
          .setLocalDescription(new RTCSessionDescription(desc))
          .then(this.sendAnswer(desc));
      });
    },
    sendAnswer(desc) {
      this.$socket.client.emit("peer new desc", {
        id: this.peerID,
        room: this.room,
        desc: desc,
        selectedHost: this.targetHost
      });
    },
    attachRTCliteners() {
      this.rtcConn.onicecandidate = event => {
        if (!event.candidate) {
          return;
        }
        this.$socket.client.emit("peer new ice", {
          id: this.peerID,
          room: this.room,
          candidate: event.candidate,
          hostID: this.targetHost
        });
      };

      this.rtcConn.onconnectionstatechange = () => {
        if (this.rtcConn.connectionState === SUCCESSFUL) {
          this.$store.dispatch("UPDATE_CONNECTED_STATUS", SUCCESSFUL);
          this.roomName = "";
          this.rtcConn.createDataChannel('mediaDescription');
        }

        if (
          this.rtcConn.connectionState == DISCONNECTED ||
          this.rtcConn.connectionState == FAILED
        ) {
          this.failedHosts.push(this.targetHost);
          this.$socket.client.emit('logoff', { room: this.$store.getters.ROOM, socketID: this.$socket.client.id });
          this.reconnect();

          this.$store.dispatch("UPDATE_CONNECTED_STATUS", DISCONNECTED);
          this.$store.dispatch("UPDATE_ROOM", "");
          this.$store.dispatch("UPDATE_PEERID", null);
          this.$store.dispatch("UPDATE_STREAM_TITLE", "");
          this.$store.dispatch("UPDATE_PLAYING", false);
          this.$store.dispatch("UPDATE_RTCCONN", null);
          this.$store.dispatch("UPDATE_AUDIO_STREAM", null);
          this.$store.dispatch("UPDATE_VIDEO_STREAM", null);

          // disconnectBtn.$refs.link.hidden = true;
        }
      };
      
      this.rtcConn.ondatachannel = event => {
        var channel = event.channel;
        channel.onmessage = this.onDataChannelMsg;
      };

      this.rtcConn.ontrack = event => {
        var incomingStream = new MediaStream([event.track]);
        this.updateOutgoingTracks(event.track);

        var _iOSDevice = !!navigator.platform.match(
          /iPhone|iPod|iPad|Macintosh|MacIntel/
        );

        if (_iOSDevice) {
          this.$store.dispatch("UPDATE_CONNECTED_STATUS", SUCCESSFUL);
          this.$store.dispatch("UPDATE_PLAYING", false);

          if(incomingStream.getAudioTracks().length > 0) {
            this.$store.dispatch("UPDATE_AUDIO_STREAM", incomingStream);
          }
          else {
            this.$store.dispatch("UPDATE_VIDEO_STREAM", incomingStream);
          }

        } else {
          this.$store.dispatch("UPDATE_CONNECTED_STATUS", SUCCESSFUL);
          this.$store.dispatch("UPDATE_PLAYING", true);

          if(incomingStream.getAudioTracks().length > 0) {
            this.$store.dispatch("UPDATE_AUDIO_STREAM", incomingStream);
          }
          else {
            this.$store.dispatch("UPDATE_VIDEO_STREAM", incomingStream);
          }
        }

      };
    },
    updateOutgoingTracks(track) {
      var keys, senders;

        if(track.kind === 'audio') {
          keys = Object.keys(this.peers);

          for(var i = 0; i < keys.length; i++) {
            senders = this.peers[keys[i]].rtcConn.getSenders();

            if(senders[0].track.kind === 'audio') { senders[0].replaceTrack(track); }
            else { senders[1].replaceTrack(track); }
          }
        } else {
          keys = Object.keys(this.peers);

          for(var j = 0; j < keys.length; j++) {
            senders = this.peers[keys[j]].rtcConn.getSenders();

            if(senders[0].track.kind === 'video') { senders[0].replaceTrack(track); }
            else { senders[1].replaceTrack(track); }
          }
        }
    },
    onDataChannelMsg(messageEvent) {
      // data channel to recieve the media title
      try {
        var keys = Object.keys(this.peers);
        for(var i = 0; i < keys.length; i++) {
          if(this.peers[keys[i]].dataChannel.readyState === 'open') {
            this.peers[keys[i]].dataChannel.send(messageEvent.data);
          }
        }

        if(messageEvent.data === 'close') {
          this.disconnect();
          return;
        }

        var mediaDescription = JSON.parse(messageEvent.data);
        this.$store.dispatch("UPDATE_STREAM_TITLE", mediaDescription.title);
      } catch (err) {
        this.logMessage();
      }
    },
    logMessage() {
      // continue regardless of error
    },
    reconnect() {
      this.rtcConn.close();
      this.$store.dispatch("UPDATE_AUDIO_STREAM", null);
      this.$store.dispatch("UPDATE_VIDEO_STREAM", null);
      this.$store.dispatch("UPDATE_PEERID", null);
      this.$store.dispatch("UPDATE_RTCCONN", null);

      var roomKey = this.$store.getters.ROOM;
      this.roomName = roomKey;
      this.connectToRoom(true);
    },
    disconnect() {
      this.$socket.client.emit('logoff', { room: this.$store.getters.ROOM, socketID: this.$socket.client.id });
      this.rtcConn.close();
      this.$store.dispatch("UPDATE_AUDIO_STREAM", null);
      this.$store.dispatch("UPDATE_VIDEO_STREAM", null);
      this.$store.dispatch("UPDATE_CONNECTED_STATUS", DISCONNECTED);
      this.$store.dispatch("UPDATE_ROOM", "");
      this.$store.dispatch("UPDATE_PEERID", null);
      this.$store.dispatch("UPDATE_STREAM_TITLE", "");
      this.$store.dispatch("UPDATE_PLAYING", false);
      this.$store.dispatch("UPDATE_RTCCONN", null);

    }
  },
  computed: {
    buttonStatus() {
      if (this.connectedStatus == "connected") {
        return "Disconnect";
      }
      return "Toonin";
    },
    cardTitle() {
      if (this.connectedStatus == "connected") {
        return `Connected to ${this.room}`;
      }
      return "Connect to a room";
    },
    ...mapState([
      "room",
      "rtcConn",
      "streamTitle",
      "playing",
      "connectedStatus",
      "peerID",
      "audioStream",
      "videoStream"
    ])
  },
  mounted: function() {
    if (this.$route.params.room) {
      this.roomName = this.$route.params.room;
      setTimeout(() => this.toonin(), 500);
    }
  }
};
</script>

<style>
div.v-text-field {
  width: 100%;
  margin-right: 0%;
}
</style>
