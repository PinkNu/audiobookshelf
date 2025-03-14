<template>
  <div id="authentication-settings">
    <app-settings-content :header-text="$strings.HeaderAuthentication">
      <div class="w-full border border-white/10 rounded-xl p-4 my-4 bg-primary/25">
        <div class="flex items-center">
          <ui-checkbox v-model="enableLocalAuth" checkbox-bg="bg" />
          <p class="text-lg pl-4">{{ $strings.HeaderPasswordAuthentication }}</p>
        </div>
      </div>
      <div class="w-full border border-white/10 rounded-xl p-4 my-4 bg-primary/25">
        <div class="flex items-center">
          <ui-checkbox v-model="enableOpenIDAuth" checkbox-bg="bg" />
          <p class="text-lg pl-4">{{ $strings.HeaderOpenIDConnectAuthentication }}</p>
          <ui-tooltip :text="$strings.LabelClickForMoreInfo" class="inline-flex ml-2">
            <a href="https://www.audiobookshelf.org/guides/oidc_authentication" target="_blank" class="inline-flex">
              <span class="material-icons text-xl w-5 text-gray-200">help_outline</span>
            </a>
          </ui-tooltip>
        </div>

        <transition name="slide">
          <div v-if="enableOpenIDAuth" class="flex flex-wrap pt-4">
            <div class="w-full flex items-center mb-2">
              <div class="flex-grow">
                <ui-text-input-with-label ref="issuerUrl" v-model="newAuthSettings.authOpenIDIssuerURL" :disabled="savingSettings" :label="'Issuer URL'" />
              </div>
              <div class="w-36 mx-1 mt-[1.375rem]">
                <ui-btn class="h-[2.375rem] text-sm inline-flex items-center justify-center w-full" type="button" :padding-y="0" :padding-x="4" @click.stop="autoPopulateOIDCClick">
                  <span class="material-icons text-base">auto_fix_high</span>
                  <span class="whitespace-nowrap break-keep pl-1">Auto-populate</span></ui-btn
                >
              </div>
            </div>

            <ui-text-input-with-label ref="authorizationUrl" v-model="newAuthSettings.authOpenIDAuthorizationURL" :disabled="savingSettings" :label="'Authorize URL'" class="mb-2" />

            <ui-text-input-with-label ref="tokenUrl" v-model="newAuthSettings.authOpenIDTokenURL" :disabled="savingSettings" :label="'Token URL'" class="mb-2" />

            <ui-text-input-with-label ref="userInfoUrl" v-model="newAuthSettings.authOpenIDUserInfoURL" :disabled="savingSettings" :label="'Userinfo URL'" class="mb-2" />

            <ui-text-input-with-label ref="jwksUrl" v-model="newAuthSettings.authOpenIDJwksURL" :disabled="savingSettings" :label="'JWKS URL'" class="mb-2" />

            <ui-text-input-with-label ref="logoutUrl" v-model="newAuthSettings.authOpenIDLogoutURL" :disabled="savingSettings" :label="'Logout URL'" class="mb-2" />

            <ui-text-input-with-label ref="openidClientId" v-model="newAuthSettings.authOpenIDClientID" :disabled="savingSettings" :label="'Client ID'" class="mb-2" />

            <ui-text-input-with-label ref="openidClientSecret" v-model="newAuthSettings.authOpenIDClientSecret" :disabled="savingSettings" :label="'Client Secret'" class="mb-2" />

            <ui-multi-select ref="redirectUris" v-model="newAuthSettings.authOpenIDMobileRedirectURIs" :items="newAuthSettings.authOpenIDMobileRedirectURIs" :label="$strings.LabelMobileRedirectURIs" class="mb-2" :menuDisabled="true" :disabled="savingSettings" />
            <p class="pl-4 text-sm text-gray-300 mb-2" v-html="$strings.LabelMobileRedirectURIsDescription" />

            <ui-text-input-with-label ref="buttonTextInput" v-model="newAuthSettings.authOpenIDButtonText" :disabled="savingSettings" :label="$strings.LabelButtonText" class="mb-2" />

            <div class="flex items-center pt-1 mb-2">
              <div class="w-44">
                <ui-dropdown v-model="newAuthSettings.authOpenIDMatchExistingBy" small :items="matchingExistingOptions" :label="$strings.LabelMatchExistingUsersBy" :disabled="savingSettings" />
              </div>
              <p class="pl-4 text-sm text-gray-300 mt-5">{{ $strings.LabelMatchExistingUsersByDescription }}</p>
            </div>

            <div class="flex items-center py-4 px-1">
              <ui-toggle-switch labeledBy="auto-redirect-toggle" v-model="newAuthSettings.authOpenIDAutoLaunch" :disabled="savingSettings" />
              <p id="auto-redirect-toggle" class="pl-4 whitespace-nowrap">{{ $strings.LabelAutoLaunch }}</p>
              <p class="pl-4 text-sm text-gray-300" v-html="$strings.LabelAutoLaunchDescription" />
            </div>

            <div class="flex items-center py-4 px-1">
              <ui-toggle-switch labeledBy="auto-register-toggle" v-model="newAuthSettings.authOpenIDAutoRegister" :disabled="savingSettings" />
              <p id="auto-register-toggle" class="pl-4 whitespace-nowrap">{{ $strings.LabelAutoRegister }}</p>
              <p class="pl-4 text-sm text-gray-300">{{ $strings.LabelAutoRegisterDescription }}</p>
            </div>
          </div>
        </transition>
      </div>
      <div class="w-full flex items-center justify-end p-4">
        <ui-btn color="success" :padding-x="8" small class="text-base" :loading="savingSettings" @click="saveSettings">{{ $strings.ButtonSave }}</ui-btn>
      </div>
    </app-settings-content>
  </div>
