<template>
  <div class="container">
    <!-- 加載中的畫面，當資料還在加載時顯示 -->
    <div v-if="isLoading" class="loading-screen">
      <img src="/image/ball.png" alt="載入中" class="loading-image">
    </div>

    <!-- 圖鑑標題（圖片） -->
    <div class="text-center my-4">
      <img src="/image/pokemon.png" alt="寶可夢圖鑑" class="img-fluid" style="width: 40%; height: auto;" />
    </div>

    <!-- 搜尋框和類型過濾器 -->
    <div class="mb-3">
      <div class="input-group">
        <input v-model="searchQuery" type="text" class="form-control w-75" placeholder="搜尋寶可夢（名稱、編號）" />
        <select v-model="typeFilter" class="form-select">
          <option value="">所有類型</option>
          <!-- 顯示所有類型的過濾選項，以及每種類型的寶可夢數量 -->
          <option v-for="(label, type) in typeTranslations" :key="type" :value="type">
            {{ label }} ({{ typeCounts[type] || 0 }})
          </option>
        </select>
      </div>
    </div>

    <!-- 使用 Bootstrap 的 row 來建立網格佈局，顯示每隻寶可夢 -->
    <div class="row">
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

    <!-- 顯示選中寶可夢的詳細資訊的 Modal -->
    <div v-if="selectedPokemon" class="modal fade show d-block" tabindex="-1">
      <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content">
          <div class="modal-header">
            <h4 class="modal-title"><strong>{{ selectedPokemon.name }}</strong></h4>
            <button type="button" class="btn-close" @click="closeModal"></button>
          </div>
          <hr class="hr-line">
          <div class="modal-body">
            <!-- 使用 flexbox 實現圖片和文字並排 -->
            <div class="pokemon-details">
              <!-- 圖片部分 -->
              <div class="pokemon-image">
                <img :src="selectedPokemon.image" class="img-fluid mb-3 rounded" :alt="selectedPokemon.name">
              </div>
              <!-- 文字部分 -->
              <div class="pokemon-info">
                <p><strong>編號：</strong> {{ selectedPokemon.id }}</p>
                <p><strong>屬型：</strong> {{ selectedPokemon.types }}</p>
                <p><strong>身高：</strong> {{ selectedPokemon.height }} m</p>
                <p><strong>體重：</strong> {{ selectedPokemon.weight }} kg</p>
                <p><strong>地區：</strong> {{ selectedPokemon.regionalDex }}</p>
              </div>
            </div>
            <hr class="hr-line">
            <!-- 描述部分，放置於下方 -->
            <div class="pokemon-description">
              <p><strong>描述：</strong> {{ selectedPokemon.description }}</p>
            </div>
          </div>
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, onMounted, computed } from "vue";
import axios from "axios";

// 定義變數來控制加載狀態
const isLoading = ref(true);

// 存放寶可夢基本資訊的陣列
const pokemons = ref([]);

// 存放選中寶可夢的詳細資料
const selectedPokemon = ref(null);

// 搜尋框的輸入內容
const searchQuery = ref("");

// 類型過濾器選擇的值
const typeFilter = ref("");

// 儲存每種類型的寶可夢數量
const typeCounts = ref({});

// 類型名稱的翻譯
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

// 地區圖鑑名稱的翻譯對照表
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

// 獲取寶可夢基本資料的函式
const fetchPokemons = async () => {
  try {
    // 透過 API 獲取寶可夢的基本資料
    const response = await axios.get("https://pokeapi.co/api/v2/pokemon?limit=1008");
    const pokemonList = response.data.results.map((poke, index) => ({
      id: index + 1,
      name: poke.name,
      image: `https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/other/official-artwork/${index + 1}.png`,
      url: poke.url,
      types: []
    }));

    // 同時請求所有寶可夢的詳細資料
    const detailsResponses = await Promise.allSettled(pokemonList.map(p => axios.get(p.url)));

    // 同時請求所有寶可夢的物種資料
    const speciesResponses = await Promise.allSettled(pokemonList.map(p => axios.get(`https://pokeapi.co/api/v2/pokemon-species/${p.id}/`)));


    // 計算每種類型的寶可夢數量
    const typeCountMap = {};

    // 更新寶可夢的基本資料
    pokemons.value = pokemonList.map((pokemon, index) => {
      const detailsData = detailsResponses[index]?.value?.data || {};
      const speciesData = speciesResponses[index]?.value?.data || {};

      const pokemonTypes = detailsData.types?.map(t => t.type.name) || [];

      // 更新類型計數
      pokemonTypes.forEach(type => {
        if (!typeCountMap[type]) {
          typeCountMap[type] = 0;
        }
        typeCountMap[type]++;
      });

      return {
        ...pokemon,
        chineseName: speciesData.names?.find(n => n.language.name === "zh-Hant")?.name || pokemon.name,
        types: pokemonTypes
      };
    });

    // 更新類型計數
    typeCounts.value = typeCountMap;

  } catch (error) {
    console.error("獲取寶可夢資料時發生錯誤:", error);
  } finally {
    // 資料加載完成後，隱藏加載動畫
    isLoading.value = false;
  }
};

