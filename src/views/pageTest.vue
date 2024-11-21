<template>
  <!-- Le template reste identique -->
  <div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold mb-4">Document tests</h1>

    <!-- Statut de synchronisation -->
    <div class="mb-4 p-4 rounded" :class="syncStatusClass">
      <p v-if="pendingChanges > 0" class="text-sm">
        {{ pendingChanges }} modification(s) en attente de synchronisation
      </p>
    </div>
    <!-- Contrôles de synchronisation -->
    <div class="mb-4 flex gap-2">
      <button
        @click="toggleSync"
        class="px-4 py-2 rounded"
        :class="isSyncEnabled ? 'bg-red-500 text-white' : 'bg-green-500 text-white'"
      >
        {{ isSyncEnabled ? 'Désactiver' : 'Activer' }} la synchronisation
      </button>
      <button
        @click="forceSync"
        class="bg-blue-500 text-white px-4 py-2 rounded"
        :disabled="!isSyncEnabled"
      >
        Synchroniser
      </button>
    </div>
    <p v-if="syncSuccessMessage" class="text-green-500">{{ syncSuccessMessage }}</p>

    <!-- historique des synchronisation -->
    <div v-if="syncHistory.length" class="mt-4">
      <h2 class="text-lg font-semibold">Historique de synchronisation</h2>
      <ul>
        <li v-for="(entry, index) in syncHistory" :key="index">
          {{ entry }}
        </li>
      </ul>
    </div>

    <!-- Liste des documents -->
    <div class="mb-6">
      <h2 class="text-xl font-semibold mb-2">Liste Personnages</h2>
      <div class="grid gap-4">
        <div v-for="doc in documents" :key="doc._id" class="border p-4 rounded">
          <div class="flex justify-between items-center">
            <div>
              <h3 class="font-bold">{{ doc.nom }}</h3>
              <p>Lvl: {{ doc.lvl }}</p>
              <p>Race: {{ doc.race }}</p>
              <p class="text-sm text-gray-500">
                Dernière modification:
                {{ new Date(doc.updatedAt || doc.createdAt).toLocaleString() }}
              </p>
            </div>
            <div>
              <button @click="editDoc(doc)" class="bg-blue-500 text-white px-3 py-1 rounded mr-2">
                Modifier
              </button>
              <button @click="deleteDoc(doc)" class="bg-red-500 text-white px-3 py-1 rounded">
                Supprimer
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Formulaire d'ajout/modification -->
    <div class="mb-6 border p-4 rounded">
      <h2 class="text-xl font-semibold mb-2">
        {{ isEditing ? 'Modifier le personnage' : 'Créer un personnage' }}
      </h2>
      <form @submit.prevent="submitForm" class="space-y-4">
        <div>
          <label class="block mb-1">Nom</label>
          <input v-model="currentDoc.nom" type="text" required class="border p-2 w-full rounded" />
        </div>
        <div>
          <label class="block mb-1">Lvl</label>
          <input
            v-model="currentDoc.lvl"
            type="number"
            required
            class="border p-2 w-full rounded"
          />
        </div>
        <div>
          <label class="block mb-1">Race</label>
          <select v-model="currentDoc.race" required class="border p-2 w-full rounded">
            <option value="Mort-vivant">Mort-vivant</option>
            <option value="Humain">Humain</option>
            <option value="Gnome">Gnome</option>
            <option value="Elf">Elf</option>
            <option value="Orc">Orc</option>
          </select>
        </div>
        <div class="flex gap-2">
          <button type="submit" class="bg-green-500 text-white px-4 py-2 rounded">
            {{ isEditing ? 'Mettre à jour' : 'Ajouter' }}
          </button>
          <button
            type="button"
            @click="generateFakeDoc"
            class="bg-purple-500 text-white px-4 py-2 rounded"
          >
            Générer démo
          </button>
        </div>
      </form>
    </div>
  </div>
</template>

<script>
import PouchDB from 'pouchdb'
import { onBeforeUnmount } from 'vue'

