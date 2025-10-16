<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb'

declare interface Post {
  _id?: string
  _rev?: string
  post_name: string
  post_content: string
  attributes: string[]
}

const storage = ref<any>(null)
const postsData = ref<Post[]>([])

const newPost = ref<Post>({
  post_name: '',
  post_content: '',
  attributes: []
})

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://admin:Ilovecash1925@localhost:5984/infrandon2_db1')
  if (db) {
    console.log("Connecté à la collection : " + db?.name)
    storage.value = db
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}
// ---------- Read ----------
const fetchData = async () => {
  if (!storage.value) {
    console.warn('Base de données non initialisée')
    return
  }
  try {
    const result = await storage.value.allDocs({ include_docs: true })
    postsData.value = result.rows
      .map((row: any) => row.doc as Post)
      .filter((doc: any) => !!doc)
    console.log('📥 Documents récupérés :', postsData.value)
  } catch (error) {
    console.error('❌ Erreur lors de la récupération des données :', error)
  }
}

// ---------- Create ----------
const addPost = async () => {
  if (!storage.value) return

  const doc: Post = {
    _id: new Date().toISOString(), // ID simple unique
    post_name: newPost.value.post_name.trim(),
    post_content: newPost.value.post_content.trim(),
    attributes: (newPost.value.attributes || []).map(a => String(a).trim()).filter(Boolean)
  }

  if (!doc.post_name || !doc.post_content) {
    console.warn('Champs requis manquants')
    return
  }

  try {
    await storage.value.put(doc)
    console.log('✅ Document ajouté :', doc)

    // Reset formulaire
    newPost.value.post_name = ''
    newPost.value.post_content = ''
    newPost.value.attributes = []

    await fetchData()
  } catch (error) {
    console.error('❌ Erreur lors de l’ajout :', error)
  }
}

onMounted(async () => {
  console.log('=> Composant initialisé')
  initDatabase()
  await fetchData()
})
</script>

<template>
  <div class="container">
    <h1>📡 CouchDB + Vue 3</h1>

    <!-- 📝 Formulaire d'ajout -->
    <div class="form">
      <h2>Ajouter un document</h2>

      <input
        v-model="newPost.post_name"
        placeholder="Nom"
        type="text"
      />
      <input
        v-model="newPost.post_content"
        placeholder="Contenu / Description"
        type="text"
      />
      <input
        v-model="newPost.attributes"
        placeholder="Attributs séparés par des virgules"
        type="text"
        @input="newPost.attributes = ($event.target as HTMLInputElement).value.split(',')"
      />
      <button @click="addPost">Ajouter</button>
    </div>

    <hr />

    <!-- 📃 Liste des documents -->
    <div v-if="postsData.length === 0">
      <p>Aucune donnée trouvée.</p>
    </div>

    <article
      v-for="post in postsData"
      :key="post._id"
      class="item"
    >
      <h2>{{ post.post_name }}</h2>
      <p>{{ post.post_content }}</p>
      <p>Attributs : {{ (post.attributes || []).join(', ') }}</p>
    </article>
  </div>
</template>

<style scoped>
.container {
  padding: 1.5rem;
  color: white;
  max-width: 600px;
  margin: auto;
}
.form {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
}
input {
  padding: 0.5rem;
  border-radius: 4px;
  border: none;
}
button {
  background: #42b883;
  color: white;
  padding: 0.5rem;
  border: none;
  cursor: pointer;
  border-radius: 4px;
}
button:hover {
  background: #2a9d6e;
}
.item {
  background: #1e1e1e;
  padding: 1rem;
  margin-top: 0.5rem;
  border-radius: 6px;
}
</style>