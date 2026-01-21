<script setup>
import { ref } from 'vue'
import axios from 'axios'

const API_URL = 'http://localhost:8000'

const documents = ref('')
const question = ref('')
const answer = ref(null)
const history = ref([])
const loading = ref(false)
const activeTab = ref('add')

const addDocument = async () => {
  if (!documents.value.trim()) return
  
  loading.value = true
  try {
    const docs = documents.value.split('\n').filter(d => d.trim())
    for (const doc of docs) {
      await axios.post(`${API_URL}/documents`, { text: doc })
    }
    alert(`${docs.length} documento(s) agregado(s)`)
    documents.value = ''
  } catch (error) {
    alert('Error: ' + error.message)
  } finally {
    loading.value = false
  }
}

const askQuestion = async () => {
  if (!question.value.trim()) return
  
  loading.value = true
  answer.value = null
  try {
    const res = await axios.post(`${API_URL}/query`, { question: question.value })
    answer.value = res.data
  } catch (error) {
    alert('Error: ' + error.message)
  } finally {
    loading.value = false
  }
}

const loadHistory = async () => {
  loading.value = true
  try {
    const res = await axios.get(`${API_URL}/history?limit=20`)
    console.log('History response:', res.data)
    
    // Backend returns array directly
    if (Array.isArray(res.data)) {
      history.value = res.data
    } else if (res.data && Array.isArray(res.data.queries)) {
      history.value = res.data.queries
    } else {
      history.value = []
    }
    
    console.log('History loaded:', history.value.length, 'items')
  } catch (error) {
    console.error('History error:', error)
    history.value = [] // Ensure it's always an array
    alert('Error: ' + error.message)
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <div class="container">
    <header>
      <h1>🤖 Test RAG API</h1>
      <p>FastAPI + PostgreSQL + Qdrant + Ollama</p>
    </header>

    <div class="tabs">
      <button @click="activeTab = 'add'" :class="{ active: activeTab === 'add' }">
        📄 Agregar Documentos
      </button>
      <button @click="activeTab = 'query'" :class="{ active: activeTab === 'query' }">
        💬 Hacer Pregunta
      </button>
      <button @click="activeTab = 'history'; loadHistory()" :class="{ active: activeTab === 'history' }">
        📚 Historial
      </button>
    </div>

    <!-- Tab: Agregar Documentos -->
    <div v-if="activeTab === 'add'" class="tab-content">
      <h2>Agregar Documentos</h2>
      <textarea
        v-model="documents"
        placeholder="Ingresa documentos (uno por línea)"
        rows="10"
      ></textarea>
      <button @click="addDocument" :disabled="loading" class="btn-primary">
        {{ loading ? 'Agregando...' : 'Agregar Documentos' }}
      </button>
    </div>

    <!-- Tab: Hacer Pregunta -->
    <div v-if="activeTab === 'query'" class="tab-content">
      <h2>Hacer Pregunta</h2>
      <input
        v-model="question"
        @keyup.enter="askQuestion"
        type="text"
        placeholder="¿Qué quieres saber?"
        :disabled="loading"
      />
      <button @click="askQuestion" :disabled="loading" class="btn-primary">
        {{ loading ? 'Consultando...' : 'Preguntar' }}
      </button>

      <div v-if="answer" class="answer-card">
        <h3>💡 Respuesta:</h3>
        <p>{{ answer.answer }}</p>
        
        <h4>📎 Fuentes:</h4>
        <ul>
          <li v-for="(source, idx) in answer.sources" :key="idx">{{ source }}</li>
        </ul>
        
        <small>ID: {{ answer.id }} | Guardado en PostgreSQL</small>
      </div>
    </div>

    <!-- Tab: Historial -->
    <div v-if="activeTab === 'history'" class="tab-content">
      <h2>Historial de Consultas</h2>
      
      <div v-if="!history || history.length === 0" class="empty">
        No hay consultas guardadas
      </div>

      <div v-for="item in (history || [])" :key="item.id" class="history-card">
        <div class="history-header">
          <strong>{{ item.question }}</strong>
          <span class="date">{{ new Date(item.created_at).toLocaleString() }}</span>
        </div>
        <p>{{ item.answer }}</p>
        <small>{{ item.sources_count }} fuentes | Modelo: {{ item.model }}</small>
      </div>
    </div>
  </div>
</template>

<style scoped>
* {
  box-sizing: border-box;
}

.container {
  max-width: 900px;
  margin: 0 auto;
  padding: 2rem;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

header {
  text-align: center;
  margin-bottom: 2rem;
}

h1 {
  margin: 0;
  color: #2c3e50;
}

header p {
  color: #7f8c8d;
  margin-top: 0.5rem;
}

.tabs {
  display: flex;
  gap: 1rem;
  margin-bottom: 2rem;
  border-bottom: 2px solid #ecf0f1;
}

.tabs button {
  padding: 0.75rem 1.5rem;
  border: none;
  background: none;
  cursor: pointer;
  font-size: 1rem;
  color: #7f8c8d;
  transition: all 0.3s;
}

.tabs button:hover {
  color: #2c3e50;
}

.tabs button.active {
  color: #3498db;
  border-bottom: 3px solid #3498db;
}

.tab-content {
  animation: fadeIn 0.3s;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

h2 {
  color: #2c3e50;
  margin-bottom: 1rem;
}

textarea, input {
  width: 100%;
  padding: 0.75rem;
  border: 2px solid #ecf0f1;
  border-radius: 8px;
  font-size: 1rem;
  font-family: inherit;
  margin-bottom: 1rem;
}

textarea:focus, input:focus {
  outline: none;
  border-color: #3498db;
}

.btn-primary {
  background: #3498db;
  color: white;
  border: none;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  font-size: 1rem;
  cursor: pointer;
  transition: background 0.3s;
}

.btn-primary:hover:not(:disabled) {
  background: #2980b9;
}

.btn-primary:disabled {
  background: #95a5a6;
  cursor: not-allowed;
}

.answer-card, .history-card {
  background: #f8f9fa;
  padding: 1.5rem;
  border-radius: 12px;
  margin-top: 1.5rem;
  border-left: 4px solid #3498db;
}

.answer-card h3 {
  margin-top: 0;
  color: #2c3e50;
}

.answer-card p {
  color: #34495e;
  line-height: 1.6;
}

.answer-card h4 {
  color: #7f8c8d;
  font-size: 0.9rem;
  margin-top: 1rem;
  margin-bottom: 0.5rem;
}

.answer-card ul {
  list-style: none;
  padding: 0;
}

.answer-card li {
  background: white;
  padding: 0.5rem;
  margin: 0.5rem 0;
  border-radius: 6px;
  font-size: 0.9rem;
  color: #7f8c8d;
}

.history-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
}

.date {
  font-size: 0.85rem;
  color: #95a5a6;
}

.history-card p {
  color: #34495e;
  margin: 0.5rem 0;
}

.history-card small {
  color: #95a5a6;
}

.empty {
  text-align: center;
  color: #95a5a6;
  padding: 2rem;
}
</style>