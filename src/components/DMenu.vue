<template lang="pug">
transition(appear enter-active-class="animated zoomIn" leave-active-class="animated zoomOut")
  q-input(for="search" v-model="term" @update:model-value="searchTerm" :placeholder="$t('menu.search')" :debounce="300")
    template(v-slot:prepend)
      q-icon.q-ml-sm(name="search")
    template(v-slot:append)
      q-icon.cursor-pointer.clear(v-if="term" name="clear" @click="clearSearchTerm")
q-scroll-area#menu.bg-grey-2(:visible="true")
  .row.flex-center.bg-white(style="height: 115px;")
    .col-5
      img.q-mr-md(src="logo/bootgly-logo-temp1.png" alt="Quasar Logo" width="75" height="75" style="float: right;")
    .col-7
      .text-weight-medium Bootgly v0.x.x
      div Documentation
      q-btn.q-mt-xs.q-pl-sm.support(
        dense no-caps outline
        size="sm" :label="$t('system.support')" icon="fas fa-donate"
        @click="openURL('https://github.com/sponsors/bootgly/')")
  q-separator.separator.list
  .row.bg-white
    .col.text-center
      q-btn-group(flat)
        q-btn(icon="fab fa-github" size="sm" @click="openURL('https://github.com/bootgly/bootgly-php-framework/')" aria-label="Bootgly Github")
          q-tooltip Github
        q-btn(icon="fas fa-comments" size="sm" @click="openURL('https://github.com/bootgly/bootgly-php-framework/discussions/')" aria-label="Bootgly Forum")
          q-tooltip Discussions
        q-btn(icon="fas fa-comment" size="sm" @click="openURL('https://t.me/bootgly/')" aria-label="Bootgly Chat")
          q-tooltip Chat
        q-btn(icon="fas fa-at" size="sm" @click="openURL('mailto:public@bootgly.com')" aria-label="Bootgly Public Email")
          q-tooltip Email
  q-separator.separator.list
  q-list(no-border link inset-delimiter)
    q-item(to="/" exact)
      q-item-section(side)
        q-icon(name="home")
      q-item-section {{ $t('menu.home') }}
    q-item(to="/changelog")
      q-item-section(side)
        q-icon(name="assignment")
      q-item-section {{ $t('menu.changelog') }}
    q-item(to="/sponsor")
      q-item-section(side)
        q-icon(name="favorite" color="red")
      q-item-section {{ $t('menu.sponsor') }}
  q-separator.separator.list
  q-list(v-if="items !== null && items.constructor === Array && items.length > 0" no-border link inset-delimiter)
    template(v-for="(item, index) in items" :key="index")
      q-item-section.label.header.sticky(v-if="item.meta.menu.header")
        q-item-label(header)
          q-icon(:name="item.meta.menu.header.icon" size="1.5rem")
          span {{ getMenuItemHeader(item) }}
        q-separator.separator.partial
      q-item-section(v-if="item.meta.menu.subheader")
        q-item-label.label.subheader(header) {{ getMenuItemSubheader(item) }}
      q-item(:to="item.path" v-show="matches[index] || !matches")
        q-item-section(side)
          q-icon(v-if="item.meta.icon" :name="item.meta.icon")
        q-item-section {{ getMenuItemLabel(item, index) }}
        q-item-section.page-status(side)
          q-icon.float-right(size="xs" :name="getPageStatusIcon(item.meta.status)")
          q-tooltip {{ getPageStatusTooltip(item.meta.status) }}
      q-separator(v-if="item.meta.menu.separator" :class="'separator' + item.meta.menu.separator")
</template>

<script>
import { openURL, scroll } from 'quasar'
const { getScrollTarget, setVerticalScrollPosition } = scroll

import menu from 'src/i18n/menu.hjson'

