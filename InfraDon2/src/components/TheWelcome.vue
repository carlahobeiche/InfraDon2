<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

// --- plugin d'indexation ---
;(PouchDB as any).plugin(PouchDBFind)

// ---------- Types ----------
interface Comment {
  text: string
  created_at: string
}

declare interface Post {
  _id?: string
  _rev?: string
  post_name: string
  post_content: string
  attributes: string[]
  created_at?: string
  updated_at?: string
  likes?: number
  comments?: Comment[]
}

// ---------- Constantes DB ----------
const LOCAL_DB_NAME = 'infradon2_local' // DB applicative (offline)
const REMOTE_DB_URL = 'http://admin:Ilovecash1925@localhost:5984/infrandon2_db1' // CouchDB distante

// ---------- State ----------
const localDb = ref<PouchDB.Database | null>(null)
const remoteDb = ref<PouchDB.Database | null>(null)
const syncHandler = ref<any>(null)

const postsData = ref<Post[]>([])
const newPost = ref<Post>({
  post_name: '',
  post_content: '',
  attributes: [],
})

const editingPost = ref<Post | null>(null)

const loading = ref(false)
const error = ref<string | null>(null)

const online = ref(true) // toggle online/offline
const searchTerm = ref('') // recherche par post_name
const sortByLikes = ref(false) // tri par likes

// brouillon de commentaire par document
const commentDrafts = ref<Record<string, string>>({})

// ---------- Helpers ----------
const normalizePost = (raw: any): Post => {
  const likes =
    typeof raw.likes === 'number' && !Number.isNaN(raw.likes) ? raw.likes : 0
  const comments = Array.isArray(raw.comments)
    ? (raw.comments as Comment[])
    : []

  return {
    ...raw,
    likes,
    comments,
    attributes: Array.isArray(raw.attributes) ? raw.attributes : [],
  } as Post
}

const initLocalDb = () => {
  if (!localDb.value) {
    localDb.value = new PouchDB(LOCAL_DB_NAME)
  }
  return localDb.value
}

const initRemoteDb = () => {
  if (!remoteDb.value) {
    remoteDb.value = new PouchDB(REMOTE_DB_URL)
  }
  return remoteDb.value
}

// ---------- Indexation ----------
const ensureIndexes = async () => {
  const db = initLocalDb()
  // index sur nom (recherche) et likes (tri)
  await (db as any).createIndex({ index: { fields: ['post_name'] } })
  await (db as any).createIndex({ index: { fields: ['likes'] } })
}

// ---------- Réplication ----------
const replicateFromRemote = async () => {
  const local = initLocalDb()
  const remote = initRemoteDb()
  await local.replicate.from(remote)
}

const replicateToRemote = async () => {
  const local = initLocalDb()
  const remote = initRemoteDb()
  await local.replicate.to(remote)
}

// sync deux sens (one shot)
const manualSync = async () => {
  await replicateFromRemote()
  await replicateToRemote()
}

// alias plus parlant
const fullSync = manualSync

// sync live (online)
const startLiveSync = () => {
  const local = initLocalDb()
  const remote = initRemoteDb()
  syncHandler.value = (local as any)
    .sync(remote, { live: true, retry: true })
    .on('change', () => {
      fetchData()
    })
    .on('error', (e: any) => {
      console.error('sync error', e)
    })
}

const stopLiveSync = () => {
  if (syncHandler.value && typeof syncHandler.value.cancel === 'function') {
    syncHandler.value.cancel()
  }
}

// toggle online/offline
const toggleOnline = async () => {
  online.value = !online.value
  if (online.value) {
    await manualSync()
    startLiveSync()
  } else {
    stopLiveSync()
  }
}

// ---------- CRUD ----------

