<template>
  <div
    class="min-h-screen bg-gray-50 dark:bg-zinc-900 text-gray-900 dark:text-white p-6"
  >
    <div class="max-w-xl mx-auto">
      <h1 class="text-3xl font-bold mb-6 text-center">
        🍎 Калькулятор Калорій
      </h1>

      <form
        @submit.prevent="addProduct"
        class="grid gap-4 mb-8 bg-white dark:bg-zinc-800 p-6 rounded-2xl shadow-xl"
      >
        <input
          v-model="product.name"
          type="text"
          placeholder="Назва продукту"
          class="input"
        />
        <input
          v-model.number="product.grams"
          type="number"
          placeholder="Кількість грамів"
          class="input"
        />
        <button type="submit" class="btn" :disabled="loading">
          {{ loading ? "⏳ Завантаження..." : "➕ Додати продукт" }}
        </button>
        <p v-if="error" class="text-red-500 text-sm">{{ error }}</p>
        <p v-if="translatedText" class="text-xs text-gray-500">
          Перекладено: {{ translatedText }}
        </p>
      </form>

      <ul class="space-y-4 mb-6">
        <li
          v-for="(item, index) in products"
          :key="index"
          class="flex justify-between items-center bg-white dark:bg-zinc-800 p-4 rounded-xl shadow"
        >
          <div>
            <p class="font-semibold">{{ item.name }}</p>
            <p class="text-sm text-gray-500 dark:text-gray-400">
              {{ item.grams }}г = {{ calculateItemCalories(item) }} ккал
              <span class="text-xs">({{ item.calories }} ккал/100г)</span>
            </p>
          </div>
          <button
            @click="removeProduct(index)"
            class="text-red-500 hover:underline transition"
          >
            🗑️ Видалити
          </button>
        </li>
      </ul>

      <div
        class="bg-white dark:bg-zinc-800 p-6 rounded-2xl shadow-xl text-center"
      >
        <p class="text-xl font-semibold">🔢 Загальна калорійність:</p>
        <p class="text-3xl font-bold mt-2 text-emerald-500">
          {{ totalCalories }} ккал
        </p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from "vue";

const API_KEY = "ТВОЙ_API_КЛЮЧ";

const product = ref({ name: "", grams: 0 });
const products = ref([]);
const loading = ref(false);
const error = ref(null);
const translatedText = ref("");

async function translateToEnglish(text) {
  try {
    const response = await fetch("https://libretranslate.com/translate", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        q: text,
        source: "auto",
        target: "en",
      }),
    });

    if (!response.ok) {
      throw new Error("Помилка перекладу");
    }

    const data = await response.json();
    translatedText.value = data.translatedText;
    return data.translatedText;
  } catch (err) {
    console.error("Помилка перекладу:", err);
    return text;
  }
}

async function getCaloriesFromAPI(productName) {
  const translatedProduct = await translateToEnglish(productName);

  const url = `https://api.calorieninjas.com/v1/nutrition?query=${encodeURIComponent(
    translatedProduct
  )}`;

  try {
    const response = await fetch(url, { headers: { "X-Api-Key": API_KEY } });
    if (!response.ok) {
      throw new Error("Помилка запиту до API");
    }

    const data = await response.json();
    if (data.items.length === 0) {
      throw new Error("Продукт не знайдено в базі даних");
    }

    return data.items[0].calories;
  } catch (error) {
    console.error("Помилка отримання даних:", error.message);
    throw error;
  }
}

const addProduct = async () => {
  if (!product.value.name || product.value.grams <= 0) {
    error.value = "Будь ласка, заповніть всі поля";
    return;
  }

  error.value = null;
  loading.value = true;
  translatedText.value = "";

  try {
    const calories = await getCaloriesFromAPI(product.value.name);
    products.value.push({
      name: product.value.name,
      calories,
      grams: product.value.grams,
      translated: translatedText.value,
    });
    product.value = { name: "", grams: 0 };
  } catch (err) {
    error.value = `Помилка: ${err.message}`;
  } finally {
    loading.value = false;
  }
};

const removeProduct = (index) => {
  products.value.splice(index, 1);
};

const calculateItemCalories = (item) => {
  return ((item.calories / 100) * item.grams).toFixed(2);
};

const totalCalories = computed(() => {
  return products.value
    .reduce((sum, item) => {
      return sum + (item.calories / 100) * item.grams;
    }, 0)
    .toFixed(2);
});
</script>
<style scoped>
.input {
  width: 100%;
  border-radius: 0.75rem;
  border: 1px solid #d1d5db;
  padding: 0.75rem;
  background-color: white;
  color: black;
  transition: all 0.2s;
}

.dark .input {
  border-color: #3f3f46;
  background-color: #18181b;
  color: white;
}

.input::placeholder {
  color: #9ca3af;
}

.input:focus {
  outline: none;
  box-shadow: 0 0 0 2px #10b981;
}

.btn {
  width: 100%;
  background-color: #10b981;
  color: white;
  font-weight: 600;
  padding: 0.75rem 1rem;
  border-radius: 0.75rem;
  transition: all 0.2s;
}

.btn:hover:not(:disabled) {
  background-color: #059669;
}

.btn:disabled {
  background-color: #94a3b8;
  cursor: not-allowed;
}
</style>
