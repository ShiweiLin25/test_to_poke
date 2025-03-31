<template>
  <div class="d-flex flex-column min-vh-100">
    <PokemonList />
    <footer class="bg-dark text-white text-center py-4 mt-auto">
      <div class="container">
        <p>Copyright &copy; Shiwei Lin 2025</p>
      </div>
    </footer>
  </div>
</template>

<script setup>
import PokemonList from './components/PokemonList.vue';
</script>

<style scoped>
/* 確保主內容區域可以填滿空間 */
.d-flex {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  /* 填滿視窗高度 */
}

footer {
  margin-top: auto;
  /* 確保 footer 顯示在最底部 */
}
</style>
