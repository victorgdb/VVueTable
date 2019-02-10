<template>
  <table :class="{
      table: true,
      scrollable: height !== 'auto' && filteredItems.length > 0
    }">
    <thead class="sorting">
    <tr>
      <th
              v-for="(header, index) in headers"
              :key="index"
              :class="{
                      sorting: sort,
                      asc: pagination.sortBy === header.id && !pagination.descending,
                      desc: pagination.sortBy === header.id && pagination.descending
                    }"
              :style="{
                  'text-align': header.align ? header.align : 'left',
                }"
              @click="sortColumn(header.id)"
      >
        <span v-html="header.text"></span>
      </th>
    </tr>
    <tr
      class="textFilter"
      v-if="textFilter"
    >
      <td
        v-for="(header, headerKey) in headers"
        :key="`1${headerKey}`"
      >
        <input
          :placeholder="header.textFilterString ? header.textFilterString : `Search ${header.text}`"
          type="text"
          v-model="pagination.textFilters[header.id]"
        />
      </td>
    </tr>
    </thead>
    <tbody
      :style="{
        'max-height': filteredItems.length > 0 ? height : 'auto',
      }"
    >
    <tr
      v-for="(item, key) in filteredItems"
      :key="`2${key}`"
    >
      <slot :item="item">
        <td
          v-for="(header, headerKey) in headers"
          :key="`2${headerKey}`"
          v-html="item[header.id].text"
          :style="{
            'text-align': item[header.id].align ? item[header.id].align : 'left',
          }"
        >
        </td>
      </slot>
    </tr>
    </tbody>
    <tfoot>
    <tr>
      <td
        :colspan="headers.length"
        class="align-right"
      >
          <span
            v-if="filteredItems.length === items.length"
          >
            {{ totalItems }} items
          </span>
        <span v-else>
          {{ totalFilteredItems }} / {{ totalItems }} items
        </span>
      </td>
    </tr>
    </tfoot>
  </table>
</template>

<script>
    export default {
        components: {
        },
        props: {
            items: {
                type: Array,
                default: () => [],
            },
            headers: {
                type: Array,
                default: () => [],
            },
            sort: {
                type: Boolean,
                default: true,
            },
            textFilter: {
                type: Boolean,
                default: false,
            },
            height: {
                type: String,
                default: 'auto',
            }
        },
        data() {
            return {
                pagination: {
                    sortBy: '',
                    descending: false,
                    textFilters: {

                    },
                },
                totalItems: 0,
                totalFilteredItems: 0,
            };
        },
        computed: {
            filteredItems() {
                let items = this.items;

                if (this.pagination.sortBy.length > 0) {
                    items = items.sort(this.sortFn)
                }

                Object.keys(this.pagination.textFilters).forEach(headerId => {
                    const searchTerm = this.pagination.textFilters[headerId].toLowerCase();
                    if (searchTerm.length > 0) {
                        // Make sure we work with strings
                        items = items.filter(item => `${item[headerId].text}`.toLowerCase().indexOf(searchTerm) === 0);
                    }
                });

                return items;
            },
        },
        watch: {
            filteredItems() {
                this.totalFilteredItems = this.filteredItems.length;
                this.totalItems = this.items.length;
            }
        },
        mounted() {
        },
        created() {
        },
        methods: {
            sortColumn(columnId) {
                if (this.pagination.sortBy === columnId) {
                    // Same column, we invert sortering
                    this.pagination.descending = !this.pagination.descending;
                } else {
                    this.pagination.sortBy = columnId;
                    this.pagination.descending = false;
                }
            },
            sortFn (a, b) {
                let aValue, bValue;

                if (this.pagination.descending) {
                    aValue = a[this.pagination.sortBy].text;
                    bValue = b[this.pagination.sortBy].text;
                } else {
                    aValue = b[this.pagination.sortBy].text;
                    bValue = a[this.pagination.sortBy].text;
                }

                // If we are sorting text
                if (isNaN(aValue) || isNaN(bValue)) {
                    if (aValue > bValue) return 1;
                    if (aValue < bValue) return -1;
                    return 0;
                } else {
                    return aValue - bValue;
                }
            },
        },
    };
</script>

<style scoped>
  table {
    background-color: white;
    width: 100%;
    max-width: 100%;
    border-collapse: collapse;
    border-radius: 2px;
    border-spacing: 0;
    box-shadow: 0 2px 1px -1px rgba(0,0,0,.2),0 1px 1px 0 rgba(0,0,0,.14),0 1px 3px 0 rgba(0,0,0,.12)!important;
  }

  .table td, .table th {
    padding: .75rem;
    vertical-align: top;
  }

  .table th, tfoot td {
    color: rgba(0,0,0,.54);
    font-weight: 500;
    font-size: 12px;
  }
  thead {
    border-bottom: 2px solid darkgrey;
  }
  .table td {
    border-top: 1px solid #c2cfd6;
    font-weight: 400;
    font-size: 13px;
  }
  tbody tr {
    -webkit-transition: background .3s cubic-bezier(.25,.8,.5,1);
    transition: background .3s cubic-bezier(.25,.8,.5,1);
    will-change: background;
  }

  th.sorting {
    position: relative;
    padding-right: 25px!important;
    cursor: pointer;
  }
  .table thead .sorting:after {
    bottom: 10px;
    content: "\f0dc";
    font-family: "Font Awesome\ 5 Free";
    font-weight: 900;
    position: absolute;
    color: lightgrey;
    right: 7px;
  }

  .table thead .sorting.desc:after {
    content: "\f0de";
    color:black;
  }
  .table thead .sorting.asc:after {
    content: "\f0dd";
    color:black;
  }
  tr.textFilter {
    border-bottom: 1px solid #c2cfd6;
  }
  tr.textFilter input {
    width: 100%;
  }
  .scrollable tbody{
    display:block;
    overflow:auto;
    width:100%;
  }
  .scrollable thead tr, .scrollable tbody tr{
    display:table;
    width:100%;
    table-layout:fixed;/* even columns width , fix width of table too*/
  }
  .align-right {
    text-align: right;
  }
</style>
