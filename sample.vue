<template>
  <router-view :class="{ 'blur-bg' : blurBackground }" />
  <AppInsights />
</template>

<script>
// Quasar deps
import { createMetaMixin } from 'quasar'

// Vue deps
import { mapActions, mapGetters } from 'vuex'
import { useSSRContext, nextTick, defineAsyncComponent } from 'vue'

import { purgev2, trackConversion, AppConfig, theme } from '@utils'

const AppInsights = defineAsyncComponent(() => import('@components/social-plugins/AppInsights'))

export default {
  name: 'App',

  components: {
    AppInsights
  },

  computed: {
    ...mapGetters({
      isNewUser: 'app/isNewUser'
    }),

    blurBackground () {
      return this.$store.state.app.blurBackground
    }
  },

  data () {
    return {
      mounted: false
    }
  },

  mixins: [
    createMetaMixin(function () {
      return {
        title: 'Buy Diamonds Online'
      }
    })
  ],

  beforeCreate () {
    purgev2()
  },

  serverPrefetch () {
    return new Promise((resolve, reject) => {
      const experiments = this.$store.state.experiments.list
      const isNewUser = this.$q.cookies.get('new-user') === 'true' || this.$q.cookies.get('new-user') === null

      if (!this.$q.cookies.has('new-user')) {
        this.$q.cookies.set('new-user', isNewUser, {
          expires: 365,
          path: '/'
        })
      }

      this.$store.dispatch('app/setIsNewUser', isNewUser)

      for (let i = 0, l = experiments.length; i < l; i++) {
        const experiment = experiments[i]
        let variant = null

        // Disable experiment if set for new users only (only if cookie was not generated already)
        if (!isNewUser && experiment.newUsersOnly && !this.$q.cookies.has(experiment.cookie)) {
          this.$store.dispatch('experiments/deactivate', experiment.name)
          continue
        }

        if (!experiment.active) {
          continue
        }

        // Generate variant or read from cookie
        if (this.$q.cookies.has(experiment.cookie)) {
          variant = parseInt(this.$q.cookies.get(experiment.cookie))
        } else {
          variant = Math.random() < 0.5 ? 0 : 1

          this.$q.cookies.set(experiment.cookie, variant, {
            expires: 365,
            path: '/'
          })
        }

        // Update store
        this.$store.dispatch('experiments/setVariant', { name: experiment.name, variant: variant })
      }

      resolve()
    })
  },

  created () {
    const ssrContext = process.env.SERVER ? useSSRContext() : null

    // Restore dark mode
    this.evaluateDarkModeRules(ssrContext)

    if (!this.$q.cookies.has('_ssn')) {
      this.$q.cookies.set('_ssn', 'clientCookie-' + Math.random())
    }

    this.$store.dispatch('app/setSessionId', this.$q.cookies.get('_ssn'))

    if (ssrContext) {
      return
    }

    // document.body.classList.remove('noscript')
    return this.$store.dispatch('app/fetchUserInfo')
  },

  methods: {
    ...mapActions({
      setPopupDiscountExperimentData: 'app/setPopupDiscountExperimentData',
      setPopupDiscountTrackEventSent: 'app/setPopupDiscountTrackEventSent'
    }),

    evaluateDarkModeRules (ssrContext) {
      const isDark = theme.isDark(ssrContext, !this.$route.meta.isHomePage)

      if (this.$route.meta.supportsDarkTheme !== false) {
        nextTick(() => {
          return this.$q.dark.set(isDark)
        })
      }

      if (this.$route.meta.supportsDarkTheme === false) {
        nextTick(() => {
          this.$q.dark.set(false)
        })
      }
    }
  },

  mounted () {
    // Call the APP config and see if PWA is active
    AppConfig.fetch()

    // Collect experiments and fire up events (for noninteraction)
    const experiments = this.$store.state.experiments.list

    // const isNewUser = this.isNewUser

    for (let i = 0, l = experiments.length; i < l; i++) {
      const experiment = experiments[i]

      // Check platform and manually deactivate
      if (experiment.platform !== 'all') {
        if ((experiment.platform === 'desktop' && !this.$q.platform.is.desktop) || (experiment.platform === 'mobile' && !this.$q.platform.is.mobile)) {
          this.$store.dispatch('experiments/deactivate', experiment.name)
        }
      }

      if (!experiment.active) {
        continue
      }

      const experimentId = experiment.id
      const variant = experiment.variant // parseInt(this.$q.cookies.get(experiment.cookie))

      // Send the information about chosen variant to Google Optimize (unless disabled by experiment)
      if (!experiment.noInitialTrack) {
        trackConversion.sendOptimizerExperimentVariant(experimentId, variant)
      }

      if (experiment.nonInteraction) {
        this.sendTrackEvent(experiment.category, variant === 0 ? experiment.labelControl : experiment.labelTest, experiment.name, true)
      }
    }

    trackConversion.trackViewPage()
    this.mounted = true

    const then = performance.now()

    window.addEventListener('load', function () {
      console.log('window load event fired after', performance.now() - then)
    })
  },

  watch: {
    '$route.path' () {
      trackConversion.trackViewPage()
      this.evaluateDarkModeRules(null)
    }
  }
}
</script>