// READ – toutes les données (local DB), tri par date
const fetchData = async () => {
  const db = initLocalDb()
  loading.value = true
  error.value = null
  try {
    const result = await db.allDocs({ include_docs: true })
    postsData.value = result.rows
      .filter((r: any) => r.doc)
      .map((r: any) => normalizePost(r.doc))
      .sort(
        (a, b) =>
          new Date(b.created_at || '').getTime() -
          new Date(a.created_at || '').getTime()
      )
  } catch (e: any) {
    console.error(e)
    error.value = 'Erreur fetchData : ' + e.message
  } finally {
    loading.value = false
  }
}

// CREATE – ajout via local DB
const addPost = async () => {
  const db = initLocalDb()

  const now = new Date().toISOString()
  const attrs = (newPost.value.attributes || [])
    .map(a => String(a).trim())
    .filter(Boolean)

  if (!newPost.value.post_name.trim() || !newPost.value.post_content.trim()) {
    console.warn('Champs requis manquants')
    return
  }

  const doc: Omit<Post, '_id' | '_rev'> = {
    post_name: newPost.value.post_name.trim(),
    post_content: newPost.value.post_content.trim(),
    attributes: attrs,
    created_at: now,
    likes: 0,
    comments: [],
  }

  try {
    await (db as any).post(doc)
    // reset formulaire
    newPost.value.post_name = ''
    newPost.value.post_content = ''
    newPost.value.attributes = []
    await fetchData()
    if (online.value) await manualSync()
  } catch (error: any) {
    console.error(error)
  }
}

// UPDATE (mode édition)
const startEdit = (post: Post) => {
  editingPost.value = JSON.parse(JSON.stringify(post))
}

const cancelEdit = () => {
  editingPost.value = null
}

const updatePost = async () => {
  const db = initLocalDb()
  const doc = editingPost.value
  if (!doc || !doc._id) {
    console.warn('_id manquant pour update')
    return
  }

  try {
    // récupérer doc frais (pour likes/comments)
    const existing = await db.get(doc._id)
    const normalized = normalizePost(existing)

    const updated: Post = {
      _id: doc._id,
      _rev: existing._rev,
      post_name: doc.post_name.trim(),
      post_content: doc.post_content.trim(),
      attributes: (doc.attributes || []).map(a => String(a).trim()),
      created_at: normalized.created_at || new Date().toISOString(),
      updated_at: new Date().toISOString(),
      likes: normalized.likes,
      comments: normalized.comments,
    }

    await db.put(updated as any)
    editingPost.value = null
    await fetchData()
    if (online.value) await manualSync()
  } catch (error: any) {
    console.error('Erreur updatePost :', error)
  }
}

// DELETE
const deletePost = async (docId?: string, docRev?: string) => {
  if (!docId || !docRev) return
  const db = initLocalDb()
  try {
    await db.remove(docId, docRev)
    await fetchData()
    if (online.value) await manualSync()
  } catch (error: any) {
    console.error('Erreur deletePost :', error)
  }
}

// ---------- Likes ----------
const likePost = async (post: Post) => {
  if (!post._id) return
  const db = initLocalDb()
  try {
    const fresh = await db.get(post._id)
    const norm = normalizePost(fresh)
    const updated: Post = {
      ...norm,
      likes: (norm.likes || 0) + 1,
    }
    await db.put(updated as any)
    await fetchData()
    if (online.value) await manualSync()
  } catch (e: any) {
    console.error('Erreur likePost :', e)
  }
}

const toggleSortLikes = () => {
  sortByLikes.value = !sortByLikes.value
}

// ---------- Commentaires ----------
const getCommentDraft = (docId?: string) =>
  docId ? commentDrafts.value[docId] || '' : ''

const setCommentDraft = (docId: string, value: string) => {
  commentDrafts.value = { ...commentDrafts.value, [docId]: value }
}

const addComment = async (post: Post) => {
  if (!post._id) return
  const draft = getCommentDraft(post._id).trim()
  if (!draft) return
  const db = initLocalDb()
  try {
    const fresh = await db.get(post._id)
    const norm = normalizePost(fresh)
    const newComment: Comment = {
      text: draft,
      created_at: new Date().toISOString(),
    }
    const updated: Post = {
      ...norm,
      comments: [...(norm.comments || []), newComment],
    }
    await db.put(updated as any)
    setCommentDraft(post._id, '')
    await fetchData()
    if (online.value) await manualSync()
  } catch (e: any) {
    console.error('Erreur addComment :', e)
  }
}

