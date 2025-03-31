<template>
  <div class="container">
    <div v-if="isLoading" class="loading-screen">
      <img src="/image/ball.png" alt="載入中" class="loading-image">
    </div>
    <!-- 圖鑑標題（圖片） -->
    <div class="text-center my-4">
      <img src="/image/pokemon.png" alt="寶可夢圖鑑" class="img-fluid" style="width: 40%; height: auto;" />
    </div>

    <!-- 搜尋欄位 + 類型過濾器 -->
    <div class="mb-3">
      <div class="input-group">
        <input v-model="searchQuery" type="text" class="form-control w-75" placeholder="搜尋寶可夢（名稱、編號）" />
        <select v-model="typeFilter" class="form-select">
          <option value="">所有類型</option>
          <option v-for="(label, type) in typeTranslations" :key="type" :value="type">
            {{ label }} ({{ typeCounts[type] || 0 }})
          </option>
        </select>
      </div>
    </div>

    <!-- 使用 Bootstrap 的 row 來建立網格佈局 -->
    <div class="row">
      <!-- 透過 v-for 迴圈遍歷過濾後的 pokemons 陣列，每個寶可夢顯示為一張卡片 -->
      <div v-for="pokemon in filteredPokemons" :key="pokemon.id" class="col-xl-2 col-lg-3 col-md-4 col-6 mb-3">
        <div class="card" @click="openModal(pokemon)">
          <!-- 顯示寶可夢圖片 -->
          <img :src="pokemon.image" class="card-img-top" :alt="pokemon.name">
          <div class="card-body">
            <h5 class="card-title">{{ "#" + pokemon.id }}</h5>
          </div>
        </div>
      </div>
    </div>

    <!-- Bootstrap Modal，顯示選中的寶可夢詳細資料 -->
    <div v-if="selectedPokemon" class="modal fade show d-block" tabindex="-1">
      <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">{{ selectedPokemon.name }}</h5>
            <button type="button" class="btn-close" @click="closeModal"></button>
          </div>
          <div class="modal-body text-center">
            <img :src="selectedPokemon.image" class="img-fluid w-50 mb-3" :alt="selectedPokemon.name">
            <p><strong>編號：</strong> {{ selectedPokemon.id }}</p>
            <p><strong>屬型：</strong> {{ selectedPokemon.types }}</p>
            <p><strong>身高：</strong> {{ selectedPokemon.height }} m</p>
            <p><strong>體重：</strong> {{ selectedPokemon.weight }} kg</p>
            <p><strong>地區：</strong> {{ selectedPokemon.regionalDex }}</p>
            <p><strong>描述：</strong> {{ selectedPokemon.description }}</p>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from "vue";
import axios from "axios";

// 是否顯示加載動畫
const isLoading = ref(true);

const pokemons = ref([]);
const selectedPokemon = ref(null);
const searchQuery = ref("");
const typeFilter = ref("");
const typeCounts = ref({});

// 屬性類型翻譯
const typeTranslations = {
  "normal": "一般",
  "fire": "火",
  "water": "水",
  "electric": "電",
  "grass": "草",
  "ice": "冰",
  "fighting": "格鬥",
  "poison": "毒",
  "ground": "地面",
  "flying": "飛行",
  "psychic": "超能力",
  "bug": "蟲",
  "rock": "岩石",
  "ghost": "幽靈",
  "dragon": "龍",
  "dark": "惡",
  "steel": "鋼",
  "fairy": "妖精"
};

// 圖鑑地區翻譯
const pokedexTranslations = {
  "kanto": "關都",
  "johto": "城都",
  "hoenn": "豐緣",
  "sinnoh": "神奧",
  "unova": "合眾",
  "kalos": "卡洛斯",
  "alola": "阿羅拉",
  "galar": "伽勒爾",
  "hisui": "洗翠",
  "paldea": "帕底亞",
  "national": "全國"
};

const fetchPokemons = async () => {
  try {
    const response = await axios.get("https://pokeapi.co/api/v2/pokemon?limit=1008");

    pokemons.value = response.data.results.map((poke, index) => ({
      id: index + 1,
      name: poke.name,
      image: `https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/other/official-artwork/${index + 1}.png`,
      url: poke.url
    }));
  } catch (error) {
    console.error("獲取寶可夢資料時發生錯誤:", error);
  } finally {
    setTimeout(() => {
      isLoading.value = false; // 過 1.5 秒再隱藏動畫，讓過渡更順暢
    }, 1500);
  }
};

const filteredPokemons = computed(() => {
  return pokemons.value.filter(pokemon => {
    const query = searchQuery.value.toLowerCase().trim();
    return pokemon.name.toLowerCase().includes(query) || pokemon.id.toString().includes(query);
  });
});

const openModal = async (pokemon) => {
  selectedPokemon.value = { ...pokemon, loading: true };

  try {
    const response = await axios.get(pokemon.url);
    const speciesResponse = await axios.get(response.data.species.url);

    const chineseName = speciesResponse.data.names.find(name => name.language.name === "zh-Hant")?.name || pokemon.name;
    const regionalPokedex = speciesResponse.data.pokedex_numbers.find(entry => entry.pokedex.name !== "national");
    const regionalDexName = regionalPokedex ? (pokedexTranslations[regionalPokedex.pokedex.name] || "未知") : "未知";

    selectedPokemon.value = {
      ...pokemon,
      name: chineseName,
      types: response.data.types.map(t => typeTranslations[t.type.name] || t.type.name).join(", "),
      height: (response.data.height / 10).toFixed(1),
      weight: (response.data.weight / 10).toFixed(1),
      regionalDex: regionalDexName,
      description: speciesResponse.data.flavor_text_entries.find(entry => entry.language.name === "zh-Hant")?.flavor_text.replace(/\n/g, " ") ||
        speciesResponse.data.flavor_text_entries.find(entry => entry.language.name === "zh")?.flavor_text.replace(/\n/g, " ") ||
        "無描述"
    };
  } catch (error) {
    console.error("獲取寶可夢詳細資料時發生錯誤:", error);
  }
};

const closeModal = () => {
  selectedPokemon.value = null;
};

onMounted(fetchPokemons);
</script>

<style scoped>
.card {
  text-align: center;
  transition: transform 0.3s;
}

.card:hover {
  transform: scale(1.05);
}

/* 模態背景透明，並加上模糊效果 */
.modal.fade.show.d-block {
  background-color: rgba(0, 0, 0, 0.6);
  /* 背景透明並加上黑色背景 */
  backdrop-filter: blur(5px);
  /* 背景模糊 */
}

/* 🌀 加載動畫畫面 */
.loading-screen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100vh;
  background-color: white;
  /* 可改為其他背景色 */
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 9999;
  transition: opacity 0.5s ease-out;
}

.loading-image {
  width: 150px;
  height: auto;
  animation: spin 2s linear infinite;
}

/* 旋轉動畫 */
@keyframes spin {
  from {
    transform: rotate(0deg);
  }

  to {
    transform: rotate(360deg);
  }
}
</style>