export default {
  data() {
    return {
      localDb: null,
      remoteDb: null,
      sync: null,
      documents: [],
      currentDoc: {
        nom: '',
        lvl: '1',
        race: ''
      },
      isEditing: false,
      editingId: null,
      syncStatus: 'idle', // idle, syncing, error
      isSyncEnabled: true,
      pendingChanges: 0,
      lastSync: null,
      previousDoc: null,
      syncSuccessMessage: '',
      syncHistory: []
    }
  },

  computed: {
    syncStatusClass() {
      return {
        'bg-gray-100': this.syncStatus === 'idle',
        'bg-blue-100': this.syncStatus === 'syncing',
        'bg-red-100': this.syncStatus === 'error'
      }
    }
  },

  methods: {
    initDatabases() {
      // Base de données locale
      this.localDb = new PouchDB('datatest')
      // Base de données distante
      this.remoteDb = new PouchDB('http://127.0.0.1:5984/datatest', {
        fetch: function (url, opts) {
          opts.headers.set('Authorization', 'Basic ' + btoa('admin:Salutsalut2.'))
          return PouchDB.fetch(url, opts)
        }
      })

      // permet la sync direct
      // this.setupSync()

      // Écouter les changements locaux
      this.localDb
        .changes({
          since: 'now',
          live: true,
          include_docs: true
        })
        .on('change', (change) => {
          this.handleDatabaseChange(change)
        })
    },

    setupSync() {
      if (this.sync) {
        this.sync.cancel()
      }

      if (!this.isSyncEnabled) {
        return
      }

      this.syncStatus = 'syncing'
      this.sync = this.localDb
        .sync(this.remoteDb, {
          live: true,
          retry: true
        })
        .on('change', (info) => {
          console.log('Sync change:', info)
          this.fetchDocuments()
          this.updatePendingChanges()
        })
        .on('paused', () => {
          this.syncStatus = 'idle'
          this.lastSync = new Date()
        })
        .on('active', () => {
          this.syncStatus = 'syncing'
        })
        .on('error', (err) => {
          console.error('Sync error:', err)
          this.syncStatus = 'error'
        })
        .on('complete', () => {
          console.log('Synchronisation terminée avec succès')
        })
    },

    toggleSync() {
      this.isSyncEnabled = !this.isSyncEnabled
      if (this.isSyncEnabled) {
        this.setupSync()
      } else if (this.sync) {
        this.sync.cancel()
        this.syncStatus = 'idle'
      }
    },

    async forceSync() {
      try {
        this.syncStatus = 'syncing'
        await this.localDb.sync(this.remoteDb)
        this.syncStatus = 'idle'
        this.lastSync = new Date()
        await this.fetchDocuments()
        this.syncSuccessMessage = 'Synchronisation réussie !'
        this.addSyncHistory()
        //set un timer pour afficher le message
        setTimeout(() => {
          this.syncSuccessMessage = ''
        }, 3000)
      } catch (error) {
        console.error('Force sync error:', error)
        this.syncStatus = 'error'
      }
    },

    async updatePendingChanges() {
      try {
        const result = await this.localDb.changes({
          since: this.lastSync ? this.lastSync.toISOString() : 'now'
        })
        this.pendingChanges = result.results.length
      } catch (error) {
        console.error('Error checking changes:', error)
      }
    },

    handleDatabaseChange(change) {
      if (change.deleted) {
        this.documents = this.documents.filter((doc) => doc._id !== change.id)
      } else {
        const index = this.documents.findIndex((doc) => doc._id === change.id)
        if (index !== -1) {
          this.documents.splice(index, 1, change.doc)
        } else {
          this.documents.push(change.doc)
        }
      }
    },

    async fetchDocuments() {
      try {
        const result = await this.localDb.allDocs({ include_docs: true, descending: true })
        this.documents = result.rows.map((row) => row.doc)
      } catch (error) {
        console.error('Erreur lors de la récupération des documents:', error)
      }
    },

    generateFakeDoc() {
      const races = ['Orc', 'Humain', 'Mort-vivant', 'Elf', 'Gnome']
      const vetements = ['Polo', 'Franc', 'Himi', 'Huper', 'Golum']
      this.currentDoc = {
        nom:
          vetements[Math.floor(Math.random() * vetements.length)] +
          '' +
          Math.floor(Math.random() * 100),
        lvl: Math.floor(Math.random() * 100) + 10,
        race: races[Math.floor(Math.random() * races.length)]
      }
    },

    async submitForm() {
      try {
        const timestamp = new Date().toISOString()
        if (this.isEditing) {
          const doc = await this.localDb.get(this.editingId)
          await this.localDb.post({
            ...doc,
            nom: this.currentDoc.nom,
            lvl: this.currentDoc.lvl,
            race: this.currentDoc.race,
            updatedAt: timestamp
          })
        } else {
          this.previousDoc = { ...this.currentDoc }
          await this.localDb.post({
            ...this.currentDoc,
            createdAt: timestamp,
            updatedAt: timestamp
          })
        }
        await this.fetchDocuments()
        this.resetForm()
      } catch (error) {
        console.error('Erreur lors de la sauvegarde:', error)
      }
    },

    editDoc(doc) {
      this.isEditing = true
      this.editingId = doc._id
      this.currentDoc = {
        nom: doc.nom,
        lvl: doc.lvl,
        race: doc.race
      }
    },

    async deleteDoc(doc) {
      try {
        await this.localDb.remove(doc)
        await this.fetchDocuments()
      } catch (error) {
        console.error('Erreur lors de la suppression:', error)
      }
    },

    undoCreate() {
      this.currentDoc = { ...this.previousDoc }
      this.isEditing = false
      this.editingId = null
      this.syncSuccessMessage = ''
    },

    resetForm() {
      this.currentDoc = {
        nom: '',
        lvl: '1',
        race: ''
      }
      this.isEditing = false
      this.editingId = null
    },

    cleanup() {
      if (this.sync) {
        this.sync.cancel()
      }
    },

    async updateDistantDatabase() {
      try {
        await this.remoteDb.bulkDocs(this.documents)
        console.log('Base de données distante mise à jour avec succès')
      } catch (error) {
        console.error('Erreur lors de la mise à jour de la base de données distante:', error)
      }
    },

    async watchRemoteDatabase() {
      this.remoteDb
        .changes({
          since: 'now',
          live: true,
          include_docs: true
        })
        .on('change', (change) => {
          if (change.deleted) {
            this.documents = this.documents.filter((doc) => doc._id !== change.id)
          } else {
            const index = this.documents.findIndex((doc) => doc._id === change.id)
            if (index !== -1) {
              this.documents[index] = change.doc
            } else {
              this.documents.push(change.doc)
            }
          }
        })
        .on('error', (error) => {
          console.error('Erreur lors de la surveillance de la base de données distante:', error)
        })
    },

    addSyncHistory() {
      const timestamp = new Date().toLocaleString()
      this.syncHistory.push(`Synchronisation réussie de à ${timestamp}`)
    },

    cancelSync() {
      if (this.sync) {
        this.sync.cancel()
        this.syncStatus = 'idle'
        this.syncHistory.push(`Synchronisation annulée à ${new Date().toLocaleString()}`)
      }
    }
  },

  mounted() {
    this.initDatabases()
    this.watchRemoteDatabase()
  },

  beforeUnmount() {
    this.cleanup()
  }
}
</script>
