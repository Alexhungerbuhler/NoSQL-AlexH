<template>
  <div class="PageTest">
    <h1>InfraDon2</h1>
    <p>This is an about page</p>
    <p>Counter: {{ datas }}</p>
    <button type="button" role="button" @click="inc">+1</button>
  </div>
</template>

<script lang="ts">
export default {
  data() {
    return {
      datas: 0, // initialize the counter with 0 or any value you prefer
      databaseReference: null,
      storage: null
    }
  },

  methods: {
    inc() {
      this.datas++ // increment the counter
    },

    initDatabase() {
      const db = new PouchDB('http://admin:Salutsalut2.@localhost:5984/datatest')
      if (db) {
        console.log("Connected to collection 'post'")
      } else {
        console.warn('Something went wrong')
      }
      this.storage = db
    },

    fetchData() {
      // fetch data from your database here
    }
  },

  mounted() {
    this.initDatabase() // called when the component is mounted
  }
}
</script>

<style>
@media (min-width: 1024px) {
  .about {
    min-height: 100vh;
    display: flex;
    align-items: center;
    flex-direction: column;
    justify-content: center;
  }

  button {
    margin-top: 20px;
    padding: 10px 20px;
    font-size: 16px;
  }
}
</style>
