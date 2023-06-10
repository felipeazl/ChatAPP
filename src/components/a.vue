<template>
    <div class="container">
      <h1 class="title">VlaskChat</h1>
      <div v-if="!isLoggedIn">
        <form @submit.prevent="handleUsernameSubmit">
          <div class="login-container">
            <label for="username">Nome de usuário:</label>
            <input
              type="text"
              id="username"
              v-model="username"
              placeholder="Digite seu nome"
            />
          </div>
          <button type="submit" class="login-btn">Entrar</button>
        </form>
      </div>
  
      <div v-if="isLoggedIn" class="chat-container">
        <div class="chat-header">
          <p>
            Conectado como: <span>{{ username }}</span>
          </p>
          <div class="num-users-container">
            <div class="on-indicator"/>
            <p v-if="numUsers > 0" class="num-users">{{ numUsers }}</p>
          </div>
          <button @click="handleDisconnect">Sair</button>
        </div>
        <div class="messages-container">
          <div
            v-for="(message, index) in messages"
            :key="index"
            class="message-card"
          >
            <strong>{{ message.username }}:</strong> {{ message.message }}
          </div>
        </div>
  
        <div class="footer-container">
          <form @submit.prevent="handleMessageSubmit">
            <div class="text-container">
              <input type="text" id="message" v-model="message" />
              <button type="submit">Enviar</button>
            </div>
          </form>
        </div>
      </div>
  
      <div v-if="numUsers > 0" class="num-users">
        <h2>Usuários conectados: {{ numUsers }}</h2>
      </div>
    </div>
  </template>
  
  <script>
  import { ref, reactive, onMounted, onBeforeUnmount } from "vue";
  import io from "socket.io-client";
  
  export default {
    name: "App",
    setup() {
      const username = ref("");
      const message = ref("");
      const messages = reactive([]);
      const socket = ref(null);
      const numUsers = ref(0);
      const isLoggedIn = ref(false);
  
      onMounted(() => {
        const newSocket = io("http://localhost:5000");
  
        newSocket.on("connected", (data) => {
          console.log(data.data);
        });
  
        newSocket.on("user_entered", (data) => {
          console.log("Threads:", data.threads); // Mostra as threads no console
          numUsers.value = data.num_users;
        });
  
        socket.value = newSocket;
  
        onBeforeUnmount(() => {
          newSocket.disconnect();
        });
      });
  
      const handleUsernameSubmit = () => {
        isLoggedIn.value = true;
      };
  
      const handleMessageSubmit = () => {
        if (message.value.trim() !== "") {
          socket.value.emit("message", {
            username: username.value,
            message: message.value,
          });
          message.value = "";
        }
      };
  
      const handleDisconnect = () => {
        socket.value.disconnect();
        username.value = "";
        numUsers.value = 0;
        isLoggedIn.value = false;
      };
  
      onMounted(() => {
        if (socket.value) {
          socket.value.on("message", (data) => {
            messages.push(data);
          });
  
          socket.value.on("user_entered", (data) => {
            console.log("Threads:", data.threads); // Mostra as threads no console
            numUsers.value = data.num_users;
          });
        }
      });
  
      return {
        username,
        message,
        messages,
        numUsers,
        isLoggedIn,
        handleUsernameSubmit,
        handleMessageSubmit,
        handleDisconnect,
      };
    },
  };
  </script>
  
  <style scoped>
  /* Estilos CSS do componente */
  </style>
  