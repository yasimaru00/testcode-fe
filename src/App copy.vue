<template>
  <div class="container">
    <h1>Sistem Pemesanan Makanan Kelompok</h1>
    <div class="order-form">
      <h2>Form Pemesanan Kelompok</h2>
      <form @submit.prevent="addToGroupOrder">
        <div class="form-group">
          <label>Pilih Pengguna:</label>
          <select v-model="currentUser" required>
            <option v-for="user in users" :key="user.id" :value="user">
              {{ user.name }}
            </option>
          </select>
        </div>
        <div class="products">
          <h3>Pilih Produk:</h3>
          <div
            v-for="product in products"
            :key="product.id"
            class="product-item"
          >
            <label>
              <input
                type="checkbox"
                :value="product.id"
                v-model="currentUserProducts"
                @change="updateProductQuantity(product.id)"
              />
              {{ product.name }} - Rp {{ product.price }}
              <input
                v-if="currentUserProducts.includes(product.id)"
                type="number"
                v-model.number="currentProductQuantities[product.id]"
                min="1"
                default="1"
                class="quantity-input"
              />
            </label>
          </div>
        </div>
        <button
          type="button"
          @click="addToGroupOrder"
          :disabled="!currentUser || currentUserProducts.length === 0"
        >
          Tambah ke Pesanan Kelompok
        </button>
      </form>
      <div class="group-order-list" v-if="groupOrders.length">
        <h3>Pesanan Kelompok Saat Ini:</h3>
        <table>
          <thead>
            <tr>
              <th>Nama</th>
              <th>Pesanan</th>
              <th>Aksi</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(order, index) in groupOrders" :key="index">
              <td>{{ order.user.name }}</td>
              <td>
                {{
                  order.items
                    .map((item) => {
                      const product = products.find(
                        (p) => p.id === item.product_id
                      );
                      return `${product.name} (${item.quantity}x)`;
                    })
                    .join(", ")
                }}
              </td>
              <td>
                <button @click="removeFromGroupOrder(index)">Hapus</button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
      <div class="group-order-summary" v-if="groupOrders.length">
        <h3>Ringkasan Pesanan Kelompok:</h3>
        <div v-for="(cost, user) in individualCosts" :key="user">
          {{ user }}: Rp {{ cost.individualCost }}
        </div>
        <p>
          <strong>Total Kelompok: Rp {{ totalGroupCost }}</strong>
        </p>
        <button @click="submitGroupOrder">Simpan Pesanan Kelompok</button>
      </div>
    </div>
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
  </div>
