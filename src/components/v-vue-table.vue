<template>
  <div>
    <slot name="header" :displayed-headers="filteredHeaders" :headers="headers" :selectColumns="selectColumns">
    </slot>
    <table :class="{
    table: true,
    scrollable: height !== 'auto' && filteredItems.length > 0
  }">
      <thead class="sorting" v-if="enableHeaders">
      <draggable tag="tr" v-model="filteredHeaders" @update="saveHeadersOrderToCookie">
        <th
          v-for="(header, index) in filteredHeaders"
          :key="`header${index}`"
          :class="{
          sorting: sort,
          asc: pagination.sortBy === header.id && !pagination.descending,
          desc: pagination.sortBy === header.id && pagination.descending,
        }"
          :style="{
          'text-align': header.align ? header.align : 'left',
        }"
          @click="sortColumn(header.id)"
          :title="header.text"
        >
          <span v-html="header.text"></span>
        </th>
      </draggable>
      <tr
        class="textFilter"
        v-if="textFilter"
      >
        <td
          v-for="(header, headerKey) in filteredHeaders"
          :key="`filter${headerKey}`"
        >
          <input
            :placeholder="header.textFilterString ? header.textFilterString : `Search ${header.text}`"
            type="text"
            v-if="header.textFilterString !== false"
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
        :key="`item${key}`"
      >
        <td
          v-for="(header, headerKey) in filteredHeaders"
          :key="`2${headerKey}`"
          :style="{
          'text-align': item[header.id] && item[header.id].align ? item[header.id].align : 'left',
        }"
          :title="item[header.id].text"
        >
          <div class="tdContent">
            <slot :header="header" :item="item" :text="item[header.id].text" :withoutHTML="item[header.id].withoutHTML" name="td">
            <span v-if="disableHtml">
              {{ item[header.id].withoutHTML }}
            </span>
              <span
                v-else
                v-html="item[header.id].text"
              >
            </span>
            </slot>
          </div>
        </td>
      </tr>
      </tbody>
      <tfoot v-if="enableFooter">
      <tr>
        <td
          :colspan="filteredHeaders.length"
        >
          <span class="align-right"
                v-if="filteredItems.length === items.length"
          >
          {{ computedText.footerCount }}
        </span>
          <span class="align-right" v-else>
          {{ computedText.filteredFooterCount }}
        </span>
        </td>
      </tr>
      </tfoot>
    </table>
  </div>
</template>

