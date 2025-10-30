<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

// ---------- Types ----------
declare interface Post {
  _id?: string
  _rev?: string
  post_name: string
  post_content: string
  attributes: string[]
}

// ---------- State ----------
const storage = ref<PouchDB.Database | null>(null)
const postsData = ref<Post[]>([])

const newPost = ref<Post>({
  post_name: '',
  post_content: '',
  attributes: []
})

const editingPost = ref<Post | null>(null)

// ---------- Init DB ----------
const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  // ⚠️ Garde ton URL/base ou adapte si besoin
  const db = new PouchDB('http://admin:Ilovecash1925@localhost:5984/infrandon2_db1')
  if (db) {
    console.log('Connecté à la collection : ' + db.name)
    storage.value = db
  } else {
    console.warn('Échec lors de la connexion à la base de données')
  }
}

// ---------- Read ----------
const fetchData = async () => {
  if (!storage.value) return console.warn('Base de données non initialisée')

  try {
    const result = await storage.value.allDocs({ include_docs: true })
    postsData.value = result.rows
      .map((row: any) => row.doc as Post)
      .filter((doc: unknown): doc is Post => !!doc)
    console.log('📥 Documents récupérés :', postsData.value)
  } catch (error) {
    console.error('❌ Erreur lors de la récupération des données :', error)
  }
}

// ---------- Create ----------
const addPost = async () => {
  if (!storage.value) return

  const doc: Post = {
    _id: new Date().toISOString(), // ID unique simple
    post_name: newPost.value.post_name.trim(),
    post_content: newPost.value.post_content.trim(),
    attributes: (newPost.value.attributes || [])
      .map(a => String(a).trim())
      .filter(Boolean)
  }

  if (!doc.post_name || !doc.post_content) {
    console.warn('Champs requis manquants')
    return
  }

  try {
    await storage.value.put(doc)
    // Reset formulaire
    newPost.value.post_name = ''
    newPost.value.post_content = ''
    newPost.value.attributes = []
    await fetchData()
  } catch (error) {
    console.error('❌ Erreur lors de l’ajout :', error)
  }
}

// ---------- Update (edit inline) ----------
const startEdit = (post: Post) => {
  // copie profonde simple
  editingPost.value = JSON.parse(JSON.stringify(post))
}

const cancelEdit = () => {
  editingPost.value = null
}

const updatePost = async () => {
  if (!storage.value) return
  const doc = editingPost.value
  if (!doc || !doc._id || !doc._rev) {
    console.warn('⚠️ _id et _rev requis pour la mise à jour')
    return
  }

  // nettoyage minimal
  doc.post_name = doc.post_name.trim()
  doc.post_content = doc.post_content.trim()
  doc.attributes = (doc.attributes || []).map(a => String(a).trim()).filter(Boolean)

  try {
    const res = await storage.value.put(doc)
    console.log('✅ Document mis à jour :', res)
    editingPost.value = null
    await fetchData()
  } catch (error) {
    console.error('❌ Erreur lors de la mise à jour :', error)
    // Optionnel: gestion simple des conflits (_conflict)
  }
}

// ---------- Delete ----------
const deletePost = async (docId?: string, docRev?: string) => {
  if (!storage.value || !docId || !docRev) return

  try {
    await storage.value.remove(docId, docRev)
    await fetchData()
  } catch (error) {
    console.error('❌ Erreur lors de la suppression :', error)
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
    <h1>📡 CouchDB + Vue 3 (CRUD)</h1>

    <!-- Actions -->
    <div class="actions">
      <button @click="fetchData">🔄 Rafraîchir</button>
    </div>

    <!-- 📝 Formulaire d'ajout -->
    <div class="form">
      <h2>Ajouter un document</h2>

      <input
        v-model="newPost.post_name"
        placeholder="Nom (post_name)"
        type="text"
      />
      <input
        v-model="newPost.post_content"
        placeholder="Contenu / Description (post_content)"
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
      <!-- Affichage normal -->
      <template v-if="!editingPost || editingPost._id !== post._id">
        <h2>{{ post.post_name }}</h2>
        <p>{{ post.post_content }}</p>
        <p>Attributs : {{ (post.attributes || []).join(', ') }}</p>
        <div class="row">
          <button @click="startEdit(post)">✏️ Modifier</button>
          <button @click="deletePost(post._id, post._rev)">🗑️ Supprimer</button>
        </div>
      </template>

      <!-- Mode édition -->
      <template v-else>
        <div class="editBox">
          <h3>✏️ Modifier le document</h3>
          <input
            v-model="editingPost.post_name"
            placeholder="Nom (post_name)"
            type="text"
          />
          <input
            v-model="editingPost.post_content"
            placeholder="Contenu / Description"
            type="text"
          />
          <input
            :value="(editingPost.attributes || []).join(', ')"
            placeholder="Attributs (séparés par des virgules)"
            type="text"
            @input="editingPost && (editingPost.attributes = ($event.target as HTMLInputElement).value.split(',').map(s => s.trim()))"
          />
          <div class="row">
            <button @click="updatePost">✅ Enregistrer</button>
            <button @click="cancelEdit">❌ Annuler</button>
          </div>
        </div>
      </template>
    </article>
  </div>
</template>

<style scoped>
.container {
  padding: 1.5rem;
  color: white;
  max-width: 720px;
  margin: auto;
}
.actions {
  margin-bottom: 1rem;
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
  margin-top: 0.75rem;
  border-radius: 6px;
}
.row {
  display: flex;
  gap: 0.5rem;
  margin-top: 0.75rem;
}
.editBox {
  padding: 1rem;
  border: 2px solid #42b883;
  border-radius: 8px;
  background: #17221f;
}
</style>