</template>
<script>
import axios from "axios";
import { ref, onMounted, computed } from "vue";
export default {
  setup() {
    const users = ref([]);
    const products = ref([]);
    const promotions = ref([
      { min_purchase: 40000, discount_percentage: 30, max_discount: 30000 },
    ]);
    const savedOrders = ref([]);
    const currentUser = ref(null);
    const currentUserProducts = ref([]);
    const currentProductQuantities = ref({});
    const groupOrders = ref([]);
    const fetchInitialData = async () => {
      try {
        const [usersResponse, productsResponse, ordersResponse] =
          await Promise.all([
            axios.get("http://localhost:8081/users"),
            axios.get("http://localhost:8081/products"),
            axios.get("http://localhost:8081/orders"),
          ]);
        users.value = usersResponse.data;
        products.value = productsResponse.data;
        savedOrders.value = ordersResponse.data;
        console.log(
          "Fetched Orders:",
          JSON.stringify(savedOrders.value, null, 2)
        ); // Log lebih detail
      } catch (error) {
        console.error("Error fetching initial data:", error);
        alert("Gagal memuat data. Silakan coba lagi.");
      }
    };

    const addToGroupOrder = () => {
      if (!currentUser.value) return;
      const existingOrderIndex = groupOrders.value.findIndex(
        (order) => order.user.id === currentUser.value.id
      );
      const orderItems = currentUserProducts.value.map((productId) => ({
        product_id: productId,
        quantity: currentProductQuantities.value[productId] || 1,
      }));
      if (existingOrderIndex !== -1) {
        groupOrders.value[existingOrderIndex].items = orderItems;
      } else {
        groupOrders.value.push({
          user: currentUser.value,
          items: orderItems,
        });
      }
      resetCurrentUserSelection();
    };

    const resetCurrentUserSelection = () => {
      currentUser.value = null;
      currentUserProducts.value = [];
      currentProductQuantities.value = {};
    };

    const removeFromGroupOrder = (index) => {
      groupOrders.value.splice(index, 1);
    };

    const updateProductQuantity = (productId) => {
      if (!currentUserProducts.value.includes(productId)) {
        delete currentProductQuantities.value[productId];
      } else if (!currentProductQuantities.value[productId]) {
        currentProductQuantities.value[productId] = 1;
      }
    };

    const groupSubtotal = computed(() => {
      return groupOrders.value.reduce((total, order) => {
        return (
          total +
          order.items.reduce((orderTotal, item) => {
            const product = products.value.find(
              (p) => p.id === item.product_id
            );
            return orderTotal + (product ? product.price * item.quantity : 0);
          }, 0)
        );
      }, 0);
    });

    const promoDiscount = computed(() => {
      const activePromo = promotions.value[0];

      if (groupSubtotal.value >= activePromo.min_purchase) {
        return Math.min(
          Math.floor(
            groupSubtotal.value * (activePromo.discount_percentage / 100)
          ),
          activePromo.max_discount
        );
      }

      return 0;
    });

    const shippingCost = 5000;

    const individualCosts = computed(() => {
      const costs = {};
      if (groupOrders.value.length === 0) return costs;

      const totalSubtotal = groupOrders.value.reduce((total, order) => {
        return (
          total +
          order.items.reduce((sum, item) => {
            const product = products.value.find(
              (p) => p.id === item.product_id
            );
            return sum + (product ? product.price * item.quantity : 0);
          }, 0)
        );
      }, 0);

      const totalBeforePromo = totalSubtotal + shippingCost;
      const activePromo = promotions.value[0];
      const promoDiscount =
        totalSubtotal >= activePromo.min_purchase
          ? Math.min(
              Math.floor(
                totalSubtotal * (activePromo.discount_percentage / 100)
              ),
              activePromo.max_discount
            )
          : 0;

      const individualProportions = {};

      groupOrders.value.forEach((order) => {
        const userSubtotal = order.items.reduce((subtotal, item) => {
          const product = products.value.find((p) => p.id === item.product_id);
          return subtotal + (product ? product.price * item.quantity : 0);
        }, 0);
        individualProportions[order.user.name] = userSubtotal / totalSubtotal;
      });

      const equalShippingCost = shippingCost / groupOrders.value.length;

      groupOrders.value.forEach((order) => {
        const username = order.user.name;
        const userSubtotal = order.items.reduce((subtotal, item) => {
          const product = products.value.find((p) => p.id === item.product_id);
          return subtotal + (product ? product.price * item.quantity : 0);
        }, 0);
        const individualPromoDiscount = Math.round(
          promoDiscount * individualProportions[username]
        );
        const individualShipping = equalShippingCost;
        const individualCost = Math.round(
          userSubtotal - individualPromoDiscount + individualShipping
        );
        costs[username] = {
          subtotal: userSubtotal,
          promoDiscount: individualPromoDiscount,
          shipping: individualShipping,
          individualCost: individualCost,
        };
      });

      return costs;
    });

    const totalGroupCost = computed(() => {
      const totalSubtotal = groupOrders.value.reduce((total, order) => {
        return (
          total +
          order.items.reduce((sum, item) => {
            const product = products.value.find(
              (p) => p.id === item.product_id
            );
            return sum + (product ? product.price * item.quantity : 0);
          }, 0)
        );
      }, 0);
      const promoDiscount = Math.min(Math.floor(totalSubtotal * 0.3), 30000);
      return totalSubtotal + shippingCost - promoDiscount;
    });

    const submitGroupOrder = async () => {
      console.log("Group Orders: ", groupOrders.value);
      console.log("Individual Costs: ", individualCosts.value);
      try {
        for (const [username, details] of Object.entries(
          individualCosts.value
        )) {
          const userOrder = groupOrders.value.find(
            (order) => order.user.name === username
          );
          if (userOrder && userOrder.user && userOrder.user.name) {
            // Penambahan pengecekan untuk memastikan userOrder tidak undefined
            const orderData = {
              user_id: userOrder.user.id,
              total_price: details.individualCost,
              order_details: userOrder.items.map((item) => {
                const product = products.value.find(
                  (p) => p.id === item.product_id
                );
                return {
                  product_id: item.product_id,
                  quantity: item.quantity,
                  price: product.price,
                  subtotal: item.quantity * product.price,
                };
              }),
            };
            console.log("Order Data: ", orderData); // Debug log untuk memeriksa order data
            await axios.post("http://localhost:8081/orders", orderData);
          } else {
            console.error("User order is undefined for username: ", username);
          }
        }
        savedOrders.value = await axios
          .get("http://localhost:8081/orders")
          .then((res) => res.data);
        groupOrders.value = [];
        alert("Pesanan kelompok berhasil disimpan!");
      } catch (error) {
        console.error("Error menyimpan pesanan:", error);
        alert("Gagal menyimpan pesanan kelompok");
      }
    };

    // onMounted(fetchInitialData);
    onMounted(async () => {
      await fetchInitialData();
      console.log("Saved Orders:", JSON.stringify(savedOrders.value, null, 2)); // Log data dengan lebih detail
    });

    return {
      users,
      products,
      promotions,
      currentUser,
      currentUserProducts,
      currentProductQuantities,
      groupOrders,
      savedOrders,
      addToGroupOrder,
      removeFromGroupOrder,
      updateProductQuantity,
      individualCosts,
      totalGroupCost,
      submitGroupOrder,
    };
  },
};
</script>

<style scoped>
/* Style CSS tetap sama seperti sebelumnya */
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}
.form-group,
.products {
  margin-bottom: 20px;
}
.product-item {
  display: flex;
  align-items: center;
  margin-bottom: 10px;
}
.quantity-input {
  margin-left: 10px;
  width: 60px;
}
table {
  width: 100%;
  border-collapse: collapse;
}
table,
th,
td {
  border: 1px solid #ddd;
  padding: 8px;
}
button {
  margin: 10px 0;
  padding: 10px;
  background-color: #4caf50;
  color: white;
  border: none;
  cursor: pointer;
}
button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}
</style>