<script setup lang="ts">
import { ref } from 'vue'
import { BskyAgent } from '@atproto/api'

const identifier = ref('')
const password = ref('')
const jsonFile = ref<File | null>(null)
const status = ref('')
const isProcessing = ref(false)

const handleFileChange = (event: Event) => {
  const target = event.target as HTMLInputElement
  if (target.files) {
    jsonFile.value = target.files[0]
  }
}

const processBlocks = async () => {
  if (!jsonFile.value) {
    status.value = 'Please select a JSON file'
    return
  }

  try {
    isProcessing.value = true
    status.value = 'Logging in...'
    
    const agent = new BskyAgent({ service: 'https://bsky.social' })
    await agent.login({ identifier: identifier.value, password: password.value })
    
    status.value = 'Reading JSON file...'
    const fileContent = await jsonFile.value.text()
    const accounts = JSON.parse(fileContent)
    
    if (!Array.isArray(accounts)) {
      throw new Error('JSON file must contain an array of account DIDs or handles')
    }

    status.value = 'Processing blocks...'
    let blockedCount = 0
    
    for (const account of accounts) {
      try {
        await agent.getProfile({ actor: account })
        await agent.app.bsky.graph.block.create(
        { repo: agent.session?.did },
        {
          subject: await getDidFromHandle(account) || '',
          createdAt: new Date().toISOString()
        })

        blockedCount++
        status.value = `Blocked ${blockedCount} of ${accounts.length} accounts...`
      } catch (error) {
        console.error(`Failed to block ${account}:`, error)
      }
    }

    status.value = `Completed! Blocked ${blockedCount} accounts.`
  } catch (error) {
    status.value = `Error: ${(error as Error).message}`
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
</script>

<template>
  <div class="blocker-form">
    <h2>Bluesky Account Blocker</h2>
    
    <div class="form-group">
      <label for="identifier">Bluesky Handle or Email:</label>
      <input 
        id="identifier"
        v-model="identifier"
        type="text"
        :disabled="isProcessing"
        placeholder="you.bsky.social"
      />
    </div>

    <div class="form-group">
      <label for="password">Password:</label>
      <input 
        id="password"
        v-model="password"
        type="password"
        :disabled="isProcessing"
      />
    </div>

    <div class="form-group">
      <label for="jsonFile">JSON File with Accounts:</label>
      <input 
        id="jsonFile"
        type="file"
        accept=".json"
        @change="handleFileChange"
        :disabled="isProcessing"
      />
    </div>

    <button 
      @click="processBlocks"
      :disabled="isProcessing || !identifier || !password || !jsonFile"
    >
      {{ isProcessing ? 'Processing...' : 'Block Accounts' }}
    </button>

    <div v-if="status" class="status" :class="{ error: status.startsWith('Error') }">
      {{ status }}
    </div>
  </div>
</template>

<style scoped>
.blocker-form {
  max-width: 500px;
  margin: 0 auto;
  padding: 20px;
}

.form-group {
  margin-bottom: 15px;
}

label {
  display: block;
  margin-bottom: 5px;
}

input {
  width: 100%;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

button {
  width: 100%;
  padding: 10px;
  background-color: #646cff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.status {
  margin-top: 15px;
  padding: 10px;
  border-radius: 4px;
  background-color: #e8f5e9;
}

.status.error {
  background-color: #ffebee;
  color: #c62828;
}
</style>