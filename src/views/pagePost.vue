<template>
  <div>
    <h2>Posts</h2>
    <ul>
      <li v-for="post in postsData" :key="post._id">
        {{ post._id }} - {{ post.post_name }}
        <button @click="deleteData(post._id, post._rev)">Supprimer</button>
      </li>
    </ul>
    <h2>Nouveaux Posts</h2>
    <ul>
      <li v-for="post in newPosts" :key="post._id">
        {{ post._id }} - {{ post.post_name }}
        <button @click="deleteData(post._id, post._rev)">Supprimer</button>
      </li>
    </ul>
    <button @click="putDocument">Créer un nouveau post</button>
    <button @click="updateLocalDatabase">Synchroniser avec la base de données distante</button>
  </div>
</template>

<script>
import PouchDB from 'pouchdb'
import { onBeforeUnmount } from 'vue'

export default {
  data() {
    return {
      remoteDB: 'http://admin:Salutsalut2.@localhost:5984/post',
      postsData: [],
      db: null,
      newPosts: [],
      isSyncing: false
    }
  },

  methods: {
    initDatabase() {
      const localDB = 'post'
      const $db = new PouchDB(localDB)
      if ($db) {
        console.log('Connected to collection: ' + $db.name)
      } else {
        console.warn('Something went wrong')
      }
      this.db = $db
    },

    fetchData() {
      const db = ref(this.db).value
      if (db) {
        db.allDocs
          .bind(this)({
            include_docs: true,
            attachments: true
          })
          .then((result) => {
            console.log('fetchData success =>', result.rows)
            this.postsData = result.rows.map((row) => row.doc)
          })
          .catch((error) => {
            console.log('fetchData error', error)
          })
      }
    },

    getFakePost() {
      return {
        _id: v4(),
        post_name: 'post_name' + new Date().toISOString(),
        post_content: 'post_content',
        attributes: {
          creation_date: new Date().toISOString()
        }
      }
    },

    putDocument() {
      const document = this.getFakePost()
      const db = ref(this.db).value
      if (db) {
        db.put
          .bind(this)(document)
          .then(() => {
            console.log('Add ok')
            this.fetchData()
          })
          .catch((error) => {
            console.log('Add ko', error)
          })
      }
    },

    async deleteData(id, rev) {
      console.log('Call deleteData', rev)
      const db = ref(this.db).value
      if (!db) {
        console.log('Database not valid')
        return
      }
      try {
        await db.remove(id, rev)
        console.log('deleteData success')
        this.fetchData()
      } catch (error) {
        console.log('deleteData error', error)
      }
    },

    updateLocalDatabase() {
      const db = ref(this.db).value
      if (db) {
        db.replicate.from
          .bind(this)(this.remoteDB)
          .on('complete', () => {
            console.log('on replicate complete')
            this.fetchData()
          })
          .on('error', (error) => {
            console.log('error', error)
          })
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
    }
  }
}
</script>
