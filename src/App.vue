<template>
  <div class="container">
    <div class="title-container">
      <img src="./assets/ChatGPT.svg" />
      <h1 class="title">Gepeto</h1>
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
    // Define variáveis reativas
    const username = ref(""); // Nome de usuário do usuário
    const message = ref(""); // Entrada de mensagem
    const messages = reactive([]); // Array para armazenar as mensagens
    const socket = ref(null); // Conexão de socket principal
    const backupSocket = ref(null); // Conexão de socket de backup
    const connectedUsers = reactive([]); // Array para armazenar os usuários conectados
    const isLoggedIn = ref(false); // Sinalizador para indicar o status de login do usuário
    const usingBackupServer = ref(false); // Sinalizador para indicar se o servidor de backup está sendo usado
    const serverSwitch = ref(true); // Sinalizador para gerenciar a troca de servidor

    // Executado quando o componente é montado
    onMounted(() => {
      // Conecta-se ao servidor de socket principal
      const mainSocket = io("https://flaskapichat.onrender.com/", {
        transports: ["websocket"],
        cors: {
          origin: "*",
        },
      });

      // Lida com erro de conexão
      mainSocket.on("connect_error", () => {
        if (serverSwitch.value) {
          // Alterna para o servidor de backup
          socket.value = backupSocket.value;
          usingBackupServer.value = true;
          serverSwitch.value = false;
          console.log("Usando servidor de backup. Tentando reconectar...");

          // Tenta voltar para o servidor principal após um delay
          setTimeout(() => {
            serverSwitch.value = true;
          }, 10000);
        }
      });

      // Lida com o evento "connected"
      mainSocket.on("connected", (data) => {
        console.log(data.data);
      });

      // Lida com o evento "user_entered"
      mainSocket.on("user_entered", (data) => {
        connectedUsers.splice(0, connectedUsers.length, ...data.users);
      });

      // Lida com o evento "thread_list"
      mainSocket.on("thread_list", (data) => {
        const threads = data.threads;
        console.log("Threads Ativas:", threads);
      });

      // Lida com o evento "thread_print"
      mainSocket.on("thread_print", (data) => {
        console.log("Impressão do Thread:", data);
      });

      // Define a conexão de socket principal
      socket.value = mainSocket;

      // Conecta-se ao servidor de socket de backup
      const backupSocketInstance = io(
        "https://flaskapichat-backup.onrender.com/",
        {
          transports: ["websocket"],
          cors: {
            origin: "*",
          },
        }
      );

      // Lida com o evento "connected" no servidor de backup
      backupSocketInstance.on("connected", (data) => {
        if (usingBackupServer.value) {
          console.log("Conectado ao servidor de backup.");
          console.log(data.data);
        }
      });

      // Lida com o evento "user_entered" no servidor de backup
      backupSocketInstance.on("user_entered", (data) => {
        connectedUsers.splice(0, connectedUsers.length, ...data.users);
      });

      // Lida com o evento "message" no servidor de backup
      backupSocketInstance.on("message", (data) => {
        messages.push(data);
      });

      // Lida com o evento "thread_list" no servidor de backup
      backupSocketInstance.on("thread_list", (data) => {
        const threads = data.threads;
        if (usingBackupServer.value) {
          console.log("Threads Ativas:", threads);
        }
      });

      // Lida com o evento "thread_print" no servidor de backup
      backupSocketInstance.on("thread_print", (data) => {
        if (usingBackupServer.value) {
          console.log("Impressão do Thread:", data);
        }
      });

      // Define a conexão de socket de backup
      backupSocket.value = backupSocketInstance;

      // Limpa as conexões de socket quando o componente é desmontado
      onBeforeUnmount(() => {
        mainSocket.disconnect();
        backupSocketInstance.disconnect();
      });
    });

    // Lida com a alteração de nome de usuário
    const handleUsernameChange = (event) => {
      username.value = event.target.value;
    };

    // Lida com a alteração de mensagem
    const handleMessageChange = (event) => {
      message.value = event.target.value;
    };

    // Lida com o envio de nome de usuário
    const handleUsernameSubmit = () => {
      isLoggedIn.value = true;
    };

    // Lida com o envio de mensagem
    const handleMessageSubmit = () => {
      if (message.value.trim() !== "") {
        socket.value.emit("message", {
          username: username.value,
          message: message.value,
        });
        message.value = "";
      }
    };

    // Lida com a ação de desconexão
    const handleDisconnect = () => {
      socket.value.disconnect();
      backupSocket.value.disconnect();
      username.value = "";
      connectedUsers.splice(0, connectedUsers.length);
      isLoggedIn.value = false;
    };

    // Executado quando o componente é montado
    onMounted(() => {
      if (socket.value) {
        // Lida com o evento "message" no servidor principal
        socket.value.on("message", (data) => {
          messages.push(data);
        });

        // Lida com o evento "user_entered" no servidor principal
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
  gap: 20px;
  justify-content: center;
  align-items: center;
  margin-bottom: 20px;
}

.title-container img {
  margin-bottom: 20px;
}

.title {
  font-size: 3rem;
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
  scroll-behavior: smooth;
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
  margin-bottom: 20px;
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

@media (max-width: 768px) {
  .login-container {
    width: 80vw;
  }
  .footer-container {
    width: 80vw;
  }
  .text-container input {
    width: 33vw;
  }
  .messages-container {
    padding: 8px;
    width: 100%;
    height: 50vh;
  }
  .message-card {
    padding: var(--default-padding);
    max-width: 100%;
    margin-bottom: 10px;
  }
}
</style>
