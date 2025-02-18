<script setup lang="ts">
import BlockerForm from './components/BlockerForm.vue'
</script>

<template>
    <div class="app-container">
      <BlockerForm />
    </div>
    <footer class="app-footer">
      <p>© {{ new Date().getFullYear() }} <a href="https://github.com/p-0-m/">Pierre-Olivier Mace</a>. Distributed under the 
        <a href="/LICENSE.txt">MIT</a> license.
      </p>
    </footer>
</template>

<style>
.app-container {
  width: 100%;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}

.app-footer {
  color:white;
  position: fixed;
  bottom: 0;
  width: 100%;
  padding: 1rem;
  text-align: center;
}

.app-footer a {
  color: white;
}
</style>