<template>
  <div>
    <vue-headful :title="$t('Internationalization title', 'Proxeus - Internationalization')"/>
    <top-nav :title="$t('Internationalization')">
      <button slot="buttons" v-if="app.userIsUserOrHigher()"
              @click="app.exportData('', null, '/api/i18n/export', 'i18n')" type="button"
              class="btn btn-primary mr-3">
        <span>{{$t('Export')}}</span>
      </button>
      <div slot="buttons" id="collapsingNavbar3">
        <ul class="nav navbar-nav ml-auto">
          <li class="nav-item">
            <language-drop-down style="margin-top: 3px;"/>
          </li>
        </ul>
      </div>
    </top-nav>
    <div ref="scrollable" class="container-fluid main-container" style="overflow: auto;">
      <h2 class="mt-3">{{ $t('Languages') }}</h2>
      <div class="row">
        <div class="col">
          <table class="table table-bordered langstbl" v-if="meta">
            <thead>
            <tr>
              <th class="cntr">{{$t('Code')}}</th>
              <th class="cntr">{{$t('Label')}}</th>
              <th class="cntr">{{$t('Active')}}</th>
            </tr>
            </thead>
            <tbody>
            <tr v-for="(lang, key) in meta.langList" :key="key">
              <td class="cntr">{{ lang.Code }}</td>
              <td class="cntr impcnt" style="max-width: 100px;">{{ $t(lang.Code) }}</td>
              <td class="cntr">
                <i18n-lang-cell :lang="lang"></i18n-lang-cell>
              </td>
            </tr>
            </tbody>
            <tfoot>
            <tr class="bg-light">
              <td colspan="2" class="pt-3 px-3">
                <animated-input :label="$t('Language code')" name="i18n_text"
                                v-model="newLangKey" :helptext="$t('For example: en, de, fr, it ...')"></animated-input>
              </td>
              <td class="cntr px-3">
                <button :disabled="newLangKey.length !== 2" @click="addLanguage"
                        type="button" class="btn btn-primary btn-round">
                  <span class="material-icons">add</span>
                </button>
              </td>
            </tr>
            </tfoot>
          </table>
        </div>
      </div>

      <div v-if="hasLangs()">
        <h2 style="margin-top:20px;">{{ $t('Translations') }}</h2>
        <div style="position:relative;">
          <div ref="listGroup" class="mlist-group mbottominset" style="width:100%;">
            <table class="table table-bordered">
              <thead>
              <tr>
                <th v-bind:class="{'w-25':cols.length>0}">
                  <div><strong>{{$t('Code')}}</strong></div>
                </th>
                <th v-for="(lang, key) in meta.langList"
                    v-bind:key="lang.Code"
                    v-bind:class="{active:cols.indexOf(key) > -1,'tdmin tdmoremin':cols.indexOf(key) === -1}"
                    v-on:click="toggleLangCol(key,$event)">
                  <div>
                    <span class="btn btn-info"
                          :class="{'btn-info-active': cols.indexOf(key) > -1}">{{ lang.Code }}</span>
                  </div>
                </th>
                <th class="tdmin action_col">
                  <div></div>
                </th>
              </tr>
              <tr class="bg-light">
                <th>
                  <input autofocus ref="search" v-model="langKey" class="form-control" type="text" placeholder="Search in codes">
                </th>
                <th :colspan="meta.langListSize" style="min-width: 200px;">
                  <input autofocus v-model="langValue" class="form-control" type="text" placeholder="Search in translations">
                </th>
                <th class="tdmin">
                  <button type="button" class="btn btn-primary btn-round"
                          @click="addTranslation"
                          :disabled="langKey.length === 0 || langValue.length === 0 || translations[langKey]">
                    <span class="material-icons">add</span>
                  </button>
                </th>
              </tr>
              </thead>
              <tbody>
              <translation :activateLangCol="activateLangCol" :cols="cols" :update="transUpdate"
                           :translations="translations"
                           :langList="meta.langList" :lk="lk"
                           v-for="(translation, lk) in translations" v-bind:key="lk"/>
              </tbody>
            </table>
            <trigger :init="triggerInit" @trigger="bottomTrigger"/>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import TopNav from '@/components/layout/TopNav'