export default {
  name: 'DMenu',

  data () {
    return {
      loaded: false,

      term: null,
      matches: false
    }
  },
  computed: {},

  methods: {
    openURL,

    // TODO highlight terms found in menu item and page content?
    // TODO search result count in input bottom?
    searchTerm (term) {
      if (term.length > 1) {
        term = term.toLowerCase()

        const locale = this.$q.localStorage.getItem('setting.language')

        this.matches = []

        for (const [index, route] of this.items.entries()) {
          // ! Search in Menu item label
          // TODO
          // ! Search in i18n/menu.hjson
          // * Search in current language
          this.matches[index] = menu[locale][index].indexOf(term) !== -1
          // * Fallback to search in en-US
          if (this.matches[index] === false && locale !== 'en-US') {
            this.matches[index] = menu['en-US'][index].indexOf(term) !== -1
          }
          // ! Search in Page content
          if (this.matches[index] === false) {
            // ? Search in Page texts (overview, showcases?, changelog?)
            // * Search in current language
            this.matches[index] = this.searchTermInI18nTexts(route.path, term, locale)
            // * Fallback to search in en-US
            if (this.matches[index] === false && locale !== 'en-US') {
              this.matches[index] = this.searchTermInI18nTexts(route.path, term, 'en-US')
            }
            // ? Search in Page codes (overview, showcases?)
            // * Search in current language
            if (this.matches[index] === false) {
              this.matches[index] = this.searchTermInI18nCodes(route.path, term, locale)
            }
            // * Fallback to search in en-US
            if (this.matches[index] === false && locale !== 'en-US') {
              this.matches[index] = this.searchTermInI18nCodes(route.path, term, 'en-US')
            }
          }
        }
      } else {
        this.matches = false
      }
    },
    searchTermInI18nTexts (route, term, locale, subpage = 'overview') {
      const path = `_${route.replace(/_$/, '').replace(/\//g, '.')}.${subpage}.texts` // TODO replace with global solution

      // * Search in page texts (i18n)
      let texts = null
      if (this.$te(path, locale)) {
        texts = this.$tm(path, locale)
      }

      let found = false
      if (texts && Object.keys(texts).length) {
        for (const text of texts) {
          if (text.toLowerCase().includes(term)) {
            found = true
            break
          }
        }
      }

      return found
    },
    searchTermInI18nCodes (route, term, locale, subpage = 'overview') {
      const path = `_${route.replace(/_$/, '').replace(/\//g, '.')}.${subpage}.codes` // TODO replace with global solution

      // * Search in page codes (i18n)
      let codes = null
      if (this.$te(path, locale)) {
        codes = this.$tm(path, locale)
      }

      let found = false
      if (codes && Object.keys(codes).length) {
        for (const code of codes) {
          if (code.toLowerCase().includes(term)) {
            found = true
            break
          }
        }
      }

      return found
    },
    clearSearchTerm () {
      this.term = ''
      this.searchTerm('')
      return true
    },

    getMenuItemHeader (item) {
      const path = `_.${item.meta.menu.header.name}._`

      if (this.$te(path)) {
        return this.$t(path)
      } else {
        return this.$t(path, 'en-US')
      }
    },
    getMenuItemSubheader (item) {
      const path = `_.${item.meta.menu.subheader}._`

      if (this.$te(path)) {
        return this.$t(path)
      } else {
        return this.$t(path, 'en-US')
      }
    },
    getMenuItemLabel (item, index) {
      // if (this.loaded) return

      if (index === this.items.length - 1) {
        this.loaded = true
      }

      const path = `_${item.path.replace(/_$/, '').replace(/\//g, '.')}._`
      // console.log(path)
      // TODO fix re-render at menu scrolling?! Lol
      if (this.$te(path)) {
        return this.$t(path)
      } else {
        return this.$t(path, 'en-US')
      }
    },

    getPageStatusIcon (status) {
      if (status === 9) {
        return ''
      } else if (status === 6) {
        return 'fas fa-spinner'
      } else {
        return 'fas fa-exclamation-circle'
      }
    },
    getPageStatusTooltip (status) {
      // TODO put text in i18n
      if (status === 9) {
        return ''
      } else if (status === 6) {
        return 'Documentation page is under constrution.'
      } else {
        return 'Documentation page is empty!'
      }
    },

    scrollToActiveMenuItem () {
      const menu = document.getElementById('menu')

      if (menu) {
        const menuItemActive = (menu.getElementsByClassName('q-router-link--active'))[0]

        if (menuItemActive && typeof menuItemActive === 'object') {
          const target = getScrollTarget(menuItemActive)
          const offset = menuItemActive.offsetTop
          const duration = 300

          // TODO fix tremor
          // TODO detect scrollarea scroll position and only setVertical in initial position (top = 0)
          setVerticalScrollPosition(target, offset - (window.innerHeight / 2) + 80, duration)
        }
      }
    }
  },

  // @ Events
  beforeCreate () {
    // console.log('DMenu - beforeCreate()!')

    const routes = this.$router.options.routes.slice(0, -1) // Delete last route
    const items = []

    for (const [index, route] of routes.entries()) {
      items[index] = Object.freeze({
        path: route.path,
        meta: {
          icon: route.meta.icon,
          menu: route.meta.menu,
          status: route.meta.status
        }
      })
    }

    this.items = Object.freeze(items)

    // console.log(this.items, routes)
  },
  created () {
    // console.log('DMenu - created()!')
  },
  beforeMount () {
    // console.log('DMenu - beforeMount()!')
  },
  mounted () {
    // console.log('DMenu - mounted()!')

    // ! Autoscrolling to active menu item after 1.5s
    setTimeout(() => {
      this.scrollToActiveMenuItem()
    }, 1500)
    // * After each route change
    this.$router.afterEach((to, from) => {
      if (!to.hash || (from.path !== to.path)) {
        setTimeout(() => {
          this.scrollToActiveMenuItem()
        }, 1500)
      }
    })

    // console.log(this.items)
  },
  beforeUpdate () {
    // console.log('DMenu - beforeUpdate()!')
  },
  updated () {
    // console.log('DMenu - updated()!')
  }
}
</script>

<style lang="sass">
#menu
  width: 100%
  height: calc(100% - 50px)

  // Support Button
  .q-btn.support
    color: #9c3700 // Contrast Checker - OK
  // List
  .q-list
    padding: 8px 0

  // List Item
  .q-item
    padding: 8px 12px
    min-height: 45px
  .q-item.q-router-link--active, .q-item--active
    color: black
    background: rgba(189,189,189,0.5)
  .q-icon
    color: #363636
  .page-status
    margin-right: 7px

  // List Item Label
  .label
    color: #363636

    &.header
      text-align: center
      min-height: 32px
      > div
        padding-bottom: 7px
        padding-top: 10px
      &.sticky
        position: sticky
        position: -webkit-sticky
        position: -moz-sticky
        position: -ms-sticky
        position: -o-sticky
        width: 100%
        top: -1px
        margin-bottom: 5px
        background-color: #f5f5f5!important
        z-index: 2
        .separator
          margin: 0 auto

      .q-icon
        padding-right: 5px
    &.subheader
      text-align: left
      padding-bottom: 5px
      padding-left: 10px
    span
      color: #363636

  // List Item Separator
  .separator
    &.list
      height: 3px
      margin: 0
    &.page
      height: 3px
    &.subpage
      height: 1px
    &.partial
      margin: 3px auto
      width: 30px
      height: 3px

// Search
label[for="search"]
  z-index: 2

  .q-field__control,
  .q-field__marginal
    height: 50px

  i.clear
    padding: 13px 8px
</style>
