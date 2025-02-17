<template>
  <div class="sequences page fixed-page">
    <div class="sequence-list-header page-header flexrow">
      <search-field
        class="flexrow-item mt1"
        ref="sequence-search-field"
        :can-save="true"
        @change="onSearchChange"
        @enter="saveSearchQuery"
        @save="saveSearchQuery"
        placeholder="ex: e01 s01 anim=wip"
      />
      <combobox
        class="mb0 flexrow-item"
        :label="$t('statistics.display_mode')"
        locale-key-prefix="statistics."
        :options="displayModeOptions"
        v-model="displayMode"
      />
      <combobox
        class="mb0 flexrow-item"
        :label="$t('statistics.count_mode')"
        locale-key-prefix="statistics."
        :options="countModeOptions"
        v-model="countMode"
      />
      <span class="filler">
      </span>
      <button-simple
        class="flexrow-item"
        icon="refresh"
        :title="$t('main.reload')"
        @click="reloadData"
      />
      <button-simple
        class="flexrow-item"
        icon="download"
        :title="$t('main.csv.export_file')"
        @click="exportStatisticsToCsv"
      />
    </div>

    <div class="query-list mt1">
      <search-query-list
        :queries="sequenceSearchQueries"
        @change-search="changeSearch"
        @remove-search="removeSearchQuery"
      />
    </div>

    <sequence-list
      ref="sequence-list"
      :count-mode="countMode"
      :display-mode="displayMode"
      :entries="displayedSequences"
      :is-loading="isShotsLoading || initialLoading"
      :is-error="isShotsLoadingError"
      :validation-columns="shotValidationColumns"
      :sequence-stats="sequenceStats"
      :show-all="sequenceSearchText.length === 0"
      @delete-clicked="onDeleteClicked"
      @edit-clicked="onEditClicked"
      @field-changed="onFieldChanged"
      @scroll="saveScrollPosition"
    />

    <edit-sequence-modal
      :active="modals.isNewDisplayed"
      :is-loading="loading.edit"
      :is-error="errors.edit"
      :sequence-to-edit="sequenceToEdit"
      @cancel="modals.isNewDisplayed = false"
      @confirm="confirmEditSequence"
    />

    <hard-delete-modal
      :active="modals.isDeleteDisplayed"
      :is-loading="loading.del"
      :is-error="errors.del"
      :text="deleteText()"
      :error-text="$t('sequences.delete_error')"
      :lock-text="sequenceToDelete ? sequenceToDelete.name : ''"
      @cancel="modals.isDeleteDisplayed = false"
      @confirm="confirmDeleteSequence"
    />

  </div>
</template>

<script>
import moment from 'moment'
import { mapGetters, mapActions } from 'vuex'
import csv from '@/lib/csv'
import stringHelpers from '@/lib/string'

import ButtonSimple from '@/components/widgets/ButtonSimple'
import Combobox from '@/components/widgets/Combobox'
import HardDeleteModal from '@/components/modals/HardDeleteModal'
import EditSequenceModal from '@/components/modals/EditSequenceModal'
import SearchField from '@/components/widgets/SearchField'
import SearchQueryList from '@/components/widgets/SearchQueryList'
import SequenceList from '@/components/lists/SequenceList.vue'

