<template>
  <q-page padding>
    <div class="video-container">
      <div class="local-video-wrapper">
        <video id="localVideo" autoplay muted class="local-video"></video>
      </div>
    </div>
  </q-page>
</template>

<script setup>
import { ref, onMounted } from "vue";
import Peer from "peerjs";
import axios from "axios";
const userId = ref(localStorage.getItem("userId") || "");
const peerId = ref("");
const error = ref("");
const remoteStreams = ref({});
const roomId = ref("");
let localPeer;
let localStream;
let currentCalls = {};
let screenStream = null;

onMounted(() => {
  initializePeer();
  getLocalStream();
});
function initializePeer() {
  localPeer = new Peer(undefined, {
    stunServers: "stun:stun.l.google.com:19302",
  });

  localPeer.on("open", (id) => {
    peerId.value = id;
    createUser();
  });

  localPeer.on("call", handleIncomingCall);
}

async function getLocalStream() {
  try {
    localStream = await navigator.mediaDevices.getUserMedia({
      video: true,
      audio: true,
    });
    document.getElementById("localVideo").srcObject = localStream;
  } catch (err) {
    handleError("Failed to get local stream: " + err.message);
  }
}
async function createUser() {
  if (!userId.value || !peerId.value) {
    return handleError("Please enter both User ID and Peer ID");
  }

  try {
    await axios.post("https://192.168.1.6:3000/api/save-peer-id", {
      userId: userId.value,
      peerId: peerId.value,
    });
    clearError();
  } catch (err) {
    handleError("Failed to save User ID and Peer ID: " + err.message);
  }
}
async function callUser(contactUserId) {
  try {
    const response = await axios.get(
      `https://192.168.1.6:3000/api/get-peer-id/${contactUserId}`
    );

    const peerIdToCall = response.data.peerId;

    if (!peerIdToCall) {
      return handleError("No Peer ID found for this User ID: " + contactUserId);
    }

    const call = localPeer.call(peerIdToCall, localStream);
    handleCallStream(call);
  } catch (err) {
    handleError(
      "Failed to retrieve Peer ID for user " +
        contactUserId +
        ": " +
        err.message
    );
  }
}
function handleIncomingCall(call) {
  call.answer(localStream);
  handleCallStream(call);
}

function handleCallStream(call) {
  currentCalls[call.peer] = call;

  call.on("stream", (remoteStream) => {
    remoteStreams.value[call.peer] = remoteStream;
    document.getElementById(`remoteVideo-${call.peer}`).srcObject =
      remoteStream;
  });

  call.on("error", (err) => {
    handleError("Failed to join call: " + err.message);
  });

  call.on("close", () => {
    delete remoteStreams.value[call.peer];
  });
}
function handleError(message) {
  error.value = message;
}

function clearError() {
  error.value = "";
}

function handleSuccess(message) {
  console.log(message);
}
</script>

<style scoped>
.video-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-bottom: 20px;
}

.local-video-wrapper {
  position: relative;
  width: 400px;
  height: 300px;
  margin-bottom: 10px;
  border: 2px solid #007bff;
  border-radius: 8px;
}

.local-video {
  width: 100%;
  height: 100%;
  border-radius: 8px;
}

.remote-videos {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
}

.remote-video-wrapper {
  position: relative;
  width: 400px;
  height: 300px;
  margin: 10px;
  border: 2px solid #007bff;
  border-radius: 8px;
}

.remote-video {
  width: 100%;
  height: 100%;
  border-radius: 8px;
}

.user-id-overlay {
  position: absolute;
  bottom: 10px;
  left: 10px;
  color: white;
  background-color: rgba(0, 0, 0, 0.7);
  padding: 5px;
  border-radius: 5px;
}

.controls {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.room-input-container {
  display: flex;
  align-items: center;
}

.barcode-image {
  margin-left: 10px;
  height: 100px;
  width: auto;
}

.error-message {
  color: red;
  text-align: center;
}
</style>

