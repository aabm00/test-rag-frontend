<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'

const API_URL = 'http://localhost:8000'

// Authentication state
const isAuthenticated = ref(false)
const currentUser = ref(null)
const authMode = ref('login') // 'login' or 'register'

// Auth form data
const loginEmail = ref('')
const loginPassword = ref('')
const registerEmail = ref('')
const registerUsername = ref('')
const registerPassword = ref('')
const registerFullName = ref('')

// App state
const documents = ref('')
const selectedFile = ref(null)
const uploadMode = ref('text') // 'text' or 'pdf'
const question = ref('')
const answer = ref(null)
const history = ref([])
const loading = ref(false)
const activeTab = ref('add')
const streamingMode = ref(true) // Enable streaming by default
const isStreaming = ref(false)

// Configure axios to include token
axios.interceptors.request.use(config => {
  const token = localStorage.getItem('access_token')
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
}, error => {
  return Promise.reject(error)
})

// Handle 401 errors (token expired) - but not during login
let isLoggingIn = false

axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response && error.response.status === 401) {
      // Only auto-logout if we're already authenticated (not during login)
      if (isAuthenticated.value && !isLoggingIn) {
        logout()
        alert('Sesión expirada. Por favor, inicia sesión nuevamente.')
      }
    }
    return Promise.reject(error)
  }
)

// Auth functions
const login = async () => {
  loading.value = true
  isLoggingIn = true
  try {
    // OAuth2 expects 'username' field but we send email
    const formData = new FormData()
    formData.append('username', loginEmail.value)
    formData.append('password', loginPassword.value)

    const res = await axios.post(`${API_URL}/auth/login`, formData, {
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
    })
    
    // Save tokens
    localStorage.setItem('access_token', res.data.access_token)
    localStorage.setItem('refresh_token', res.data.refresh_token)
    
    // Small delay to ensure localStorage is ready
    await new Promise(resolve => setTimeout(resolve, 50))
    
    // Get user info - the interceptor will add the token
    const userRes = await axios.get(`${API_URL}/auth/me`)
    
    currentUser.value = userRes.data
    isAuthenticated.value = true
    
    // Clear password field
    loginPassword.value = ''
    
    console.log('Login successful:', currentUser.value.username)
  } catch (error) {
    console.error('Login error:', error)
    // Clear tokens on error
    localStorage.removeItem('access_token')
    localStorage.removeItem('refresh_token')
    
    // Don't show "session expired" for login errors
    if (error.response?.status === 401) {
      alert('Email o contraseña incorrectos')
    } else {
      alert('Error de login: ' + (error.response?.data?.detail || error.message))
    }
  } finally {
    loading.value = false
    isLoggingIn = false
  }
}

const register = async () => {
  if (registerPassword.value.length < 8) {
    alert('La contraseña debe tener al menos 8 caracteres')
    return
  }
  
  loading.value = true
  try {
    await axios.post(`${API_URL}/auth/register`, {
      email: registerEmail.value,
      username: registerUsername.value,
      password: registerPassword.value,
      full_name: registerFullName.value || null
    })
    
    alert('Registro exitoso! Ahora puedes iniciar sesión.')
    authMode.value = 'login'
    loginEmail.value = registerEmail.value
    
    // Clear register form
    registerEmail.value = ''
    registerUsername.value = ''
    registerPassword.value = ''
    registerFullName.value = ''
  } catch (error) {
    alert('Error de registro: ' + (error.response?.data?.detail || error.message))
  } finally {
    loading.value = false
  }
}

const loadCurrentUser = async () => {
  try {
    const res = await axios.get(`${API_URL}/auth/me`)
    currentUser.value = res.data
    isAuthenticated.value = true
  } catch (error) {
    logout()
  }
}

const logout = () => {
  localStorage.removeItem('access_token')
  localStorage.removeItem('refresh_token')
  isAuthenticated.value = false
  currentUser.value = null
  activeTab.value = 'add'
}