</template>

<script>
export default {
  async asyncData({ store, redirect, app }) {
    if (!store.getters['user/getIsAdminOrUp']) {
      redirect('/')
      return
    }

    const authSettings = await app.$axios.$get('/api/auth-settings').catch((error) => {
      console.error('Failed', error)
      return null
    })
    if (!authSettings) {
      redirect('/config')
      return
    }
    return {
      authSettings
    }
  },
  data() {
    return {
      enableLocalAuth: false,
      enableOpenIDAuth: false,
      savingSettings: false,
      newAuthSettings: {}
    }
  },
  computed: {
    authMethods() {
      return this.authSettings.authActiveAuthMethods || []
    },
    matchingExistingOptions() {
      return [
        {
          text: 'Do not match',
          value: null
        },
        {
          text: 'Match by email',
          value: 'email'
        },
        {
          text: 'Match by username',
          value: 'username'
        }
      ]
    }
  },
  methods: {
    autoPopulateOIDCClick() {
      if (!this.newAuthSettings.authOpenIDIssuerURL) {
        this.$toast.error('Issuer URL required')
        return
      }
      // Remove trailing slash
      let issuerUrl = this.newAuthSettings.authOpenIDIssuerURL
      if (issuerUrl.endsWith('/')) issuerUrl = issuerUrl.slice(0, -1)

      // If the full config path is on the issuer url then remove it
      if (issuerUrl.endsWith('/.well-known/openid-configuration')) {
        issuerUrl = issuerUrl.replace('/.well-known/openid-configuration', '')
        this.newAuthSettings.authOpenIDIssuerURL = this.newAuthSettings.authOpenIDIssuerURL.replace('/.well-known/openid-configuration', '')
      }

      this.$axios
        .$get(`/auth/openid/config?issuer=${issuerUrl}`)
        .then((data) => {
          if (data.issuer) this.newAuthSettings.authOpenIDIssuerURL = data.issuer
          if (data.authorization_endpoint) this.newAuthSettings.authOpenIDAuthorizationURL = data.authorization_endpoint
          if (data.token_endpoint) this.newAuthSettings.authOpenIDTokenURL = data.token_endpoint
          if (data.userinfo_endpoint) this.newAuthSettings.authOpenIDUserInfoURL = data.userinfo_endpoint
          if (data.end_session_endpoint) this.newAuthSettings.authOpenIDLogoutURL = data.end_session_endpoint
          if (data.jwks_uri) this.newAuthSettings.authOpenIDJwksURL = data.jwks_uri
        })
        .catch((error) => {
          console.error('Failed to receive data', error)
          const errorMsg = error.response?.data || 'Unknown error'
          this.$toast.error(errorMsg)
        })
    },
    validateOpenID() {
      let isValid = true
      if (!this.newAuthSettings.authOpenIDIssuerURL) {
        this.$toast.error('Issuer URL required')
        isValid = false
      }
      if (!this.newAuthSettings.authOpenIDAuthorizationURL) {
        this.$toast.error('Authorize URL required')
        isValid = false
      }
      if (!this.newAuthSettings.authOpenIDTokenURL) {
        this.$toast.error('Token URL required')
        isValid = false
      }
      if (!this.newAuthSettings.authOpenIDUserInfoURL) {
        this.$toast.error('Userinfo URL required')
        isValid = false
      }
      if (!this.newAuthSettings.authOpenIDJwksURL) {
        this.$toast.error('JWKS URL required')
        isValid = false
      }
      if (!this.newAuthSettings.authOpenIDClientID) {
        this.$toast.error('Client ID required')
        isValid = false
      }
      if (!this.newAuthSettings.authOpenIDClientSecret) {
        this.$toast.error('Client Secret required')
        isValid = false
      }

      function isValidRedirectURI(uri) {
        // Check for somestring://someother/string
        const pattern = new RegExp('^\\w+://[\\w\\.-]+$', 'i')
        return pattern.test(uri)
      }

      const uris = this.newAuthSettings.authOpenIDMobileRedirectURIs
      if (uris.includes('*') && uris.length > 1) {
        this.$toast.error('Mobile Redirect URIs: Asterisk (*) must be the only entry if used')
        isValid = false
      } else {
        uris.forEach((uri) => {
          if (uri !== '*' && !isValidRedirectURI(uri)) {
            this.$toast.error(`Mobile Redirect URIs: Invalid URI ${uri}`)
            isValid = false
          }
        })
      }
      return isValid
    },
    async saveSettings() {
      if (!this.enableLocalAuth && !this.enableOpenIDAuth) {
        this.$toast.error('Must have at least one authentication method enabled')
        return
      }

      if (this.enableOpenIDAuth && !this.validateOpenID()) {
        return
      }

      this.newAuthSettings.authActiveAuthMethods = []
      if (this.enableLocalAuth) this.newAuthSettings.authActiveAuthMethods.push('local')
      if (this.enableOpenIDAuth) this.newAuthSettings.authActiveAuthMethods.push('openid')

      this.savingSettings = true
      this.$axios
        .$patch('/api/auth-settings', this.newAuthSettings)
        .then((data) => {
          this.$store.commit('setServerSettings', data.serverSettings)
          if (data.updated) {
            this.$toast.success('Server settings updated')
          } else {
            this.$toast.info(this.$strings.MessageNoUpdatesWereNecessary)
          }
        })
        .catch((error) => {
          console.error('Failed to update server settings', error)
          this.$toast.error('Failed to update server settings')
        })
        .finally(() => {
          this.savingSettings = false
        })
    },
    init() {
      this.newAuthSettings = {
        ...this.authSettings
      }
      this.enableLocalAuth = this.authMethods.includes('local')
      this.enableOpenIDAuth = this.authMethods.includes('openid')
    }
  },
  mounted() {
    this.init()
  }
}
</script>

<style>
#authentication-settings code {
  font-size: 0.8rem;
  border-radius: 6px;
  background-color: rgb(82, 82, 82);
  color: white;
  padding: 2px 4px;
  white-space: nowrap;
}
</style>