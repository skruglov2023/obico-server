<template>
  <layout
  >
    <template v-slot:topBarLeft>
      <search-input @input="updateSearch" class="search-input mr-3"></search-input>
    </template>
    <template v-slot:content>
      <b-container>
        <b-row>
          <b-col>
            <vue-dropzone
              class="upload-box"
              id="dropzone"
              :options="dropzoneOptions"
              :useCustomSlot="true"
              @vdropzone-queue-complete="gcodeUploadSuccess"
              @vdropzone-error="gcodeUploadError"
              ref="gcodesDropzone"
            >
              <div class="dz-message needsclick">
                <i class="fas fa-upload fa-2x"></i> <br>
                <div>G-Code file (*.gcode, *.gco, or *.g) only.</div>
                <div>Up to {{maxFilesize}} MB each file, {{maxTotalFilesize}} GB total.</div>
              </div>
            </vue-dropzone>

            <!-- GCodes list -->
            <div class="gcodes-wrapper">
              <div class="control-panel">
                <!-- <search-input v-model="searchText" class="search-input"></search-input> -->
              </div>

              <div class="sorting-panel">
                <div
                  class="sorting-option"
                  :class="{'active': activeSorting === sorting.NAME}"
                  @click="updateSorting(sorting.NAME)"
                >
                  <span class="text">File name</span>
                  <div class="direction">
                    <i class="fas fa-arrow-down" v-if="activeSorting === sorting.NAME && sortDirection === direction.DESC"></i>
                    <i class="fas fa-arrow-up" v-else></i>
                  </div>
                </div>

                <div
                  class="sorting-option"
                  :class="{'active': activeSorting === sorting.SIZE}"
                  @click="updateSorting(sorting.SIZE)"
                >
                  <span class="text">File size</span>
                  <div class="direction">
                    <i class="fas fa-arrow-down" v-if="activeSorting === sorting.SIZE && sortDirection === direction.DESC"></i>
                    <i class="fas fa-arrow-up" v-else></i>
                  </div>
                </div>
                <div
                  class="sorting-option"
                  :class="{'active': activeSorting === sorting.UPLOADED}"
                  @click="updateSorting(sorting.UPLOADED)"
                >
                  <span class="text">Uploaded</span>
                  <div class="direction">
                    <i class="fas fa-arrow-down" v-if="activeSorting === sorting.UPLOADED && sortDirection === direction.DESC"></i>
                    <i class="fas fa-arrow-up" v-else></i>
                  </div>
                </div>
              </div>

              <div class="gcode-items-wrapper">
                <div v-for="item in gcodesToShow" :key="item.id" class="item">
                  <div class="item-info">
                    <div class="filename">{{ item.filename }}</div>
                    <div class="filesize">{{ item.filesize }}</div>
                    <div class="uploaded">{{ item.created_at.fromNow() }}</div>
                  </div>
                  <div class="remove-button-wrapper">
                    <div class="remove-button" @click="removeItem(item.id)">
                      <i class="far fa-trash-alt"></i>
                    </div>
                  </div>
                </div>
              </div>

              <mugen-scroll :handler="fetchGCodes" :should-handle="!loading" class="text-center p-4">
                <div v-if="noMoreData" class="text-center p-2">End of your G-Codes list.</div>
                <b-spinner v-if="!noMoreData" label="Loading..."></b-spinner>
              </mugen-scroll>
            </div>
          </b-col>
        </b-row>
      </b-container>
    </template>
  </layout>
</template>

<script>
import Layout from '@src/components/Layout.vue'
import vue2Dropzone from 'vue2-dropzone'
import 'vue2-dropzone/dist/vue2Dropzone.min.css'
import urls from '@config/server-urls'
import axios from 'axios'
import MugenScroll from 'vue-mugen-scroll'
import { normalizedGcode } from '@src/lib/normalizers'
import { user } from '@src/lib/page_context'
import SearchInput from '@src/components/SearchInput.vue'

const SORTING = {
  NAME: 1,
  SIZE: 2,
  UPLOADED: 3
}

const SORT_DIRECTION = {
  ASC: 1,
  DESC: -1
}