// App functions
const handleFileSelect = (event) => {
  const file = event.target.files[0]
  if (file && file.type === 'application/pdf') {
    selectedFile.value = file
  } else if (file) {
    alert('Por favor selecciona un archivo PDF')
    event.target.value = ''
  }
}

const addDocument = async () => {
  if (uploadMode.value === 'text') {
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
      alert('Error: ' + (error.response?.data?.detail || error.message))
    } finally {
      loading.value = false
    }
  } else {
    // PDF upload mode
    if (!selectedFile.value) return
    
    loading.value = true
    try {
      const formData = new FormData()
      formData.append('file', selectedFile.value)
      
      await axios.post(`${API_URL}/documents/upload`, formData, {
        headers: { 'Content-Type': 'multipart/form-data' }
      })
      
      alert(`PDF "${selectedFile.value.name}" subido exitosamente`)
      selectedFile.value = null
      // Reset file input
      const fileInput = document.querySelector('input[type="file"]')
      if (fileInput) fileInput.value = ''
    } catch (error) {
      alert('Error: ' + (error.response?.data?.detail || error.message))
    } finally {
      loading.value = false
    }
  }
}

const askQuestion = async () => {
  if (!question.value.trim()) return
  
  if (streamingMode.value) {
    await askQuestionStreaming()
  } else {
    await askQuestionNormal()
  }
}

const askQuestionNormal = async () => {
  loading.value = true
  answer.value = null
  try {
    const res = await axios.post(`${API_URL}/query`, { question: question.value })
    answer.value = res.data
  } catch (error) {
    alert('Error: ' + (error.response?.data?.detail || error.message))
  } finally {
    loading.value = false
  }
}