import Translation from '@/views/appDependentComponents/i18n/Translation'
import I18nLangCell from '@/views/appDependentComponents/i18n/I18nLangCell'
// import I18nTransCell from '@/views/appDependentComponents/i18n/I18nTransCell'
import LanguageDropDown from '@/views/appDependentComponents/LanguageDropDown'
import Trigger from '../components/Trigger'
import mafdc from '@/mixinApp'
import AnimatedInput from '../components/AnimatedInput'

export default {
  mixins: [mafdc],
  name: 'i18n',
  components: {
    AnimatedInput,
    Trigger,
    I18nLangCell,
    // I18nTransCell,
    Translation,
    TopNav,
    LanguageDropDown
  },
  created () {
  },
  mounted () {
    this.$nextTick(() => this.$refs.search && this.$refs.search.field && this.$refs.search.field.focus())
    this.find()
  },
  watch: {
    langKey: function () {
      this.find()
    },
    langValue: function () {
      this.find()
    }
  },
  beforeDestroy () {
  },
  updated () {
  },
  methods: {
    triggerInit (startFunc, stopFunc, hideFunc) {
      this.triggerApi.start = startFunc
      this.triggerApi.stop = stopFunc
      this.triggerApi.hide = hideFunc
      this.resetEntries()
    },
    resetEntries () {
      this.translations = { '-': { en: '' } }
      if (this.triggerApi.start) {
        this.triggerApi.start()
      }
    },
    getReqLink () {
      return '/api/admin/i18n/find'
    },
    bottomTrigger (startFunc, stopFunc, hideFunc) {
      if (!this.reachedTheEnd) {
        this.listIndex += 1
        this.loading = true
        axios.get(this.getReqLink(), { params: this.getFindParams() }).then(response => {
          if (this.mapSizeGreaterThan0(response.data)) {
            hideFunc()
            const obj = response.data
            for (const key in obj) {
              if (Object.prototype.hasOwnProperty.call(obj, key)) {
                for (const innerKey in obj[key]) {
                  if (Object.prototype.hasOwnProperty.call(obj[key], innerKey)) {
                    if (!this.translations[key]) {
                      this.translations[key] = {}
                    }
                    if (!this.translations[key][innerKey]) {
                      this.translations[key][innerKey] = {}
                    }
                    this.translations[key][innerKey] = obj[key][innerKey]
                  }
                }
              }
            }
            this.$forceUpdate()
          } else {
            this.reachedTheEnd = true
            stopFunc()
          }
          this.loading = false
        }, (err) => {
          // on err
          stopFunc()
          this.loading = false
          this.app.handleError(err)
        })
      }
    },
    fallbackLang () {
      return this.app.meta.langFallback
    },
    transUpdate (trans) {
      let t = this.translations[trans.k]
      if (!t) {
        t = { [trans.lang]: trans.value }
        this.translations[trans.k] = t
        return
      }
      t[trans.lang] = trans.value
    },
    getFindParams () {
      return { k: this.langKey, v: this.langValue, l: this.listLimit, i: this.listIndex }
    },
    find () {
      this.listIndex = 0
      this.reachedTheEnd = false
      if (this.triggerApi.start) {
        this.triggerApi.start()
      }
      // if (this.lastRequest !== null) {
      //   this.lastRequest.cancel()
      // }
      this.loading = true
      // let CancelToken = axios.CancelToken;
      // this.lastRequest = CancelToken.source();
      // let source = this.lastRequest;
      axios.get(this.getReqLink(), {
        // cancelToken: source.token,
        params: this.getFindParams()
      }).then(response => {
        if (this.mapSizeGreaterThan0(response.data)) {
          this.resetEntries()
          if (this.triggerApi.hide) {
            this.triggerApi.hide()
          }
          this.translations = response.data
        } else {
          this.resetEntries()
          this.reachedTheEnd = true
          if (this.triggerApi.stop) {
            this.triggerApi.stop()
          }
        }
        // this.lastRequest = null;
        this.loading = false
      }, error => {
        this.resetEntries()
        if (axios.isCancel(error)) {
        } else {
          // this.lastRequest = null;
          this.loading = false
          if (this.triggerApi.stop) {
            this.triggerApi.stop()
          }
        }
        this.app.handleError(error)
      })
      // this.isLastRequest = true
    },
    mapSizeGreaterThan0 (obj) {
      let size = 0
      if (obj) {
        for (const key in obj) {
          if (Object.prototype.hasOwnProperty.call(obj, key)) {
            size++
            break
          }
        }
      }
      return size
    },
    toggleLangCol (key, e) {
      if (this.cols.indexOf(key) > -1) {
        this.cols.splice(this.cols.indexOf(key), 1)
      } else {
        this.cols.push(key)
      }
    },
    activateLangCol (key, e) {
      if (this.cols.indexOf(key) === -1) {
        this.cols.push(key)
      }
    },
    addTranslation () {
      axios.post('/api/admin/i18n/update', { [this.langKey]: { [this.fallbackLang()]: this.langValue } }).then(response => {
        this.find()
        this.$notify({
          group: 'app',
          title: this.$t('Success'),
          text: this.$t('Added translation'),
          type: 'success'
        })
      }, (err) => {
        this.app.handleError(err)
        this.$notify({
          group: 'app',
          title: this.$t('Error'),
          text: this.$t(
            'Could not save translation. Please try again or if the error persists contact the platform operator.'),
          type: 'error'
        })
      })
    },
    hasLangs () {
      return this.meta && this.meta.langList && this.meta.langList.length > 0
    },
    async addLanguage () {
      axios.post('/api/admin/i18n/lang', { [this.newLangKey]: true }).then(response => {
        this.app.loadMeta(this.find)
        this.$notify({
          group: 'app',
          title: this.$t('Success'),
          text: this.$t('Added language'),
          type: 'success'
        })
      }, (err) => {
        this.app.handleError(err)
        this.$notify({
          group: 'app',
          title: this.$t('Error'),
          text: this.$t(
            'Could not add language. Please try again or if the error persists contact the platform operator.'),
          type: 'error'
        })
      })
    }
  },
  computed: {
    meta () {
      return this.app.meta
    },
    activeLang () {
      return this.app.selectedLang
    }
  },
  data () {
    return {
      translations: {},
      langKey: '',
      langValue: '',
      newLangKey: '',
      listIndex: 0,
      listLimit: 50,
      loading: false,
      triggerApi: {
        start: null,
        stop: null,
        hide: null
      },
      reachedTheEnd: false,
      cols: []
    }
  }
}
</script>

