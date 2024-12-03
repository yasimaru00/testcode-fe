<template>
  <div v-if="!isLoggedIn" class="login-wrapper">
    <Login @login-success="handleLoginSuccess" />
  </div>
  <div v-else class="container">
    <h1>Sistem Pemesanan Makanan Kelompok</h1>
    <div class="user-info">
      <span>Selamat datang, {{ currentUser.name }}</span>
      <button @click="handleLogout" class="logout-btn">Logout</button>
    </div>
    <GroupOrderForm @order-submitted="refreshSavedOrders" />
    <SavedOrderList ref="savedOrderListRef" />
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';
import Login from './components/Login.vue';
import GroupOrderForm from './components/OrderForm.vue';
import SavedOrderList from './components/OrderList.vue';

export default {
  components: {
    Login,
    GroupOrderForm,
    SavedOrderList
  },
  setup() {
    const isLoggedIn = ref(false);
    const currentUser = ref(null);
    const savedOrderListRef = ref(null);

    const checkLoginStatus = () => {
      const storedUser = localStorage.getItem('user');
      const token = localStorage.getItem('token');
      
      if (storedUser && token) {
        currentUser.value = JSON.parse(storedUser);
        isLoggedIn.value = true;
      }
    };

    const handleLoginSuccess = (user) => {
      currentUser.value = user;
      isLoggedIn.value = true;
    };

    const handleLogout = () => {
      localStorage.removeItem('user');
      localStorage.removeItem('token');
      isLoggedIn.value = false;
      currentUser.value = null;
    };

    const refreshSavedOrders = () => {
      if (savedOrderListRef.value) {
        savedOrderListRef.value.refreshOrders();
      }
    };

    onMounted(checkLoginStatus);

    return {
      isLoggedIn,
      currentUser,
      savedOrderListRef,
      handleLoginSuccess,
      handleLogout,
      refreshSavedOrders
    };
  }
};
</script>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.user-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.logout-btn {
  background-color: #f44336;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
}
</style>