// 根據搜尋關鍵字和類型過濾寶可夢
const filteredPokemons = computed(() => {
  return pokemons.value.filter(pokemon => {
    const query = searchQuery.value.toLowerCase().trim();
    const matchesSearchQuery =
      pokemon.name.toLowerCase().includes(query) ||
      pokemon.id.toString().includes(query) ||
      (pokemon.chineseName?.toLowerCase() || "").includes(query);

    // 確保符合類型過濾條件
    const matchesTypeFilter = typeFilter.value
      ? pokemon.types.includes(typeFilter.value)
      : true;

    return matchesSearchQuery && matchesTypeFilter;
  });
});

// 開啟模態窗口顯示寶可夢詳細資料
const openModal = async (pokemon) => {
  selectedPokemon.value = pokemon;

  try {
    const response = await axios.get(pokemon.url);
    const speciesResponse = await axios.get(response.data.species.url);

    const chineseName = speciesResponse.data.names.find(name => name.language.name === "zh-Hant")?.name || pokemon.name;
    const regionalPokedex = speciesResponse.data.pokedex_numbers.find(entry => entry.pokedex.name !== "national");
    const regionalDexName = regionalPokedex ? (pokedexTranslations[regionalPokedex.pokedex.name] || "未知") : "未知";

    // 更新選中寶可夢的詳細資訊
    selectedPokemon.value = {
      ...selectedPokemon.value,
      name: chineseName,
      types: response.data.types.map(t => typeTranslations[t.type.name] || t.type.name).join(", "),
      height: (response.data.height / 10).toFixed(1),
      weight: (response.data.weight / 10).toFixed(1),
      regionalDex: regionalDexName,
      description: speciesResponse.data.flavor_text_entries.find(entry => entry.language.name === "zh-Hant")?.flavor_text.replace(/\n/g, " ") ||
        speciesResponse.data.flavor_text_entries.find(entry => entry.language.name === "zh")?.flavor_text.replace(/\n/g, " ") ||
        speciesResponse.data.flavor_text_entries.find(entry => entry.language.name === "en")?.flavor_text.replace(/\n/g, " ") ||
        "無描述"
    };
  } catch (error) {
    console.error("獲取寶可夢詳細資料時發生錯誤:", error);
  }
};

// 關閉模態窗口
const closeModal = () => {
  selectedPokemon.value = null;
};

// 組件加載時獲取寶可夢資料
onMounted(fetchPokemons);
</script>

<style scoped>
/* 卡片效果，當鼠標懸停時放大 */
.card {
  text-align: center;
  transition: transform 0.3s;
}

.card:hover {
  transform: scale(1.05);
}

/* 模態背景透明並加上模糊效果 */
.modal.fade.show.d-block {
  background-color: rgba(0, 0, 0, 0.6);
  backdrop-filter: blur(5px);
}

/* 加載動畫樣式 */
.loading-screen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100vh;
  background-color: white;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 9999;
  transition: opacity 0.5s ease-out;
}

/* 旋轉動畫 */
.loading-image {
  width: 150px;
  height: auto;
  animation: spin 2s linear infinite;
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }

  to {
    transform: rotate(360deg);
  }
}

/* Modal 樣式 */
.modal-content {
  border-radius: 15px;
  padding: 20px;
}

/* 使用 flexbox 排版圖片和文字 */
.pokemon-details {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 20px;
  /* 給圖片和右側信息一點間距 */
}

/* 圖片樣式，確保背景透明 */
.pokemon-image {
  flex: 0 0 50%;
  /* 圖片佔據 40% 的寬度 */
  text-align: center;
}

img.img-fluid {
  max-width: 100%;
}

/* 右側文字區域樣式 */
.pokemon-info {
  flex: 1;
  /* 文字區域佔據剩餘空間 */
  padding-top: 15px;
  padding-left: 35px;
}

.pokemon-info p {
  font-size: 1.1rem;
  line-height: 1.6;
  margin: 8px 0;
}

/* 描述文字樣式 */
.pokemon-description p {
  font-size: 1.1rem;
  line-height: 1.6;
  margin-top: 20px;
  text-align: left;
}

/* 標題樣式 */
.modal-header {
  border-bottom: none;
  font-size: 1.5rem;
  font-weight: bold;
  color: #333;
}

.modal-body {
  font-family: 'Arial', sans-serif;
}

/* 強調的文字樣式 */
.pokemon-info strong,
.pokemon-description strong {
  color: #5e5e5e;
}

.hr-line {
  border: 1px #333 solid;
}
</style>