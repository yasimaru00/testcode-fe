<template>
  <div class="login-container">
    <form @submit.prevent="handleLogin" class="login-form">
      <h2>Login Sistem Pemesanan</h2>
      <div class="form-group">
        <label>Username:</label>
        <input
          type="text"
          v-model="username"
          required
          placeholder="Masukkan username"
        />
      </div>
      <div class="form-group">
        <label>Password:</label>
        <input
          type="password"
          v-model="password"
          required
          placeholder="Masukkan password"
        />
      </div>
      <button type="submit" :disabled="isLoading">
        {{ isLoading ? "Logging in..." : "Login" }}
      </button>
      <p v-if="errorMessage" class="error-message">
        {{ errorMessage }}
      </p>
    </form>
  </div>
</template>
  
  <script>
import axios from "axios";
import { ref } from "vue";

export default {
  setup(props, { emit }) {
    const username = ref("");
    const password = ref("");
    const isLoading = ref(false);
    const errorMessage = ref("");

    const handleLogin = async () => {
      isLoading.value = true;
      errorMessage.value = "";

      try {
        const response = await axios.post("http://localhost:8081/login", {
          username: username.value,
          password: password.value,
        });

        // Simpan token dan informasi user di localStorage
        localStorage.setItem("user", JSON.stringify(response.data.user));
        localStorage.setItem("token", response.data.token);

        // Emit event login berhasil
        emit("login-success", response.data.user);
      } catch (error) {
        errorMessage.value = error.response?.data?.message || "Login gagal";
        console.error("Login error:", error);
      } finally {
        isLoading.value = false;
      }
    };

    return {
      username,
      password,
      isLoading,
      errorMessage,
      handleLogin,
    };
  },
};
</script>
  
<style scoped>
.login-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background: linear-gradient(to right, #ffecd2, #fcb69f);
  /* Add padding to ensure proper centering */
  padding: 20px;
  box-sizing: border-box;
}

.login-form {
  background-color: white;
  padding: 40px;
  border-radius: 10px;
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1);
  width: 400px;
  /* Ensure form stays centered on smaller screens */
  max-width: 100%;
}

.form-group {
  margin-bottom: 20px;
}

input {
  width: 100%;
  padding: 12px;
  margin-top: 5px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 16px;
}

button {
  width: 100%;
  padding: 12px;
  background-color: #4caf50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.error-message {
  color: red;
  text-align: center;
  margin-top: 10px;
}

h2 {
  font-family: 'Arial', sans-serif;
  text-align: center;
  margin-bottom: 20px;
  color: #333;
}

label {
  font-family: 'Arial', sans-serif;
  color: #333;
}
</style>

