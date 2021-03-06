<template>
  <section class="section">
    <!-- main containers:
     .container  http://bulma.io/documentation/layout/container/
     .content    http://bulma.io/documentation/elements/content/
     .section    http://bulma.io/documentation/layout/section/
     .block      base/classes.sass (just adds 20px margin-bottom except for last)
     -->
    <form novalidate ref="form"
      name="formData" class="container signup"
      @submit.prevent="submit"
    >
      <div class="box centered" style="max-width:400px">
        <div class="level is-mobile">
          <div class="level-left">
            <div class="level-item">
              <p class="subtitle"><i18n>Sign Up</i18n></p>
            </div>
          </div>
          <div class="level-right">
            <p class="level-item content is-small">
              <a @click="forwardToLogin"><i18n>Have an account?</i18n></a>
            </p>
          </div>
        </div>
        <div class="field">
          <p class="control has-icon">
            <input id="name" class="input" name="name" v-model="name" @blur="checkName" @keypress="checkName" v-validate data-vv-rules="required|regex:^\S+$" placeholder="username" required autofocus>
            <span class="icon is-small"><i class="fa fa-user"></i></span>
          </p>
          <p class="help is-danger" v-if="errors.has('name')" id="badUsername">
            <i18n v-if="nameAvailable === false">name is unavailable</i18n>
            <i18n v-else-if="name && name.length > 0">cannot contain spaces</i18n>
          </p>
          <p v-else-if="nameAvailable" class="help is-success"><i18n>name is available</i18n></p>
        </div>
        <div class="field">
          <p class="control has-icon">
            <input class="input" id= "email" name="email" v-model="email" v-validate data-vv-rules="required|email" type="email" placeholder="email" required>
            <span class="icon is-small"><i class="fa fa-envelope"></i></span>
          </p>
          <i18n v-if="errors.has('email')" id="badEmail" class="help is-danger">Not an email</i18n>
        </div>
        <div class="field">
          <p class="control has-icon">
            <input class="input" id="password" name="password" v-model="password" v-validate data-vv-rules="required|min:7" placeholder="password" type="password" required>
            <span class="icon is-small"><i class="fa fa-lock"></i></span>
          </p>
          <i18n v-if="errors.has('password')" id="badPassword" class="help is-danger">Password must be at least 7 characters</i18n>
        </div>
        <div class="level is-mobile">
          <div class="level-left">
            <div class="level-item content is-small">
              <span id="serverMsg" class="help is-marginless" :class="[error ? 'is-danger' : 'is-success']">
                {{response}}
              </span>
            </div>
          </div>
          <div class="level-right">
            <div class="level-item is-narrow">
              <button class="button submit is-success" type="submit" :disabled="errors.any()">
                <i18n>Sign Up</i18n>
              </button>
            </div>
          </div>
        </div>
      </div>
    </form>
  </section>
</template>

<style>
.signup .level-item { margin-top: 10px; }
.signup .level.top-align { align-items: flex-start; }
</style>

<script>
import backend from '../js/backend'
import Vue from 'vue'
import _ from 'lodash'
import * as Events from '../../../shared/events'
import * as contracts from '../js/events'
import {namespace} from '../js/backend/hapi'
// TODO: fix all this
export default {
  name: 'SignUp',
  methods: {
    submit: async function () {
      try {
        // Do this mutation first in order to have events correctly save
        this.$store.commit('login', this.name)
        let user = new contracts.IdentityContract({
          authorizations: [Events.CanModifyAuths.dummyAuth()],
          attributes: [
            {name: 'name', value: this.name},
            {name: 'email', value: this.email},
            {name: 'picture', value: `${window.location.origin}/simple/images/128x128.png`}
          ]
        })
        await backend.subscribe(user.toHash())
        await backend.publishLogEntry(user.toHash(), user)
        let mailbox = new contracts.MailboxContract({
          authorizations: [Events.CanModifyAuths.dummyAuth(user.toHash())]
        })
        await backend.subscribe(mailbox.toHash())
        await backend.publishLogEntry(mailbox.toHash(), mailbox)
        let attribute = new Events.HashableIdentitySetAttribute({attribute: {name: 'mailbox', value: mailbox.toHash()}}, user.toHash())
        await backend.publishLogEntry(user.toHash(), attribute)
        await namespace.register(this.name, user.toHash())
        // TODO: Just add cryptographic magic
        this.response = 'success'
        this.$store.dispatch('login', {name: this.name, identityContractId: user.toHash()})
        if (this.$route.query.next) {
          setTimeout(() => {
            this.$router.push({path: this.$route.query.next})
          }, 1000)
        } else {
          this.$router.push({path: '/'})
        }
        this.error = false
      } catch (ex) {
        this.$store.commit('logout')
        console.log(ex)
        this.response = ex.toString()
        this.error = true
      }
    },
    forwardToLogin: function () {
      Vue.events.$emit('loginModal')
    },
    checkName: _.debounce(async function () {
      this.nameAvailable = null
      if (this.name && !this.errors.has('name')) {
        try {
          await namespace.lookup(this.name)
          this.nameAvailable = false
        } catch (ex) {
          if (ex.message === 'Not Found') {
            this.nameAvailable = true
          } else {
            console.log(ex)
            // TODO: Replace with w/e the ultimate global error notification solution is
            this.response = ex.toString()
            this.error = true
          }
        }
      }
    }, 700)
  },
  data () {
    return {
      error: false,
      response: '',
      nameAvailable: null,
      name: null,
      password: null,
      email: null
    }
  }
}
</script>