const askQuestionStreaming = async () => {
  if (!question.value.trim()) return
  
  loading.value = true
  isStreaming.value = true
  answer.value = {
    question: question.value,
    answer: '',
    sources: [],
    model_used: '',
    created_at: new Date().toISOString()
  }
  
  const token = localStorage.getItem('access_token')
  
  try {
    const response = await fetch(`${API_URL}/query/stream`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ question: question.value })
    })
    
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`)
    }
    
    const reader = response.body.getReader()
    const decoder = new TextDecoder()
    
    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      
      const chunk = decoder.decode(value)
      const lines = chunk.split('\n')
      
      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = JSON.parse(line.substring(6))
          
          if (data.type === 'token') {
            answer.value.answer += data.content
          } else if (data.type === 'answer') {
            // Cached response (complete answer)
            answer.value.answer = data.content
          } else if (data.type === 'sources') {
            answer.value.sources = data.sources
          } else if (data.type === 'done') {
            answer.value.id = data.id
          } else if (data.type === 'error') {
            throw new Error(data.message)
          } else if (data.type === 'status') {
            // Optional: show status messages
            console.log('Status:', data.message)
          }
        }
      }
    }
  } catch (error) {
    alert('Error: ' + (error.message || 'Streaming failed'))
    answer.value = null
  } finally {
    loading.value = false
    isStreaming.value = false
  }
}

const loadHistory = async () => {
  loading.value = true
  try {
    const res = await axios.get(`${API_URL}/history?limit=20`)
    
    if (Array.isArray(res.data)) {
      history.value = res.data
    } else if (res.data && Array.isArray(res.data.queries)) {
      history.value = res.data.queries
    } else {
      history.value = []
    }
  } catch (error) {
    console.error('History error:', error)
    history.value = []
    alert('Error: ' + (error.response?.data?.detail || error.message))
  } finally {
    loading.value = false
  }
}

// Check authentication on mount
onMounted(() => {
  const token = localStorage.getItem('access_token')
  if (token) {
    loadCurrentUser()
  }
})
</script>

<template>
  <div class="container">
    <!-- Login/Register Screen -->
    <div v-if="!isAuthenticated" class="auth-container">
      <div class="auth-box">
        <h1>🤖 Test RAG API</h1>
        <p class="subtitle">FastAPI + PostgreSQL + Qdrant + Ollama</p>

        <div class="auth-tabs">
          <button 
            @click="authMode = 'login'" 
            :class="{ active: authMode === 'login' }"
          >
            Iniciar Sesión
          </button>
          <button 
            @click="authMode = 'register'" 
            :class="{ active: authMode === 'register' }"
          >
            Registrarse
          </button>
        </div>

        <!-- Login Form -->
        <div v-if="authMode === 'login'" class="auth-form">
          <h2>Iniciar Sesión</h2>
          <input
            v-model="loginEmail"
            type="email"
            placeholder="Email"
            @keyup.enter="login"
            :disabled="loading"
          />
          <input
            v-model="loginPassword"
            type="password"
            placeholder="Contraseña"
            @keyup.enter="login"
            :disabled="loading"
          />
          <button @click="login" :disabled="loading" class="btn-primary">
            {{ loading ? 'Iniciando...' : 'Iniciar Sesión' }}
          </button>
        </div>

        <!-- Register Form -->
        <div v-if="authMode === 'register'" class="auth-form">
          <h2>Crear Cuenta</h2>
          <input
            v-model="registerEmail"
            type="email"
            placeholder="Email"
            :disabled="loading"
          />
          <input
            v-model="registerUsername"
            type="text"
            placeholder="Nombre de usuario (mín. 3 caracteres)"
            :disabled="loading"
          />
          <input
            v-model="registerPassword"
            type="password"
            placeholder="Contraseña (mín. 8 caracteres)"
            :disabled="loading"
          />
          <input
            v-model="registerFullName"
            type="text"
            placeholder="Nombre completo (opcional)"
            :disabled="loading"
          />
          <button @click="register" :disabled="loading" class="btn-primary">
            {{ loading ? 'Registrando...' : 'Crear Cuenta' }}
          </button>
        </div>
      </div>
    </div>

    <!-- Main App (authenticated) -->
    <div v-else>
      <header>
        <div class="header-content">
          <div>
            <h1>🤖 Test RAG API</h1>
            <p>FastAPI + PostgreSQL + Qdrant + Ollama</p>
          </div>
          <div class="user-info">
            <span>👤 {{ currentUser?.username }}</span>
            <button @click="logout" class="btn-logout">Cerrar Sesión</button>
          </div>
        </div>
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
        
        <!-- Mode selector -->
        <div class="upload-mode-selector">
          <button 
            @click="uploadMode = 'text'" 
            :class="{ active: uploadMode === 'text' }"
            class="mode-btn"
          >
            📝 Texto
          </button>
          <button 
            @click="uploadMode = 'pdf'" 
            :class="{ active: uploadMode === 'pdf' }"
            class="mode-btn"
          >
            📄 PDF
          </button>
        </div>

        <!-- Text input mode -->
        <div v-if="uploadMode === 'text'">
          <textarea
            v-model="documents"
            placeholder="Ingresa documentos (uno por línea)"
            rows="10"
          ></textarea>
        </div>

        <!-- PDF upload mode -->
        <div v-else class="pdf-upload">
          <input
            type="file"
            accept=".pdf"
            @change="handleFileSelect"
            class="file-input"
          />
          <p v-if="selectedFile" class="file-selected">
            ✅ Archivo seleccionado: {{ selectedFile.name }}
          </p>
        </div>

        <button @click="addDocument" :disabled="loading" class="btn-primary">
          {{ loading ? 'Subiendo...' : (uploadMode === 'pdf' ? 'Subir PDF' : 'Agregar Documentos') }}
        </button>
      </div>

      <!-- Tab: Hacer Pregunta -->
      <div v-if="activeTab === 'query'" class="tab-content">
        <h2>Hacer Pregunta</h2>
        
        <!-- Streaming Mode Toggle -->
        <div class="streaming-toggle">
          <label>
            <input type="checkbox" v-model="streamingMode" :disabled="loading" />
            <span>⚡ Modo Streaming (respuesta en tiempo real)</span>
          </label>
        </div>
        
        <input
          v-model="question"
          @keyup.enter="askQuestion"
          type="text"
          placeholder="¿Qué quieres saber?"
          :disabled="loading"
        />
        <button @click="askQuestion" :disabled="loading" class="btn-primary">
          {{ loading ? (isStreaming ? 'Generando...' : 'Consultando...') : 'Preguntar' }}
        </button>

        <div v-if="answer" class="answer-card">
          <h3>💡 Respuesta:</h3>
          <p>{{ answer.answer }}</p>
          
          <h4>📎 Fuentes:</h4>
          <ul v-if="answer.sources && answer.sources.length > 0">
            <li v-for="(source, idx) in answer.sources" :key="idx">{{ source }}</li>
          </ul>
          <p v-else class="no-sources">No se utilizaron fuentes específicas</p>
          
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
  min-height: 100vh;
}

/* Auth Styles */
.auth-container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 80vh;
}

.auth-box {
  background: white;
  padding: 3rem;
  border-radius: 16px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.1);
  width: 100%;
  max-width: 450px;
}

.auth-box h1 {
  text-align: center;
  margin: 0 0 0.5rem 0;
  color: #2c3e50;
}

.subtitle {
  text-align: center;
  color: #7f8c8d;
  margin-bottom: 2rem;
  font-size: 0.9rem;
}

.auth-tabs {
  display: flex;
  gap: 1rem;
  margin-bottom: 2rem;
  border-bottom: 2px solid #ecf0f1;
}

.auth-tabs button {
  flex: 1;
  padding: 0.75rem;
  border: none;
  background: none;
  cursor: pointer;
  font-size: 1rem;
  color: #7f8c8d;
  transition: all 0.3s;
}

.auth-tabs button:hover {
  color: #2c3e50;
}

.auth-tabs button.active {
  color: #3498db;
  border-bottom: 3px solid #3498db;
}

.auth-form {
  animation: fadeIn 0.3s;
}

.auth-form h2 {
  color: #2c3e50;
  margin-bottom: 1.5rem;
  font-size: 1.3rem;
}

/* Header with user info */
.header-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
}

.user-info {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.user-info span {
  color: #2c3e50;
  font-weight: 500;
}

.btn-logout {
  background: #e74c3c;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 6px;
  font-size: 0.9rem;
  cursor: pointer;
  transition: background 0.3s;
}

.btn-logout:hover {
  background: #c0392b;
}

header h1 {
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
  width: 100%;
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

.no-sources {
  color: #95a5a6;
  font-style: italic;
  font-size: 0.9rem;
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

/* Streaming Toggle Styles */
.streaming-toggle {
  margin-bottom: 1rem;
  padding: 0.75rem;
  background: #f0f8ff;
  border-radius: 8px;
  border-left: 4px solid #3498db;
}

.streaming-toggle label {
  display: flex;
  align-items: center;
  cursor: pointer;
  font-size: 0.95rem;
  user-select: none;
}

.streaming-toggle input[type="checkbox"] {
  width: auto;
  margin-right: 0.5rem;
  cursor: pointer;
}

.streaming-toggle span {
  color: #2c3e50;
  font-weight: 500;
}

.streaming-toggle input[type="checkbox"]:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Upload mode selector */
.upload-mode-selector {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
}

.mode-btn {
  flex: 1;
  padding: 0.75rem 1rem;
  border: 2px solid #ddd;
  background: white;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1rem;
  transition: all 0.3s;
}

.mode-btn:hover {
  border-color: #3498db;
  background: #f8f9fa;
}

.mode-btn.active {
  border-color: #3498db;
  background: #3498db;
  color: white;
}

/* PDF upload styles */
.pdf-upload {
  padding: 2rem;
  border: 2px dashed #ddd;
  border-radius: 8px;
  text-align: center;
  margin-bottom: 1rem;
  background: #f8f9fa;
}

.file-input {
  padding: 0.75rem;
  border: none;
  cursor: pointer;
  font-size: 1rem;
}

.file-selected {
  margin-top: 1rem;
  color: #27ae60;
  font-weight: 500;
}
</style>
