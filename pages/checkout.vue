<script setup>
import { ref, onMounted } from "vue";
import { useRouter } from "vue-router";
import { loadStripe } from "@stripe/stripe-js";
import { useCartStore } from "~/stores/cart";
import { useAuthStore } from "~/stores/auth.js";
import { useOrderStore } from "~/stores/order";

definePageMeta({
  middleware: "auth",
});

import { useNuxtApp } from "#app";

const { $toast } = useNuxtApp();

const cartStore = useCartStore();
const cartItems = cartStore.cart;
const total = cartStore.totalProductsPrice;
const isProcessing = ref(false);
const stripe = ref(null);
const elements = ref(null);
const card = ref(null);
const clientSecret = ref(null);
const router = useRouter();

const authStore = useAuthStore();

const orderStore = useOrderStore();

const dateTime = ref("");

onMounted(() => {
  const calculateDateTime = () => {
    const now = new Date();
    now.setHours(now.getHours() + 24);

    const year = now.getFullYear();
    const month = String(now.getMonth() + 1).padStart(2, "0"); // Months are zero-indexed
    const day = String(now.getDate()).padStart(2, "0");
    const hours = String(now.getHours()).padStart(2, "0");
    const minutes = String(now.getMinutes()).padStart(2, "0");

    const formattedDateTime = `${year}-${month}-${day} ${hours}:${minutes}`;
    dateTime.value = formattedDateTime;
  };

  calculateDateTime();
});

onMounted(async () => {
  const stripePublishableKey = useRuntimeConfig().public.stripePublishableKey;

  if (!stripePublishableKey) {
    console.error("Stripe publishable key is not defined.");
    return;
  }

  stripe.value = await loadStripe(stripePublishableKey);

  const { clientSecret: secret } = await $fetch("/api/stripe/paymentintent", {
    method: "POST",
    body: { amount: total },
  });
  clientSecret.value = secret;

  elements.value = stripe.value.elements();
  card.value = elements.value.create("card");
  card.value.mount("#card-element");
});

const pay = async () => {
  isProcessing.value = true;

  const result = await stripe.value.confirmCardPayment(clientSecret.value, {
    payment_method: {
      card: card.value,
    },
  });

  if (result.error) {
    document.querySelector("#card-error").textContent = result.error.message;
    isProcessing.value = false;
  } else {
    // if (!name.value || !address.value || !phoneNo.value) {
    //   return console.error("Please fill the form with your correct details");
    // }

    const newOrder = {
      status: "preparing order",
      hoursLeft: 24,
      deliveryDay: dateTime.value,
      products: [...cartStore.cart],
      name: authStore.fullName.value,
      address: authStore.address.value,
      phoneNumber: authStore.phoneNumber.value,
      // paymentIntentId: result.paymentIntent.id,
    };

    try {
      const orderId = await orderStore.createOrder(newOrder);

      if (orderId) {
        await orderStore.handleSetCurrentOrderId(orderId.id);
        authStore.fullName.value = "";
        authStore.address.value = "";
        authStore.phoneNumber.value = "";
      }
    } catch (error) {
      console.error("Failed to place the order");
      isProcessing.value = false;
    }

    isProcessing.value = false;
    cartStore.handleClearCart();
    router.push("/order");
    $toast.success("payment successfull: order successfully placed!");
  }
};
</script>

<template>
  <div
    id="CheckoutPage"
    class="container mx-auto p-4 text-white flex justify-center items-start lg:px-36 px-6"
  >
    <div
      class="my-10 flex justify-center items-center mx-auto sm:flex-nowrap lg:flex-nowrap gap-5 gap-x-20 flex-wrap"
    >
      <form class="flex flex-col items-start justify-end gap-y-6 w-full">
        <h4
          class="text-center mx-auto font-semibold text-2xl text-accent tracking-wide"
        >
          Ready to order? Let go!
        </h4>
        <span class="flex flex-col">
          <label class="font-normal text-lg text-accent tracking-wide">
            Full Name
          </label>
          <input
            v-model="name"
            class="w-full text-primary py-2 mt-2 outline-none px-5 rounded-lg"
            type="text"
          />
        </span>
        <span class="flex flex-col">
          <label class="font-normal text-lg text-accent tracking-wide">
            Phoner Number
          </label>
          <input
            v-model="phoneNo"
            class="w-auto text-primary py-2 mt-2 outline-none px-5 rounded-lg"
            type="number"
          />
        </span>
        <span class="flex flex-col">
          <label class="font-normal text-lg text-accent tracking-wide">
            Address
          </label>
          <input
            v-model="address"
            class="w-auto text-primary py-2 mt-2 outline-none px-5 rounded-lg"
            type="text"
          />
        </span>
      </form>

      <div>
        <div v-if="cartItems.length === 0">
          <p>Your cart is empty.</p>
          <NuxtLink to="/" class="text-blue-500">Go to Shop</NuxtLink>
        </div>
        <div v-else>
          <div class="mb-4">
            <h2 class="text-2xl font-semibold my-4">Cart Items</h2>

            <ul class="divide-y divide-yellow-500 border-b border-t px-5">
              <li
                v-for="item in cartItems"
                :key="item.id"
                class="py-3 space-y-1"
              >
                <div class="flex text-sm items-center justify-between gap-4">
                  <p>
                    <span class="font-bold">{{ item.quantity }}&times;</span
                    >{{ item.name }}
                  </p>
                  <p class="font-bold">
                    ${{ item.quantity * item.unitPrice }}.00
                  </p>
                </div>
              </li>
            </ul>
          </div>
          <div class="mb-4">
            <h2 class="text-xl font-semibold">Total: ${{ total }}</h2>
          </div>
          <form @submit.prevent="pay">
            <div
              id="card-element"
              class="mb-4 bg-white text-primary border-none p-4 mt-10 font-bold"
            ></div>
            <p id="card-error" class="text-red-500"></p>
            <button
              type="submit"
              class="text-primary bg-accent hover:bg-accent-hover px-5 py-3 font-semibold rounded-lg mt-3 flex"
              :disabled="isProcessing"
            >
              <span v-if="isProcessing">Processing...</span>
              <span v-else>Place Order</span>
            </button>
          </form>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
#card-element {
  border: 1px solid #ccc;
  padding: 10px;
  border-radius: 5px;
}
</style>