<style lang="scss">
  .action_col {
  }

  .i18n-tbl thead th > div {
    position: absolute;
    top: -6px;
    /*width: 100%;*/
  }

  .i18n-tbl thead th:last-child > div {
    top: 8px;
  }

  .i18n-tbl .mlist-group {
    border-top: 1px solid gainsboro;
  }

  .i18n-tbl thead tr:nth-child(2) th > div {
    top: 71px;
    margin-left: -3px;
  }

  table.i18n-tbl > tbody > tr td:first-child {
    padding: 0 15px;
    min-width: 212px;
  }

  .i18n-tbl .tdmoremin {
    max-width: 50px;
    padding-right: 4px;
  }

  .i18n-tbl .tdmin {
    overflow: hidden;
  }

  .i18n-tbl .active {
    border-left: 1px solid #062a85;
  }

  .i18n-tbl thead th {
    border-left: 1px solid #40e1d1;
  }

  .i18n-tbl .tdmoremin {
    border-left: 1px solid #40e1d1;
  }

  .i18n-tbl .btn-info-active {
    background: #062a85;
    border-color: #062a85;
  }

  .i18n-tbl span.btn.btn-info {
    padding: 2px 9px;
    cursor: pointer;
  }

  .i18n-tbl .btn.btn-round {
    padding: 12px;
  }

  .langstbl .cntr {
    vertical-align: middle;
    text-align: center;
  }

  .langstbl td {
    padding: 4px;
  }
</style>
