<template>
  <div class="container" id="app">
    <h1>Idea Bin</h1>
    <div class="input-group mb-3">
      <input type="text" class="form-control" placeholder="Your idea" v-model="textInput" v-on:keyup.enter="addIdea" autofocus/>
      <div class="input-group-append">
        <button class="btn btn-outline-secondary" type="button" v-on:click="addIdea">Add</button>
      </div>
    </div>
    <ul class="list-group" v-if="Object.keys(ideas).length !== 0">
      <li class="list-group-item idea-item" v-for="(idea, key) in ideas" v-bind:key="key">{{idea.text}}
        <div class="float-right" v-on:click="removeIdea(key)">
         <i class="fas fa-times delete-icon"></i>
        </div>
        <a href="#" class="badge badge-info" v-if="idea.tags.length > 0" v-for="tag in idea.tags.split(' ')" v-bind:key="tag">
          {{tag}}
        </a>
      </li>
    </ul>
    <p id="welcome-text" v-else>
      Idea Bin is an easy way to keep track of your ideas while keeping them private.

      Feel free to bookmark this page so that you can quickly jot down your ideas as they come to you.

      If you're curious: the URL consists of the id of your "bin" and your aes256 key.
      Both of these things are in the hash—that way they're never sent to our servers.

      Of course, you don't have to take my word for it; you should check yourself by reviewing the source code at <a href="https://github.com/aaronduino/idea-bin">https://github.com/aaronduino/idea-bin</a>
    </p>
  </div>
</template>

<script>
import * as firebase from 'firebase'
import Vue from 'vue'
import aesjs from 'aes-js'

const hash = window.location.hash.slice(1)

function uuidv4 () {
  return ([1e7] + -1e3 + -4e3 + -8e3 + -1e11).replace(/[018]/g, c =>
    (c ^ crypto.getRandomValues(new Uint8Array(1))[0] & 15 >> c / 4).toString(16)
  )
}

// from https://stackoverflow.com/a/2450976/9436888
function shuffle (array) {
  var currentIndex = array.length
  var temporaryValue, randomIndex

  // While there remain elements to shuffle...
  while (currentIndex !== 0) {
    // Pick a remaining element...
    randomIndex = Math.floor(Math.random() * currentIndex)
    currentIndex -= 1

    // And swap it with the current element.
    temporaryValue = array[currentIndex]
    array[currentIndex] = array[randomIndex]
    array[randomIndex] = temporaryValue
  }

  return array
}

if (!hash) {
  var buf = new Uint8Array(32)
  window.crypto.getRandomValues(buf)
  const aesKey = aesjs.utils.hex.fromBytes(buf)

  window.location.hash = `/${uuidv4()}/${aesKey}`
}

var config = {
  apiKey: 'AIzaSyDKIWHp6l7t-bkKAb3Jk9nFwftNsu8fdC0',
  authDomain: 'idea-bin.firebaseapp.com',
  databaseURL: 'https://idea-bin.firebaseio.com',
  projectId: 'idea-bin',
  storageBucket: 'idea-bin.appspot.com',
  messagingSenderId: '944635474741'
}
firebase.initializeApp(config)

function encrypt (str) {
  var text = str
  if (text.length % 16 !== 0) {
    text += ' '.repeat(16 - (text.length % 16))
  }

  const textBytes = aesjs.utils.utf8.toBytes(text)

  var iv = new Uint8Array(16)
  window.crypto.getRandomValues(iv)
  const ivHex = aesjs.utils.hex.fromBytes(iv)

  // eslint-disable-next-line
  const aesCbc = new aesjs.ModeOfOperation.cbc(aesKey, iv)
  const encryptedBytes = aesCbc.encrypt(textBytes)
  const encryptedHex = aesjs.utils.hex.fromBytes(encryptedBytes)

  return {encryptedHex: encryptedHex, ivHex: ivHex}
}

function decrypt (encryptedData) {
  const ivHex = encryptedData.ivHex
  const iv = aesjs.utils.hex.toBytes(ivHex)

  // eslint-disable-next-line
  const aesCbc = new aesjs.ModeOfOperation.cbc(aesKey, iv)

  const encryptedHex = encryptedData.encryptedHex
  const encryptedBytes = aesjs.utils.hex.toBytes(encryptedHex)

  const textBytes = aesCbc.decrypt(encryptedBytes)
  const text = aesjs.utils.utf8.fromBytes(textBytes)

  return text.trim()
}

const [, uuid, aesKeyHex] = window.location.hash.slice(1).split('/')
const aesKey = aesjs.utils.hex.toBytes(aesKeyHex)

var database = firebase.database()
var firebaseIdeasRef = database.ref(`/ideas/${uuid}`)

export default {
  name: 'App',
  data () {
    return {
      textInput: '',
      ideas: {}
    }
  },
  methods: {
    addIdea: function () {
      const words = this.textInput.split(' ')
      const obj = {
        text: words.filter(word => !word.startsWith('#')).join(' '),
        tags: words.filter(word => word.startsWith('#')).join(' ').trim()
      }
      const encryptedObj = {
        text: encrypt(obj.text),
        tags: encrypt(obj.tags)
      }

      const id = firebaseIdeasRef.push(encryptedObj).key
      this.ideas[id] = obj

      this.textInput = ''
    },
    removeIdea: function (key) {
      firebaseIdeasRef.child(key).remove()
      Vue.delete(this.ideas, key)
    }
  },
  mounted () {
    firebase.auth().signInAnonymously().catch(function (error) {
      var errorCode = error.code
      var errorMessage = error.message

      console.error(errorCode, errorMessage)
    })

    firebase.auth().onAuthStateChanged(function (user) {
      if (user) {
        firebaseIdeasRef.once('value').then(function (snapshot) {
          var encryptedIdeas = snapshot.val() || {}
          var ideasList = []
          for (var key in encryptedIdeas) {
            if (encryptedIdeas.hasOwnProperty(key)) {
              const encryptedIdea = encryptedIdeas[key]
              ideasList.push([key, {
                text: decrypt(encryptedIdea.text),
                tags: decrypt(encryptedIdea.tags)
              }])
            }
          }
          shuffle(ideasList).forEach(function ([key, value]) {
            Vue.set(this.ideas, key, value)
          }.bind(this))
        }.bind(this))
      } else {
        // User is signed out.
      }
    }.bind(this))
  }
}

</script>

<style scoped>

.badge {
  margin-left: 0.25em;
}

.delete-icon {
  position: relative;
  top: 0.2rem;
  float: right;

  color: #999;

  visibility: hidden;
}

.idea-item:hover > div > .delete-icon {
  visibility: visible;
}

.delete-icon:hover {
  color: initial;
}

#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
  margin-bottom: 60px;
}

#welcome-text {
  text-align: left;
  white-space: pre-line;
}
</style>