// ---------- Factory / Seed ----------
const seedFactory = async (n = 20) => {
  const db = initLocalDb()
  const t = Date.now()
  const docs: Post[] = Array.from({ length: n }).map((_, i) => ({
    _id: 'seed_' + (t + i),
    post_name: `Post ${i}`,
    post_content: `Contenu ${i}`,
    attributes: ['demo', `tag${i}`],
    created_at: new Date().toISOString(),
    likes: Math.floor(Math.random() * 10),
    comments: [],
  }))
  await (db as any).bulkDocs(docs)
  await fetchData()
  if (online.value) await manualSync()
}

// ---------- Top 10 les plus likés ----------
const fetchTop10Liked = async () => {
  const db = initLocalDb()
  try {
    const res = await (db as any).find({
      selector: { likes: { $gte: 0 } },
      sort: [{ likes: 'desc' }],
      limit: 10,
    })
    postsData.value = (res.docs as any[]).map(normalizePost)
  } catch (e: any) {
    console.error('Erreur fetchTop10Liked :', e)
  }
}

// ---------- Recherche (indexée) ----------
const onSearchInput = async () => {
  const db = initLocalDb()
  const term = searchTerm.value.trim()

  // si aucun terme et tri désactivé → on revient à la vue normale
  if (!term && !sortByLikes.value) {
    await fetchData()
    return
  }

  const selector: any = {}

  if (term) {
    // recherche partielle sur le nom
    selector.post_name = { $regex: term }
  }
  if (sortByLikes.value) {
    selector.likes = { $gte: 0 }
  }

  const query: any = {
    selector: Object.keys(selector).length ? selector : { _id: { $gte: null } },
  }

  if (sortByLikes.value) {
    query.sort = [{ likes: 'desc' }]
  }

  try {
    const res = await (db as any).find(query)
    postsData.value = (res.docs as any[]).map(normalizePost)
  } catch (e: any) {
    console.error('Erreur onSearchInput :', e)
  }
}

// ---------- Watchers ----------
watch(online, async value => {
  if (value) {
    await fullSync()
  }
})

// relance la recherche/tri automatiquement
watch([searchTerm, sortByLikes], () => {
  onSearchInput()
})

// ---------- Lifecycle ----------
onMounted(async () => {
  initLocalDb()
  initRemoteDb()
  await ensureIndexes()
  await manualSync()
  await fetchData()
  startLiveSync()
})
</script>

