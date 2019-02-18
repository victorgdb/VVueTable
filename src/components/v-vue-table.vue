<template>
  <table :class="{
    table: true,
    scrollable: height !== 'auto' && filteredItems.length > 0
  }">
    <thead class="sorting">
    <draggable element="tr" v-model="filteredHeaders" @update="saveHeadersOrderToCookie">
      <th
        v-for="(header, index) in filteredHeaders"
        :key="`header${index}`"
        :class="{
          sorting: sort,
          asc: pagination.sortBy === header.id && !pagination.descending,
          desc: pagination.sortBy === header.id && pagination.descending,
          toHide: colSelectionMode && hidingHeaders.includes(header.id),
          toShow: colSelectionMode && !hidingHeaders.includes(header.id),
        }"
        :style="{
          'text-align': header.align ? header.align : 'left',
        }"
        @click="sortColumn(header.id)"
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
        :class="{
          toHide: colSelectionMode && hidingHeaders.includes(header.id),
          toShow: colSelectionMode && !hidingHeaders.includes(header.id),
        }"
        :key="`filter${headerKey}`"
      >
        <input
          :placeholder="header.textFilterString ? header.textFilterString : `Search ${header.text}`"
          type="text"
          :class="{
            toHide: colSelectionMode && hidingHeaders.includes(header.id),
            toShow: colSelectionMode && !hidingHeaders.includes(header.id),
          }"
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
        :class="{
          toHide: colSelectionMode && hidingHeaders.includes(header.id),
          toShow: colSelectionMode && !hidingHeaders.includes(header.id),
        }"
        :style="{
          'text-align': item[header.id] && item[header.id].align ? item[header.id].align : 'left',
        }"
      >
        <div class="tdContent">
          <slot :text="item[header.id].text" :withoutHTML="item[header.id].withoutHTML">
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
    <tr v-show="colSelectionMode">
      <td
        v-for="(header, headerKey) in filteredHeaders"
        :key="`colSelection${headerKey}`"
        class="colSelectTD"
      >
        <button
          :class="{
            toShow: colSelectionMode && !hidingHeaders.includes(header.id),
            toHide: colSelectionMode && hidingHeaders.includes(header.id),
            'colSelectButtons': true
          }"
          :id="`colSelectionButton${headerKey}`"
          @click="toggleHeaderDisplay(header.id)"
        >
          <i class="fa fa-check"></i>
          <template v-if="hidingHeaders.includes(header.id)">
            {{ computedText.columnSelectionDisplayButton }}
          </template>
          <template v-else>
            {{ computedText.columnSelectionDisplayedButton }}
          </template>
        </button>
      </td>
    </tr>
    <tr>
      <td
        :colspan="filteredHeaders.length"
      >
        <button
          :class="[
            'colSelectButton',
            'colSelectButtons'
          ]"
          v-if="!colSelectionMode"
          @click="selectColumns">
          <i class="colSelectIcon fa fa-cog"></i>
        </button>

        <template v-else>
          <span>
            {{ computedText.columnSelectionHelp }}
          </span>

          <button
            class="colSelectButtons colSelectCancelButton"
            @click="cancelColumnSelection"
          >
            {{ computedText.columnSelectionCancelButton }}
          </button>
          <button
            class="colSelectButtons colSelectSaveButton"
            @click="validateColumnSelection"
          >
            <i class="fa fa-check"></i>
            {{ computedText.columnSelectionSaveButton }}
          </button>
        </template>


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
      enableFooter: {
        type: Boolean,
        default: false,
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
        colSelectionMode: false,
        totalItems: 0,
        totalFilteredItems: 0,
        hidingHeaders: [],
        hiddenHeaders: [],
        headerOrder: null,
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
            items.push(itemToAdd);
          });

          if (this.pagination.sortBy.length > 0) {
            items = items.sort(this.sortFn)
          }

          Object.keys(this.pagination.textFilters).forEach(headerId => {
            const searchTerm = this.pagination.textFilters[headerId].toLowerCase();
            if (searchTerm.length > 0) {
              // Make sure we work with strings
              items = items.filter((item) => {
                return item[headerId].withoutHTML.toLowerCase().indexOf(searchTerm) === 0;
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
          return this.colSelectionMode || !this.hiddenHeaders.includes(aHeader.id);
        });
      }
    },
    watch: {
      displayedHeaders() {
        this.updateFilteredHeaders();
      }
    },
    mounted() {
    },
    created() {
      this.hiddenHeaders = JSON.parse(this.$cookie.get(`${this.cookieIdentifier}-hidden`)) || [];
      this.headerOrder = JSON.parse(this.$cookie.get(`${this.cookieIdentifier}-order`)) || [];
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
          if (header) {
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
      selectColumns() {
        this.colSelectionMode = true;
        this.hidingHeaders = [...this.hiddenHeaders];
      },
      cancelColumnSelection() {
        this.colSelectionMode = false;
      },
      validateColumnSelection() {
        this.hiddenHeaders =  [...this.hidingHeaders];

        this.colSelectionMode = false;
        this.saveHeadersDisplayToCookie();
      },
      toggleHeaderDisplay(headerId) {
        if (this.hidingHeaders.includes(headerId)) {
          // We remove this item from the stack
          this.$delete(this.hidingHeaders, this.hidingHeaders.findIndex(hidingHeader => hidingHeader === headerId));
        } else {
          // We add it
          this.hidingHeaders.push(headerId);
        }
      },
      saveHeadersDisplayToCookie() {
        this.$cookie.set(`${this.cookieIdentifier}-hidden`, JSON.stringify(this.hiddenHeaders), Infinity);
      },
      saveHeadersOrderToCookie() {
        this.headerOrder = [];

        this.filteredHeaders.forEach((aHeader) => {
          this.headerOrder.push(aHeader.id)
        });

        this.$cookie.set(`${this.cookieIdentifier}-order`, JSON.stringify(this.headerOrder), Infinity);
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
      }
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
  .align-right {
    float: right;
  }

  .toShow{
    background-color: #D5F5E3;
  }


  /* Basic buttons */
  .colSelectButtons {
    display: inline-block;
    border: 1px solid #A6ACAF;
    outline: none;
    border-radius: .25rem;
    padding: .08rem .35rem;
    background-color:white;
    color: #A6ACAF;
  }
  .colSelectButtons:hover {
    background-color: white;
    color:  rgba(0,0,0,.54);
    border: 1px solid  rgba(0,0,0,.54);
  }

  .colSelectCancelButton {
    margin-left:10px;
  }

  /* setttings buttons */
  .colSelectButton {
    color:lightgrey;
    border:none;
    margin:1px;
  }
  .colSelectButton:hover {
    color: rgba(0,0,0,.54);
    border: 1px solid rgba(0,0,0,.54);
    margin: 0;
    background-color:white;
  }

  table.scrollable div.tdContent {
    padding: 0;
    margin: 0;
    text-overflow: ellipsis;
    max-height: 20px;
    white-space: nowrap;
    overflow: hidden;
  }
  /* Buttons to show */
  button.toShow {
    background-color:white;
    color:#52BE80;
    border: 1px solid #52BE80;
  }

  button.toShow:hover {
    background-color:white;
    color:#239B56;
    border: 1px solid #239B56;
  }

  /* Basic buttons */
  button.toHide i {
    display: none;
  }

  td.toHide, th.toHide, input.toHide::placeholder{
    color:#c2cfd6;
    background-color:white;
  }

  .textFilter td.toHide {
    border-top: 2px solid lightgrey;
    border-bottom: 2px solid lightgrey;
  }
  .colSelectSaveButton {
    background-color:#52BE80;
    color:white;
  }
  .colSelectSaveButton:hover {
    background-color:#239B56;
    border: 1px solid #A6ACAF;
    color:white;
  }
  tfoot td {
    background-color: white;
  }
  .colSelectTD {
    text-align:center;
    background-color: white;
  }
</style>
