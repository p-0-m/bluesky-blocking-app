<template>
  <div class="w-sm p-6 bg-white shadow-lg rounded-lg">
    <h2 class="text-xl font-semibold text-gray-700 mb-6 text-center">
      {{ translations.title }}
    </h2>

    <div class="mb-4">
      <label for="identifier" class="block text-sm font-medium text-gray-600">{{ translations.enterHandle }}</label>
      <input 
        id="identifier"
        v-model="identifier"
        type="text"
        :disabled="isProcessing"
        :placeholder="translations.handlePlaceholder"
        class="w-full px-3 py-2 mt-1 border border-gray-600 rounded-md focus:outline-none focus:ring focus:ring-blue-300 disabled:opacity-50"
      />
    </div>

    <div class="mb-4">
      <label for="password" class="block text-sm font-medium text-gray-600">{{ translations.enterPassword }}</label>
      <input 
        id="password"
        v-model="password"
        type="password"
        :disabled="isProcessing"
        class="w-full px-3 py-2 mt-1 border border-gray-600 rounded-md focus:outline-none focus:ring focus:ring-blue-300 disabled:opacity-50"
      />
    </div>

    <div class="mt-6 mb-6 flex justify-between mx-4 text-sm font-medium text-gray-600">
      <label class="cursor-pointer">
        <input type="radio" value="main" v-model="selectedOption" /> {{ translations.mainFile }}
      </label>
      <label class="cursor-pointer">
        <input type="radio" value="custom" v-model="selectedOption" /> {{ translations.customFile }}
      </label>
    </div>

    <div v-if="selectedOption === 'custom'" class="my-8 ">
      <label for="csvFile" class="block text-sm font-medium text-gray-600">{{ translations.enterCSV }}</label>
      <input 
        id="csvFile"
        type="file"
        accept=".csv"
        @change="handleFileChange"
        :disabled="isProcessing"
        class="text-gray-600 w-full px-3 py-2 mt-1 border border-gray-600 rounded-md focus:outline-none focus:ring focus:ring-blue-300 disabled:opacity-50"
      />
      <p class="text-xs text-gray-500 mt-1">{{ translations.csvInfo }}</p>
    </div>

    <button 
      :disabled="isProcessing || !identifier || !password || (!csvFile && selectedOption === 'custom')"
      @click="processBlocks"
      class="w-full px-4 py-2 text-white bg-blue-600 rounded-md hover:bg-blue-700 focus:outline-none focus:ring focus:ring-blue-300 disabled:opacity-50 disabled:cursor-not-allowed cursor-pointer"
    >
      {{ isProcessing ? translations.processing : translations.block }}
    </button>

    <div v-if="status" class="mt-4 text-sm p-2 rounded-md align-center" :class="{ 'bg-red-10|| !csvFile0 text-red-700': status.startsWith('Error'), 'bg-green-100 text-green-700': !status.startsWith('Error') }">
      {{ status }}
    </div>
  </div>
</template>


<script lang="ts">
  import { onMounted, ref, watch } from 'vue'
  import { BskyAgent } from '@atproto/api'
  import { defineComponent } from 'vue'

  export default defineComponent({

    setup() {
      const identifier = ref('')
      const password = ref('')
      const csvFile = ref<File | null>(null)
      const status = ref('')
      const isProcessing = ref(false)
      const translations = ref({})
      const selectedOption = ref('main')
      const mainCsvContent = ref<string | null>(null)

      const handleFileChange = (event: Event) => {
        const target = event.target as HTMLInputElement
        if (target.files) {
          csvFile.value = target.files[0]
        }
      }

      const loadMainCsv = async () => {
      try {
        const response = await fetch('/main.csv')
        if (!response.ok) throw new Error('Impossible de charger main.csv')
        mainCsvContent.value = await response.text()
      } catch (error) {
        console.error('Erreur de chargement du fichier principal:', error)
      }
    }

    watch(selectedOption, async (newValue) => {
      if (newValue === 'main') {
        await loadMainCsv()
      }
    })


    const processBlocks = async () => {
      try {
        isProcessing.value = true

        const agent = new BskyAgent({ service: 'https://bsky.social' })
        await agent.login({ identifier: identifier.value, password: password.value })

        let fileContent = ''
        if (selectedOption.value === 'main') {
          fileContent = mainCsvContent.value || ''
        } else {
          fileContent = await csvFile.value?.text() || ''
        }

        const accounts = fileContent.split('\n').map(handle => handle.trim()).filter(handle => handle)

        if (accounts.length === 0) {
          throw new Error('La liste d’identifiants ne peut pas être vide.')
        }

        let blockedCount = 0

        for (const account of accounts) {
          try {
            await agent.getProfile({ actor: account })
            await agent.app.bsky.graph.block.create(
              { repo: agent.session?.did },
              {
                subject: await getDidFromHandle(account) || '',
                createdAt: new Date().toISOString()
              }
            )
            blockedCount++
          } catch (error) {
            console.error(`Échec du blocage de ${account}:`, error)
          }
        }

        status.value = `${blockedCount} ${translations.value.success}`
      } catch (error) {
        status.value = `Erreur: ${(error as Error).message}`
      } finally {
        isProcessing.value = false
      }
    }

      async function getDidFromHandle(handle: string): Promise<string | null> {
        try {
          const response = await fetch(
            `https://bsky.social/xrpc/com.atproto.identity.resolveHandle?handle=${encodeURIComponent(handle)}`
          );

          if (!response.ok) {
            throw new Error(`Erreur lors de la récupération du DID : ${response.statusText}`);
          }

          const data = await response.json();
          return data.did || null;
        } catch (error) {
          console.error("Erreur :", error);
          return null;
        }
      }

      onMounted(async () => {
        try {
          const response = await fetch('/EN-en.json');
          translations.value = await response.json();
        } catch (error) {
          console.error('Erreur de chargement des traductions:', error);
        }
      });

      return {
        identifier,
        password,
        csvFile,
        status,
        isProcessing,
        handleFileChange,
        processBlocks,
        selectedOption,
        translations
      }
    }
    
  })
</script>