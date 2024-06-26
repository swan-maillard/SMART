<script setup>
/* eslint-disable */
import { computed, ref, watchEffect } from "@vue/runtime-core";
import * as room1 from "../scenes/room1";
import * as room2 from "../scenes/room2";
import * as room3 from "../scenes/room3";
import useAuth from "../stores/auth.store";
import { storeToRefs } from "pinia";
import { useRouter } from "vue-router";
import useApi from "@/stores/api.store";
import usePopup from "@/stores/popup.store";

const router = useRouter();

const auth = useAuth();
const { user, game, elementGrabbed } = storeToRefs(auth);
const gameLoaded = ref(false);

useApi()
  .get("/game/salle")
  .then((res) => {
    if (res.status === 401) router.push("/");
    auth.user = res.data.user;
    auth.game = res.data.game;
  })
  .catch(console.log)
  .finally(() => (gameLoaded.value = true));

watchEffect(() => {
  // TODO : uncomment that
  if (gameLoaded.value && !game.value.hasStarted) router.push("/waiting");
}, [game]);

const bjsCanvas = ref(null);
const canvaMounted = ref(false);
const popup = usePopup();
let scene;

const room = ref();
watchEffect(() => {
  if (
    room.value ||
    !gameLoaded.value ||
    !user.value.salle ||
    !game.value.hasStarted
  )
    return;

  switch (user.value.salle) {
    case 1:
      room.value = room1;
      break;
    case 2:
      room.value = room2;
      break;
    case 3:
      room.value = room3;
      break;
  }
}, [user]);

const resizeObserver = new ResizeObserver((entries) => {
  for (let entry of entries) {
    const cr = entry.contentRect;
    entry.target.width = cr.width;
    entry.target.height = cr.height;
  }
});

watchEffect(() => {
  if (!canvaMounted.value && bjsCanvas.value && room.value) {
    canvaMounted.value = true;
    scene = room.value.createScene(bjsCanvas.value);
    resizeObserver.observe(bjsCanvas.value);
  }
}, [bjsCanvas, room]);

function imgDrop() {
  if (elementGrabbed.value && room.value) {
    room.value.verif(scene, elementGrabbed.value.id);
    elementGrabbed.value = null;
  } else {
    console.log("Aucun élément sélectionné");
  }
}

const isFinished = computed(() => game.value.isFinished || false);
const dateStart = computed(() => game.value.dateStart || null);
const chronoHours = ref("00");
const chronoMinutes = ref("00");
const chronoSeconds = ref("00");
let chronoInterval = null;

const updateElapsedTime = () => {
  if (!dateStart.value) return;

  const elapsedTime = Math.floor((Date.now() - dateStart.value) / 1000); // Elapsed time in seconds
  chronoHours.value = formatNumber(Math.floor(elapsedTime / 3600), 2); // Format hours with two digits
  chronoMinutes.value = formatNumber(Math.floor((elapsedTime % 3600) / 60), 2); // Format minutes with two digits
  chronoSeconds.value = formatNumber(Math.floor(elapsedTime % 60), 2); // Format minutes with two digits
};

const formatNumber = (number, width) => {
  return String(number).padStart(width, "0"); // Pad the number with zeros to ensure two digits
};

updateElapsedTime();
chronoInterval = setInterval(updateElapsedTime, 500);

watchEffect(() => {
  if (game.value.isFinished) {
    clearInterval(chronoInterval);
    useAuth().clearSession();
  }
}, [game]);
</script>

<template>
  <div v-if="gameLoaded && room" id="GameScreen" class="d-flex flex-raw">
    <div id="jeu" @drop="imgDrop" @dragover.prevent>
      <div
        v-if="!isFinished"
        class="popup d-flex justify-content-center align-items-center gap-4"
        :class="{
          open: popup.open,
          error: popup.type === 'error',
        }"
      >
        <img
          src="../assets/avatar_yeti.png"
          width="50"
          height="50"
          alt="yeti"
        />
        <span>{{ popup.text }}</span>
      </div>
      <div class="chrono" v-if="dateStart && !isFinished">
        {{ chronoHours !== "00" ? chronoHours + ":" : ""
        }}{{ chronoMinutes }}:{{ chronoSeconds }}
      </div>
      <div
        class="end d-flex justify-content-center align-items-center flex-column gap-4"
        v-if="isFinished"
      >
        <img
          src="../assets/avatar_yeti.png"
          width="100"
          height="100"
          alt="yeti"
        />
        <span
          ><b
            >Vous vous êtes échappés en
            {{ chronoHours !== "00" ? chronoHours + " heure(s) et" : "" }}
            {{ chronoMinutes }} minute(s) et {{ chronoSeconds }} seconde(s) !</b
          ></span
        >
        <span>Le Yeti ira se coucher le ventre vide, félicitations !</span>
      </div>
      <canvas id="GameCanva" ref="bjsCanvas" />
    </div>
  </div>
</template>

<style scoped>
#GameCanva {
  height: 100%;
  width: 100%;
  border-radius: 20px;
  box-shadow: 0 0 9px 1px #000000a6;
}
#GameScreen {
  width: 100%;
  height: 100%;
}
#jeu {
  width: 100%;
  height: 100%;
  position: relative;
  overflow: hidden;
  border-radius: 20px;
}

.chrono {
  position: absolute;
  bottom: 0;
  right: 0;
  font-size: 2em;
  padding: 10px 20px;
  background-color: rgba(34, 34, 34, 0.9);
  border-radius: 20px 0 20px 0;
}

.end {
  position: absolute;
  width: 100%;
  height: 100%;
  font-size: 2em;
  padding: 20px;
  background-color: rgba(34, 34, 34, 0.9);
  border-radius: 20px;
}

.end img {
  animation: joyAnimation 1.5s ease-in-out;
}

.popup {
  background-color: rgba(34, 34, 34, 0.9);
  position: absolute;
  padding: 20px 30px;
  border-radius: 20px 20px 0 0;
  width: 100%;
  transform: translateY(-100%);
  transition: 0.5s ease-in-out;
  color: #25e95f;
}

.popup.open {
  transform: none;
}

.popup span {
  text-align: center;
}

.popup.error {
  color: #ff3a3a;
}
@keyframes joyAnimation {
  0% {
    transform: translateY(0) rotate(0deg);
  }
  70% {
    transform: translateY(0) rotate(360deg);
  }
  85% {
    transform: translateY(-30px) rotate(360deg);
  }
  100% {
    transform: translateY(0) rotate(360deg);
  }
}
</style>
