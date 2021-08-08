<template>
  <v-card class="crud-main">
    <v-card-title class="text-capitalize justify-space-between">
      {{ config.title }}
      <slot name="searchBar" />
    </v-card-title>
    <slot name="toolbar" />
    <v-data-table
      ref="crud"
      :loading="loading"
      :headers="config.headers"
      :items="items"
      :options.sync="options"
      :multi-sort="true"
      loading-text="Loading... Please wait"
      disable-pagination
      hide-default-footer
      @update:options="updateList"
    >
      <template #item="{ item, headers, expand, isExpanded }">
        <tr v-if="isDesktop">
          <td
            v-for="(header, idx) in headers"
            :key="idx"
            :class="header.class"
            :style="header.style"
          >
            <div
              v-if="header.value !== 'data-table-expand'"
              class="crud-content my-2"
            >
              <div
                v-if="!header.comp"
                v-html="
                  typeof header.value === 'function'
                    ? header.value(item)
                    : getValueNested(item, header.value)
                "
              ></div>
              <component
                :is="setComponent(header.pathComp, header.comp)"
                v-else
                :opt="header"
                :item="item"
                :more-data="moreData"
                @outdata="$emit('outdata', $event)"
              />
            </div>
            <v-icon
              v-else
              :class="{
                'v-data-table__expand-icon--active': !isExpanded,
                'v-data-table__expand-icon': true
              }"
              @click="expand(!isExpanded)"
            >
              mdi-chevron-down
            </v-icon>
          </td>
        </tr>
        <tr v-else>
          <td
            v-for="(header, idx) in headers"
            :key="idx"
            class="v-data-table__mobile-row"
          >
            <div class="v-data-table__mobile-row__wrapper my-2">
              <div class="v-data-table__mobile-row__header">
                {{ header.text }}
              </div>
              <div
                v-if="header.value !== 'data-table-expand'"
                class="v-data-table__mobile-row__cell crud-content"
              >
                <div
                  v-if="!header.comp"
                  v-html="
                    typeof header.value === 'function'
                      ? header.value(item)
                      : getValueNested(item, header.value)
                  "
                ></div>
                <div v-else>
                  <component
                    :is="setComponent(header.pathComp, header.comp)"
                    :opt="header"
                    :item="item"
                    :more-data="moreData"
                    @outdata="$emit('outdata', $event)"
                  />
                </div>
              </div>
              <div v-else class="v-data-table__mobile-row__cell">
                <v-icon
                  :class="{
                    'v-data-table__expand-icon--active': !isExpanded,
                    'v-data-table__expand-icon': true
                  }"
                  @click="expand(!isExpanded)"
                >
                  mdi-chevron-down
                </v-icon>
              </div>
            </div>
          </td>
        </tr>
      </template>
      <template #expanded-item="{ item, headers }">
        <tr class="v-data-table__expanded v-data-table__expanded__content">
          <td :colspan="headers.length + 1">
            <v-container grid-list-md>
              <v-layout row wrap>
                <v-flex
                  v-for="(ed, key) in config.expand"
                  :key="key"
                  :class="ed.class"
                >
                  <component
                    :is="setComponent(ed.pathComp, ed.comp)"
                    v-if="ed.comp"
                    :opt="ed"
                    :item="item"
                    :more-data="moreData"
                    @outdata="$emit('outdata', $event)"
                  />
                </v-flex>
              </v-layout>
            </v-container>
          </td>
        </tr>
      </template>
    </v-data-table>
    <div class="text-center pt-2">
      <v-pagination
        v-model="pagination.page"
        :length="pageCount"
        :total-visible="7"
        @input="updateList"
      />
    </div>
  </v-card>
</template>

<script>
import _debounce from 'lodash/debounce'

export default {
  props: {
    moreData: {
      type: Object,
      default() {
        return {}
      }
    },
    config: {
      type: Object,
      default() {
        return {}
      }
    },
    localItems: {
      type: Array,
      default() {
        return []
      }
    }
  },
  data() {
    return {
      isDesktop: true,
      items: [],
      loading: false,
      options: {
        sortBy: ['createdAt'],
        sortDesc: [true],
        itemsPerPage: 5,
        count: true
      },
      pagination: {
        page: 1,
        itemsLength: 0
      },
      processData: _debounce(function (v) {
        this.list()
      }, 1000)
    }
  },
  computed: {
    pageCount() {
      return this.pagination.itemsLength
        ? Math.ceil(this.pagination.itemsLength / this.options.itemsPerPage)
        : 0
    }
  },
  watch: {
    'config.search': {
      deep: true,
      handler(v) {
        this.processData(v)
      }
    }
  },
  mounted() {
    if (this.config.expand) {
      this.$refs.crud.computedHeaders.push({
        text: '',
        sortable: false,
        width: '1px',
        value: 'data-table-expand'
      })
    }
    this.isDesktop = window.innerWidth > 600
    window.addEventListener('resize', e => {
      this.isDesktop = window.innerWidth > 600
    })
  },
  methods: {
    async list(
      pagination = { ...this.options, ...this.pagination },
      path = this.config.path,
      queryMore = this.config.queryMore || {}
    ) {
      this.loading = true
      let result = {
        items: [],
        totalItems: 0
      }
      const search = Object.keys(this.config.search || {}).reduce((o, key) => {
        o[key] = this.config.search[key]
        return o
      }, {})
      const query = Object.assign(pagination, queryMore, search)

      if (!path) {
        result = this.localQuery(query)
      } else {
        try {
          const { data } = await this.$axios.get(`${path}?${this.qs(query)}`)
          result = data
        } catch (err) {
          console.log(err)
        }
      }
      this.items = result.items
      this.pagination.itemsLength = result.totalItems
      this.loading = false
    },
    updateList() {
      this.list()
    },
    getValueNested(item, path) {
      const indexes = path ? path.split('.') : []

      const nestedItem = indexes.reduce((o, i) => {
        if (o) {
          o = o[i]
        }
        return o
      }, item)
      return nestedItem
    },
    qs(params) {
      return Object.keys(params)
        .map(k => encodeURIComponent(k) + '=' + encodeURIComponent(params[k]))
        .join('&')
    },
    localQuery(q) {
      console.log('?????')
      const { searchField, keyword } = q
      let items = this.localItems
      if (searchField && keyword) {
        items = items.filter(o =>
          o[searchField].toLowerCase().includes(keyword.toLowerCase())
        )
      }
      return this.paginator(items, q.page, q.itemsPerPage)
    },
    paginator(items, page = 1, itemsPerPage = 10) {
      const offset = (page - 1) * itemsPerPage
      const pages = Math.ceil(items.length / itemsPerPage)
      return {
        pagination: {
          page,
          pages,
          itemsPerPage
        },
        totalItems: items.length,
        items: items.slice(offset).slice(0, itemsPerPage)
      }
    },
    setComponent(path, comp) {
      return require(`~/components/${path ? `comps/${path}` : 'comps'}/${comp}`)
        .default
    }
  }
}
</script>

<style lang="scss" scoped>
.crud-main ::v-deep {
  .v-data-table td {
    height: 100% !important;
  }
  .v-data-table__mobile-row__wrapper {
    width: 100%;
    display: flex;
  }
  .v-data-table__mobile-row__header {
    width: 40%;
  }
  .v-data-table__mobile-row__cell {
    width: 100%;
  }
}
</style>