export default {
  name: 'GCodesPage',

  components: {
    Layout,
    SearchInput,
    vueDropzone: vue2Dropzone,
    MugenScroll,
  },

  props: {
    csrf: {
      type: String,
      requeired: true,
    },
  },

  data() {
    return {
      user: null,
      searchText: '',
      activeSorting: SORTING.UPLOADED,
      sortDirection: SORT_DIRECTION.DESC,
      sorting: SORTING,
      direction: SORT_DIRECTION,
      gcodes: [],
      currentPage: 1,
      loading: false,
      noMoreData: false,
    }
  },

  computed: {
    maxFilesize() {
      return this.user.is_pro ? 500 : 50 // MB
    },

    maxTotalFilesize() {
      return this.user.is_pro ? 50 : 1 // GB
    },

    dropzoneOptions() {
      return {
        withCredentials: true,
        maxFilesize: this.maxFilesize,
        timeout: 60 * 60 * 1000, // For large files
        acceptedFiles: '.g,.gcode,.gco',
        url: 'upload/',
        headers: { 'X-CSRFToken': this.csrf },
      }
    },
    gcodesToShow() {
      let gcodes = this.gcodes

      if (!gcodes) {
        return []
      }

      const query = this.searchText.toLowerCase()
      gcodes = gcodes.filter((gcf) => gcf.filename.toLowerCase().indexOf(query) > -1)

      const sortDirection = this.sortDirection

      switch (this.activeSorting) {
      case SORTING.NAME:
        gcodes.sort(function(a, b) {
          var filenameA = a.filename.toUpperCase() // ignore upper and lowercase
          var filenameB = b.filename.toUpperCase() // ignore upper and lowercase

          if (filenameA < filenameB) {
            return sortDirection === SORT_DIRECTION.ASC ? -1 : 1
          }
          if (filenameA > filenameB) {
            return sortDirection === SORT_DIRECTION.ASC ? 1 : -1
          }

          return 0
        })
        break

      case SORTING.SIZE:
        gcodes.sort(function(a, b) {
          if (sortDirection === SORT_DIRECTION.ASC) {
            return a.num_bytes - b.num_bytes
          } else {
            return b.num_bytes - a.num_bytes
          }
        })
        break

      case SORTING.UPLOADED:
        gcodes.sort(function(a, b) {
          const uploadedA = a.created_at.unix()
          const uploadedB = b.created_at.unix()

          if (sortDirection === SORT_DIRECTION.ASC) {
            return uploadedA - uploadedB
          } else {
            return uploadedB - uploadedA
          }
        })
        break
      }

      return gcodes
    },
  },

  created() {
    this.user = user()
    this.fetchGCodes()
  },

  methods: {
    updateSearch(search) {
      this.searchText = search
    },
    gcodeUploadSuccess() {
      this.$refs.gcodesDropzone.removeAllFiles()

      this.gcodes = []
      this.currentPage = 1
      this.noMoreData = false
      this.fetchGCodes()
    },

    gcodeUploadError(file, message) {
      this.$swal.Reject.fire({
        html: `<p class="text-center">${message}</p>`})
    },

    fetchGCodes() {
      if (this.noMoreData) {
        return
      }

      this.loading = true

      return axios
        .get(urls.gcodes(this.currentPage))
        .then(response => {
          this.loading = false

          this.noMoreData = response.data.next === null
          let gcodeFiles = response.data.results

          if (gcodeFiles) {
            this.gcodes.push(...gcodeFiles.map(data => normalizedGcode(data)))
            this.currentPage += 1
          }
        }).catch(err => {
          console.log(err)
          this.noMoreData = true
        })
    },

    updateSorting(sortOption) {
      if (this.activeSorting === sortOption) {
        this.sortDirection = -this.sortDirection
      } else {
        this.activeSorting = sortOption
        this.sortDirection = SORT_DIRECTION.ASC
      }
    },

    removeItem(id) {
      axios
        .delete(urls.gcode(id), )
        .then(() => {
          for (let i = 0; i < this.gcodes.length; i++) {
            const deleted = this.gcodes[i]

            if (deleted.id === id) {
              this.gcodes.splice(i, 1)

              // Toast for user
              let toastHtml = ''
              toastHtml += `<h6 class="text-danger">${deleted.filename} successfully deleted!</h6>`

              this.$swal.Toast.fire({
                // icon: 'success',
                html: toastHtml,
              })
            }
          }
        })
    }
  },
}
</script>

<style lang="sass" scoped>
.upload-box
  margin-bottom: var(--gap-between-blocks)

.search-input
  height: 30px
  input
    background-color: var(--color-surface-secondary)
    border: var(--color-surface-secondary)

.gcodes-wrapper
  background-color: var(--color-surface-secondary)
  padding: 2em
  border-radius: var(--border-radius-lg)

.control-panel
  border-bottom: 1px solid var(--color-divider)
  padding-bottom: 16px

.sorting-panel
  display: flex
  padding: 1em calc(1em + 30px) 1em 1em
  border-bottom: 1px solid var(--color-divider)

  .sorting-option
    flex: 1
    display: flex
    justify-content: space-between
    margin-left: 30px
    align-items: center
    font-size: .9rem

    &:hover
      cursor: pointer

    &:first-child
      margin-left: 0

    .direction
      font-size: .8rem
      opacity: .3

  .sorting-option.active .direction
    opacity: 1

  @media (max-width: 768px)
    &
      flex-direction: column
      padding: 1em 0

    .sorting-option
      margin-left: 0
      margin-bottom: 4px

    .remove-button-placeholder
      display: none

.gcode-items-wrapper
  .item
    display: flex
    align-items: center
    padding: .6em 1em

    &:nth-child(2n)
      background-color: var(--color-table-accent)

    .item-info
      display: flex
      width: 100%
      overflow: hidden
      flex: 1

      div
        flex: 1
        margin-left: 30px

        &:first-child
          margin-left: 0

      .filename
        text-overflow: ellipsis
        overflow: hidden
        white-space: nowrap
        width: 100%

    .remove-button
      width: 30px
      height: 30px
      text-align: center
      line-height: 30px
      border-radius: 50%
      transition: background-color .2s ease-out

      &:hover
        background-color: var(--color-danger)
        color: var(--color-on-primary)
        cursor: pointer

    @media (max-width: 768px)
      &
        padding: .6em 0

      &:nth-child(2n)
        margin: 0 -.6em
        padding: .6em

      .item-info
        flex-direction: column
        align-items: flex-start

        div
          margin-left: 0
</style>
