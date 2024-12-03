<template>
  <div class="order-list">
    <h2>Daftar Pesanan Tersimpan</h2>
    <table>
      <thead>
        <tr>
          <th>Nama</th>
          <th>Pesanan</th>
          <th>Total</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="order in savedOrders" :key="order.id">
          <td>{{ order.name }}</td>
          <td>
            <div v-for="item in order.items" :key="item.name">
              {{ item.name }} ({{ item.quantity }}x)
            </div>
          </td>
          <td>Rp {{ order.total_price }}</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
import axios from 'axios';
import { ref, onMounted } from 'vue';

export default {
  setup() {
    const savedOrders = ref([]);

    const fetchSavedOrders = async () => {
      try {
        const response = await axios.get("http://localhost:8081/orders");
        savedOrders.value = response.data;
      } catch (error) {
        console.error("Error fetching saved orders:", error);
        alert("Gagal memuat pesanan tersimpan");
      }
    };

    const refreshOrders = () => {
      fetchSavedOrders();
    };

    onMounted(fetchSavedOrders);

    return {
      savedOrders,
      refreshOrders
    };
  }
};
</script>

<style scoped>
.order-list {
  margin-top: 20px;
}
table {
  width: 100%;
  border-collapse: collapse;
}
table, th, td {
  border: 1px solid #ddd;
  padding: 8px;
}
</style>