<template>
  <div class="container">
    <h1>CouchDB + Vue 3 (CRUD + Réplication + Indexation)</h1>

    <!-- Status + mode online/offline -->
    <section class="status-bar">
      <p v-if="loading" class="info">Chargement…</p>
      <p v-if="error" class="error">{{ error }}</p>
      <p>
        Mode :
        <span :class="online ? 'online' : 'offline'">
          {{ online ? 'ONLINE (sync activée)' : 'OFFLINE (local seulement)' }}
        </span>
      </p>
      <button class="btn small" @click="toggleOnline">
        Passer en mode {{ online ? 'OFFLINE' : 'ONLINE' }}
      </button>
      <button class="btn small secondary" @click="manualSync">
        Sync manuel (2 sens)
      </button>
    </section>

    <!-- Formulaire d'ajout -->
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
        @input="
          newPost.attributes = ($event.target as HTMLInputElement).value
            .split(',')
            .map(s => s.trim())
        "
      />
      <div class="row">
        <button @click="addPost">Ajouter</button>
        <button class="secondary" @click="seedFactory(20)">
          Générer 20 docs (factory)
        </button>
      </div>
    </div>

    <hr />

    <!-- Recherche + tri -->
    <section class="search-bar">
      <h2>Recherche & tri</h2>
      <input
        v-model="searchTerm"
        @input="onSearchInput"
        placeholder="Recherche par post_name…"
        type="text"
      />
      <p class="search-help">
        Tri actuel :
        <strong>{{ sortByLikes ? 'par likes (index DB)' : 'par date (created_at)' }}</strong>
      </p>
      <button class="btn small secondary" @click="toggleSortLikes">
        {{ sortByLikes ? 'Trier par date' : 'Trier par likes' }}
      </button>
      <button class="btn small secondary" @click="fetchData">
        Rafraîchir
      </button>
      <button class="btn small secondary" @click="fetchTop10Liked">
        Top 10 les plus likés
      </button>
    </section>

    <!-- Liste des documents -->
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
        <p class="meta">
          <small>Créé : {{ post.created_at }}</small>
          <span v-if="post.updated_at">
            • <small>Modifié : {{ post.updated_at }}</small>
          </span>
          • <small>Likes : {{ post.likes || 0 }}</small>
        </p>

        <div class="row">
          <button @click="startEdit(post)">Modifier</button>
          <button @click="deletePost(post._id, post._rev)">🗑️ Supprimer</button>
          <button class="secondary" @click="likePost(post)">❤️ Like</button>
        </div>

        <!-- Commentaires -->
        <div class="comments">
          <h3>Commentaires</h3>
          <p v-if="!post.comments || post.comments.length === 0" class="empty">
            Aucun commentaire.
          </p>
          <ul v-else class="comments-list">
            <li v-for="(c, idx) in post.comments" :key="idx">
              <span class="comment-text">{{ c.text }}</span>
              <span class="comment-date">({{ c.created_at }})</span>
            </li>
          </ul>

          <div v-if="post._id" class="comment-form">
            <input
              type="text"
              :placeholder="'Nouveau commentaire…'"
              :value="getCommentDraft(post._id)"
              @input="
                setCommentDraft(
                  post._id!,
                  ($event.target as HTMLInputElement).value
                )
              "
            />
            <button class="secondary small" type="button" @click="addComment(post)">
              Ajouter
            </button>
          </div>
        </div>
      </template>

      <!-- Mode édition -->
      <template v-else>
        <div class="editBox">
          <h3>Modifier le document</h3>
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
            @input="
              editingPost &&
                (editingPost.attributes = ($event.target as HTMLInputElement).value
                  .split(',')
                  .map(s => s.trim()))
            "
          />
          <div class="row">
            <button @click="updatePost">Enregistrer</button>
            <button @click="cancelEdit">Annuler</button>
          </div>
        </div>
      </template>
    </article>
  </div>
</template>

<style scoped>
/* ton CSS inchangé */
.container {
  padding: 1.5rem;
  color: white;
  max-width: 900px;
  margin: auto;
}
.status-bar {
  margin-bottom: 1rem;
}
.online {
  color: #4caf50;
  font-weight: bold;
}
.offline {
  color: #f44336;
  font-weight: bold;
}
.form {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
}
.search-bar {
  margin: 1rem 0;
}
.search-help {
  font-size: 0.85rem;
  opacity: 0.8;
}
input,
textarea {
  padding: 0.5rem;
  border-radius: 4px;
  border: none;
}
button {
  background: #42b883;
  color: white;
  padding: 0.5rem 0.8rem;
  border: none;
  cursor: pointer;
  border-radius: 4px;
}
button.secondary {
  background: #2f4858;
}
button.danger {
  background: #e53935;
}
button.small {
  font-size: 0.8rem;
  padding: 0.3rem 0.6rem;
}
button:hover {
  opacity: 0.9;
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
.meta {
  font-size: 0.8rem;
  opacity: 0.9;
}
.comments {
  margin-top: 0.75rem;
  padding-top: 0.5rem;
  border-top: 1px solid #333;
}
.comments-list {
  list-style: none;
  padding-left: 0;
}
.comment-text {
  margin-right: 0.3rem;
}
.empty {
  font-size: 0.85rem;
  opacity: 0.7;
}
.comment-form {
  display: flex;
  gap: 0.5rem;
  margin-top: 0.5rem;
}
</style>


