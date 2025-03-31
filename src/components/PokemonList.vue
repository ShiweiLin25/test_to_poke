<template>
  <div class="container">
    <!-- 圖鑑標題（圖片） -->
    <div class="text-center my-4">
      <img src="..\..\public\image\pokemon.png" alt="寶可夢圖鑑" class="img-fluid" style="width: 40%; height: auto;" />
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

// 存放寶可夢的基本資訊
const pokemons = ref([]);
const selectedPokemon = ref(null);
const searchQuery = ref(""); // 搜尋框的輸入內容
const typeFilter = ref(""); // 選擇的寶可夢類型篩選器
const typeCounts = ref({}); // 儲存每種類型的寶可夢數量

// 固定的屬性類型對照表
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

// 取得寶可夢的基本資料
const fetchPokemons = async () => {
  try {
    const response = await axios.get("https://pokeapi.co/api/v2/pokemon?limit=1008");
    const pokemonList = await Promise.all(response.data.results.map(async (poke, index) => {
      const pokemonData = {
        id: index + 1,
        name: poke.name,
        image: `https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/other/official-artwork/${index + 1}.png`,
        url: poke.url
      };

      // 取得寶可夢詳細資料以獲得中文名稱
      const pokemonDetail = await axios.get(poke.url);
      const speciesResponse = await axios.get(pokemonDetail.data.species.url);

      // 取得中文名稱
      const chineseName = speciesResponse.data.names.find(name => name.language.name === "zh-Hant")?.name || poke.name;

      return {
        ...pokemonData,
        chineseName, // 儲存中文名稱
        types: pokemonDetail.data.types.map(t => t.type.name) // 儲存寶可夢的類型
      };
    }));

    pokemons.value = pokemonList;

    // 計算每種類型的數量
    const counts = {};
    pokemonList.forEach(pokemon => {
      pokemon.types.forEach(type => {
        counts[type] = counts[type] ? counts[type] + 1 : 1;
      });
    });

    typeCounts.value = counts;

  } catch (error) {
    console.error("獲取寶可夢資料時發生錯誤:", error);
  }
};

// 依據搜尋關鍵字和類型過濾寶可夢
const filteredPokemons = computed(() => {
  return pokemons.value.filter(pokemon => {
    const query = searchQuery.value.toLowerCase().trim();
    const matchesSearchQuery = pokemon.name.toLowerCase().includes(query) ||
      pokemon.id.toString().includes(query) ||
      pokemon.chineseName.toLowerCase().includes(query);

    const matchesTypeFilter = typeFilter.value ? pokemon.types.includes(typeFilter.value) : true;

    return matchesSearchQuery && matchesTypeFilter;
  });
});

const openModal = async (pokemon) => {
  selectedPokemon.value = pokemon;

  try {
    const response = await axios.get(pokemon.url);
    const speciesResponse = await axios.get(response.data.species.url);

    // 取得中文名稱
    const chineseName = speciesResponse.data.names.find(name => name.language.name === "zh-Hant")?.name || pokemon.name;

    // 取得地區圖鑑資料
    const regionalPokedex = speciesResponse.data.pokedex_numbers.find(entry => entry.pokedex.name !== "national");

    // 如果沒有其他地區圖鑑，則使用第一個地區圖鑑
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
</style>
