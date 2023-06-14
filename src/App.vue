<template>
  <div class="container">
    <div class="title-container">
      <img src="./assets/chats.svg" />
      <h1 class="title">VlaskChat</h1>
    </div>
    <!-- LoginForm -->
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
    <!-- Chat-Header -->
    <div v-if="isLoggedIn" class="chat-container">
      <div class="chat-header">
        <p>
          Conectado como: <span>{{ username }}</span>
        </p>
      </div>
      <div class="messages-container">
        <!-- Chat-Message -->
        <div
          v-for="(message, index) in messages"
          :key="index"
          class="message-card"
        >
          <strong>{{ message.username }}:</strong> {{ message.message }}
        </div>
      </div>
      <!-- Chat-Form -->
      <div class="footer-container">
        <button @click="handleDisconnect" class="exit-button">
          <img src="./assets/sign-out.svg" />
        </button>
        <form @submit.prevent="handleMessageSubmit">
          <div class="text-container">
            <input type="text" id="message" v-model="message" />
            <button type="submit" class="send-button">
              <img src="./assets/paper-plane-right.svg" />
            </button>
          </div>
        </form>
      </div>
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
    const connectedUsers = reactive([]);
    const isLoggedIn = ref(false);
    onMounted(() => {
      // "https://flaskapichat.onrender.com/"
      const newSocket = io("https://flaskapichat.onrender.com/", {
        transports: ["websocket"],
        cors: {
          origin: "*",
        },
      });
      newSocket.on("connected", (data) => {
        console.log(data.data);
      });
      newSocket.on("user_entered", (data) => {
        connectedUsers.splice(0, connectedUsers.length, ...data.users);
      });
      newSocket.on("thread_list", (data) => {
        const threads = data.threads;
        console.log("Active Threads:", threads);
      });
      newSocket.on("thread_print", (data) => {
        console.log("Thread Print:", data);
      });
      socket.value = newSocket;
      onBeforeUnmount(() => {
        newSocket.disconnect();
      });
    });
    const handleUsernameChange = (event) => {
      username.value = event.target.value;
    };
    const handleMessageChange = (event) => {
      message.value = event.target.value;
    };
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
      connectedUsers.splice(0, connectedUsers.length);
      isLoggedIn.value = false;
    };
    onMounted(() => {
      if (socket.value) {
        socket.value.on("message", (data) => {
          messages.push(data);
        });
        socket.value.on("user_entered", (data) => {
          connectedUsers.splice(0, connectedUsers.length, ...data.users);
        });
      }
    });
    return {
      username,
      message,
      messages,
      isLoggedIn,
      connectedUsers,
      handleUsernameChange,
      handleMessageChange,
      handleUsernameSubmit,
      handleMessageSubmit,
      handleDisconnect,
    };
  },
};
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  background-color: var(--background);
  min-width: 100vw;
  min-height: 100vh;
  color: #fff;
}

.title-container {
  display: flex;
  gap: 16px;
  justify-content: center;
  align-items: center;
}

.title-container img {
  margin-bottom: 20px;
}

.title {
  font-size: 2rem;
  font-weight: bold;
  padding-bottom: 16px;
}

.login-container {
  display: flex;
  flex-direction: column;
  width: 400px;
}

.login-container label {
  font-size: 1rem;
  font-weight: 400;
  margin-bottom: 8px;
}

.login-container input {
  padding: var(--default-padding);
  background-color: var(--grey);
  border: none;
  outline: var(--default-outline);
  border-radius: var(--radius-md);
  margin-bottom: 16px;
  color: white;
  font-size: 1rem;
}

.login-container input:focus {
  outline: var(--default-outline);
}

.login-btn {
  width: 100%;
  padding: var(--default-padding);
  border-radius: var(--radius-md);
  background-color: var(--violet);
  border: none;
  font-size: 1rem;
  font-weight: bold;
  color: #fff;
  cursor: pointer;
}

.login-btn:hover {
  opacity: 0.9;
}

.chat-container {
  display: flex;
  flex-direction: column;
  width: 80vw;
  border-radius: var(--radius-lg);
}

.chat-header {
  display: flex;
  width: 100%;
  justify-content: space-between;
  align-items: center;
}

.chat-header p {
  font-size: 1rem;
  font-weight: 400;
}

.chat-header span {
  font-size: 1rem;
  font-weight: bold;
}
.messages-container {
  margin: 20px 0px;
  padding: var(--default-padding);
  overflow-y: auto;
  width: 100%;
  height: 50vh;
}

.message-card {
  margin-bottom: 20px;
  background-color: var(--grey);
  padding: var(--default-padding);
  font-size: 1rem;
  font-weight: 400;
  max-width: 50%;
  overflow-wrap: break-word;
  border-radius: var(--radius-md);
}

.footer-container {
  display: flex;
  width: 100%;
  gap: 16px;
  justify-content: center;
}

.text-container {
  display: flex;
  gap: 16px;
}

.text-container input {
  padding: var(--default-padding);
  background-color: var(--grey);
  border: none;
  outline: var(--default-outline);
  border-radius: var(--radius-md);
  color: #fff;
  font-size: 1rem;
  width: 400px;
}

.text-container input:focus {
  outline: 2px solid var(--violet);
}
.send-button {
  padding: var(--default-padding);
  border-radius: var(--radius-md);
  font-size: 1rem;
  font-weight: 500;
  background-color: var(--violet);
  color: #fff;
  border: none;
  cursor: pointer;
}

.send-button:hover {
  opacity: 0.9;
}

.exit-button {
  padding: var(--default-padding);
  border-radius: var(--radius-md);
  font-size: 1rem;
  font-weight: 500;
  background-color: #b91c1c;
  color: #fff;
  border: none;
  cursor: pointer;
}

.exit-button:hover {
  opacity: 0.9;
}

.img-button {
  color: white;
}
</style>