export default {
  name: 'sequences',

  components: {
    ButtonSimple,
    Combobox,
    HardDeleteModal,
    EditSequenceModal,
    SearchField,
    SearchQueryList,
    SequenceList
  },

  data () {
    return {
      countMode: 'count',
      displayMode: 'pie',
      initialLoading: true,
      sequenceToDelete: null,
      sequenceToEdit: null,
      countModeOptions: [
        { label: 'shots', value: 'count' },
        { label: 'frames', value: 'frames' }
      ],
      displayModeOptions: [
        { label: 'pie', value: 'pie' },
        { label: 'count', value: 'count' }
      ],
      errors: {
        edit: false,
        del: false
      },
      loading: {
        edit: false,
        del: false
      },
      modals: {
        isNewDisplayed: false,
        isDeleteDisplayed: false
      }
    }
  },

  computed: {
    ...mapGetters([
      'currentEpisode',
      'currentProduction',
      'displayedSequences',
      'isCurrentUserManager',
      'isShotsLoading',
      'isShotsLoadingError',
      'isTVShow',
      'searchSequenceFilters',
      'sequences',
      'sequenceMap',
      'sequencesPath',
      'sequenceStats',
      'sequenceSearchText',
      'sequenceSearchQueries',
      'sequenceListScrollPosition',
      'shotValidationColumns',
      'taskTypeMap',
      'taskStatusMap'
    ])
  },

  mounted () {
    this.loadShots(() => {
      this.initSequences()
        .then(() => {
          this.initialLoading = false
        })
        .catch(err => console.error(err))
      this.setDefaultSearchText()
      this.setDefaultListScrollPosition()
    })
  },

  methods: {
    ...mapActions([
      'computeSequenceStats',
      'deleteSequence',
      'editSequence',
      'hideAssignations',
      'initSequences',
      'loadShots',
      'removeSequenceSearch',
      'saveSequenceSearch',
      'setLastProductionScreen',
      'setSequenceSearch',
      'setSequenceListScrollPosition',
      'showAssignations'
    ]),

    reloadData () {
      this.initialLoading = true
      this.loadShots(() => {
        this.initialLoading = false
        this.computeSequenceStats()
      })
    },

    setDefaultSearchText () {
      if (this.sequenceSearchText.length > 0) {
        this.$refs['sequence-search-field'].setValue(this.sequenceSearchText)
      }
    },

    setDefaultListScrollPosition () {
      if (this.$refs['sequence-list']) {
        this.$refs['sequence-list'].setScrollPosition(
          this.sequenceListScrollPosition
        )
      }
    },

    navigateToList () {
      this.$router.push(this.sequencesPath)
    },

    confirmEditSequence (form) {
      this.loading.edit = true
      this.errors.edit = false

      this.editSequence(form)
        .then(() => {
          this.loading.edit = false
          this.modals.isNewDisplayed = false
        }).catch((err) => {
          console.error(err)
          this.loading.edit = false
          this.errors.edit = true
        })
    },

    confirmDeleteSequence () {
      this.loading.del = true
      this.deleteSequence(this.sequenceToDelete)
        .then(() => {
          this.loading.del = false
          this.modals.isDeleteDisplayed = false
        }).catch((err) => {
          console.error(err)
          this.loading.del = false
          this.errors.del = true
        })
    },

    resetEditModal () {
      const form = { name: '' }
      if (this.sequences.length > 0) {
        form.sequence_id = this.sequences[0].id
      }
      if (this.openProductions.length > 0) {
        form.production_id = this.openProductions[0].id
      }
      this.sequenceToEdit = form
    },

    deleteText () {
      const sequence = this.sequenceToDelete
      if (sequence) {
        return this.$t('sequences.delete_text', { name: sequence.name })
      } else {
        return ''
      }
    },

    onEditClicked (sequence) {
      this.sequenceToEdit = sequence
      this.modals.isNewDisplayed = true
    },

    onDeleteClicked (sequence) {
      this.sequenceToDelete = sequence
      this.modals.isDeleteDisplayed = true
    },

    onSearchChange (event) {
      const searchQuery = this.$refs['sequence-search-field'].getValue()
      this.setSequenceSearch(searchQuery)
    },

    changeSearch (searchQuery) {
      this.$refs['sequence-search-field'].setValue(searchQuery.search_query)
      this.$refs['sequence-search-field']
        .$emit('change', searchQuery.search_query)
    },

    saveSearchQuery (searchQuery) {
      this.saveSequenceSearch(searchQuery)
        .catch(console.error)
    },

    removeSearchQuery (searchQuery) {
      this.removeSequenceSearch(searchQuery)
        .catch(console.error)
    },

    saveScrollPosition (scrollPosition) {
      this.setSequenceListScrollPosition(scrollPosition)
    },

    exportStatisticsToCsv () {
      const nameData = [
        moment().format('YYYYMMDD'),
        this.currentProduction.name,
        'sequences',
        'statistics'
      ]
      if (this.currentEpisode) {
        nameData.splice(2, 0, this.currentEpisode.name)
      }
      const name = stringHelpers.slugify(nameData.join('_'))
      csv.generateStatReports(
        name,
        this.sequenceStats,
        this.taskTypeMap,
        this.taskStatusMap,
        this.sequenceMap,
        this.countMode
      )
    },

    onFieldChanged ({ entry, fieldName, value }) {
      const data = { id: entry.id }
      data[fieldName] = value
      this.editSequence(data)
    }
  },

  watch: {
    currentProduction () {
      this.$refs['sequence-search-field'].setValue('')
      this.$store.commit('SET_SEQUENCE_LIST_SCROLL_POSITION', 0)

      if (!this.isTVShow) {
        this.loadShots(() => {
          this.initSequences()
            .then(this.handleModalsDisplay)
            .catch(err => console.error(err))
        })
      }
    },

    currentEpisode () {
      if (this.isTVShow && this.currentEpisode) {
        this.loadShots(() => {
          this.initSequences()
            .then(this.handleModalsDisplay)
            .then(() => {
              this.initialLoading = false
            })
            .catch(err => console.error(err))
        })
      }
    },

    searchSequenceFilters () {
      this.computeSequenceStats()
    }
  },

  metaInfo () {
    if (this.isTVShow) {
      return {
        title: `${this.currentProduction ? this.currentProduction.name : ''}` +
               ` - ${this.currentEpisode ? this.currentEpisode.name : ''}` +
               ` | ${this.$t('sequences.title')} - Kitsu`
      }
    } else {
      return {
        title: `${this.currentProduction ? this.currentProduction.name : ''}` +
               ` ${this.$t('sequences.title')} - Kitsu`
      }
    }
  }

}
</script>

<style lang="scss" scoped>
.mb0 {
  margin-bottom: 0;
}
</style>