<script>
  import draggable from 'vuedraggable'
  export default {
    components: {
      draggable
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
      },
      cookieIdentifier: {
        type: String,
        default: 'vVueTable-cookie-hide',
      },
      disableHtml: {
        type: Boolean,
        default: false,
      },
      initHeaders: {
        type: Array,
        default: () => [],
      },
      enableFooter: {
        type: Boolean,
        default: false,
      },
      enableHeaders: {
        type: Boolean,
        default: true,
      },
      text: {
        type: Object,
        default: () => {
          return {
            footerCount: '%itemCount% items',
            filteredFooterCount: '%filteredItemCount%/%itemCount% items',
            columnSelectionHelp: 'Pick the columns you want to be displayed.',
            columnSelectionCancelButton: 'Cancel',
            columnSelectionSaveButton: 'Save',
            columnSelectionDisplayedButton: 'Displayed',
            columnSelectionDisplayButton: 'Display',
          };
        }
      },
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
        hiddenHeadersBuffer: null,
        headerOrderBuffer: null,
        filteredHeaders: [],
      };
    },
    computed: {
      filteredItems: {
        get() {
          let items = [];

          // Creating no HTML property for all items, and initializing missing headers
          this.items.forEach((anItem) => {
            const itemToAdd = {};
            this.filteredHeaders.forEach(aHeader => {
              itemToAdd[aHeader.id] = {};
              if (typeof anItem[aHeader.id] === 'undefined') {
                itemToAdd[aHeader.id].text = '';
                itemToAdd[aHeader.id].withoutHTML = '';
              } else {
                itemToAdd[aHeader.id].text = anItem[aHeader.id].text;
                itemToAdd[aHeader.id].withoutHTML = this.removeHTML(anItem[aHeader.id].text);
              }
            });

            // Make sure all provided variables are kept (for slots)
            Object.keys(anItem).forEach((key) => {
              if (typeof itemToAdd[key] === 'undefined') {
                itemToAdd[key] = anItem[key];
              }
            });
            items.push(itemToAdd);
          });

          if (this.pagination.sortBy.length > 0) {
            items = items.sort(this.sortFn)
          }

          Object.keys(this.pagination.textFilters).forEach(headerId => {
            const searchTerm = this.removeAccents(this.pagination.textFilters[headerId].toLowerCase());
            if (searchTerm.length > 0) {
              // Do we have a special AND/OR character?
              const and = searchTerm.split(',').reduce((acc, term) => {
                if (term.length) acc.push(term);
                return acc;
              }, []);
              const or = searchTerm.split('/').reduce((acc, term) => {
                if (term.length) acc.push(term);
                return acc;
              }, []);
              items = items.filter((item) => {
                let found = false;
                if (and.length > 1 && !or.length <= 1) {
                  found = true;
                  and.forEach((aTerm) => {
                    if (this.removeAccents(item[headerId].withoutHTML.toLowerCase()).indexOf(aTerm) === -1) {
                      found = false;
                    }
                  })
                } else if (or.length > 1 && and.length <= 1) {
                  found = false;
                  or.forEach((aTerm) => {
                    if (this.removeAccents(item[headerId].withoutHTML.toLowerCase()).indexOf(aTerm) !== -1) {
                      found = true;
                    }
                  })
                } else {
                  found = this.removeAccents(item[headerId].withoutHTML.toLowerCase()).indexOf(searchTerm) !== -1;
                }
                return found;
              });
            }
          });
          return items;
        }
      },
      computedText() {
        const toReturn = {};
        Object.keys(this.text).forEach((key) => {
          toReturn[key] = this.text[key];
          toReturn[key] = toReturn[key].replace('%filteredItemCount%', this.filteredItems.length);
          toReturn[key] = toReturn[key].replace('%itemCount%', this.items.length);
        });
        return toReturn;
      },
      displayedHeaders() {
        return this.headers.filter((aHeader) => {
          return !this.hiddenHeaders.includes(aHeader.id);
        });
      },
      headerOrder: {
        get() {
          if (!this.headerOrderBuffer) {
            return JSON.parse(this.$cookie.get(`${this.cookieIdentifier}-order`)) || [];
          }
          return this.headerOrderBuffer;
        },
        set(val) {
          this.headerOrderBuffer = val;
          this.$cookie.set(`${this.cookieIdentifier}-order`, JSON.stringify(val), Infinity);
        }
      },
      hiddenHeaders: {
        get() {
          if (!this.hiddenHeadersBuffer) {
            let headers =  JSON.parse(this.$cookie.get(`${this.cookieIdentifier}-hidden`));
            if (!headers) {
              headers = this.headers.filter((aHeader) => {
                return !this.initHeaders.includes(aHeader.id);
              }).map(header => header.id);
            }
            return headers || [];
          }
          return this.hiddenHeadersBuffer;
        },
        set(val) {
          this.hiddenHeadersBuffer = val;
          this.$cookie.set(`${this.cookieIdentifier}-hidden`, JSON.stringify(val), Infinity);
        },
      },
    },
    watch: {
      displayedHeaders() {
        this.updateFilteredHeaders();
      },
    },
    mounted() {

    },
    created() {
      this.updateFilteredHeaders();
    },
    updated() {
    },
    methods: {
      updateFilteredHeaders() {
        this.filteredHeaders = [];
        const addedIds = [];

        // Filling ranked header first
        this.headerOrder.forEach((headerId) => {
          const header = this.displayedHeaders.find(header => header.id === headerId);
          if (header && !addedIds.includes(headerId)) {
            addedIds.push(headerId);
            this.filteredHeaders.push(header);
          }
        });
        // filling the rest AFTER the ranked ones
        this.displayedHeaders.forEach((header) => {
          if (!addedIds.includes(header.id)) {
            this.filteredHeaders.push(header);
          }
        })
      },
      sortColumn(columnId) {
        if (this.pagination.sortBy === columnId) {
          // Same column, we invert sortering
          this.pagination.descending = !this.pagination.descending;
        } else {
          this.pagination.sortBy = columnId;
          this.pagination.descending = false;
        }
      },
      removeAccents(s) {
        return s.normalize('NFD').replace(/[\u0300-\u036f]/g, "")
      },
      saveHeadersOrderToCookie() {
        this.headerOrder = this.filteredHeaders.map((header) => {
          return header.id;
        });
      },
      sortFn (a, b) {
        let aValue, bValue;

        if (this.pagination.descending) {
          aValue = a[this.pagination.sortBy].withoutHTML;
          bValue = b[this.pagination.sortBy].withoutHTML;
        } else {
          aValue = b[this.pagination.sortBy].withoutHTML;
          bValue = a[this.pagination.sortBy].withoutHTML;
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
      removeHTML(content) {
        return `${content}`.replace(/(<([^>]+)>)/ig,''); // Remove HTML tags
      },
      selectColumns(colSelection) {
        const hiddenHeaders = [];
        this.headers.forEach((aHeader) => {
          if (!colSelection.includes(aHeader.id)) {
            hiddenHeaders.push(aHeader.id);
          }
        });
        this.hiddenHeaders = hiddenHeaders;
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
    border-bottom: 2px solid lightgrey;
  }
  .table td {
    border-top: 1px solid #c2cfd6;
    font-weight: 400;
    font-size: 13px;
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
  tr.textFilter input {
    width: 100%;
    border: 1px solid lightgrey;
    font-size: 12px;
    outline: none;
    margin:-2px;
    padding: 2px
  }

  table.scrollable {
    display: block;
  }
  .scrollable tbody{
    display:block;
    overflow:auto;
    width:100%;
  }
  .scrollable thead tr, .scrollable tbody tr, .scrollable tfoot tr{
    display:table;
    width:100%;
    table-layout:fixed;/* even columns width , fix width of table too*/
  }

  .scrollable th {
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .align-right {
    float: right;
  }


  table.scrollable div.tdContent {
    padding: 0;
    margin: 0;
    text-overflow: ellipsis;
    max-height: 20px;
    white-space: nowrap;
    overflow: hidden;
  }

  tfoot td {
    background-color: white;
  }
</style>
