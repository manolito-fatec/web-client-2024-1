<template>
  <div class="history-wrapper">
    <h2 class="history-title">Histórico:</h2>
    <div class="history-container">
      <textarea v-model="historyText" readonly></textarea>
    </div>
  </div>
</template>

<script setup>
import {ref} from 'vue'

const historyText = ref('')
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Poppins:ital,wght@0,100;0,200;0,300;0,400;0,500;0,600;0,700;0,800;0,900;1,100;1,200;1,300;1,400;1,500;1,600;1,700;1,800;1,900&display=swap');


.history-wrapper {
  margin-top: 16px;
}

.history-title {
  color: #ffffff;
  margin-bottom: 8px;
  font-size: 16px;
  font-family: Arial, sans-serif;
}

.history-container {
  padding: 0;
  border-radius: 8px;
  box-sizing: border-box;
  background-color: #3D3D3D;
  display: flex;
  height: 400px;
}

.history-container textarea {
  width: 100%;
  height: 100%;
  border: none;
  border-radius: 8px;
  background-color: transparent;
  color: #ffffff;
  padding: 16px;
  font-size: 16px;
  font-family: 'Poppins', regular, sans-serif;
  resize: none;
  box-sizing: border-box;
  cursor: default;
}
</style>
