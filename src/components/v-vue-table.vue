<template>
  <table :class="{
    table: true,
    scrollable: height !== 'auto' && filteredItems.length > 0
  }">
    <colgroup>
      <col
        v-for="(header, index) in variableHeaders"
        :key="`col${index}`"
        :class="{
          toHide: colSelectionMode && header.toHide,
          toShow: colSelectionMode && !header.toHide,
        }"
      />
    </colgroup>
    <thead class="sorting">
    <draggable v-model="variableHeaders" element="tr" @update="saveHeadersToCookie">
      <th
        v-for="(header, index) in variableHeaders"
        :key="`header${index}`"
        :class="{
          sorting: sort,
          asc: pagination.sortBy === header.id && !pagination.descending,
          desc: pagination.sortBy === header.id && pagination.descending,
          toHide: colSelectionMode && header.toHide,
          toShow: colSelectionMode && !header.toHide,
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
        v-for="(header, headerKey) in variableHeaders"
        :class="{
          toHide: colSelectionMode && header.toHide,
        }"
        :key="`filter${headerKey}`"
      >
        <input
          :placeholder="header.textFilterString ? header.textFilterString : `Search ${header.text}`"
          type="text"
          :class="{
            toHide: colSelectionMode && header.toHide,
            toShow: colSelectionMode && !header.toHide,
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
      <slot :item="item">
        <td
          v-for="(header, headerKey) in variableHeaders"
          :key="`2${headerKey}`"
          v-if="item[header.id]"
          :style="{
            'text-align': item[header.id] && item[header.id].align ? item[header.id].align : 'left',
          }"
          :class="{
            toHide: colSelectionMode && header.toHide,
            toShow: colSelectionMode && !header.toHide,
          }"
        >
          <span v-if="disableHtml">
            {{ item[header.id].text | removeHTML }}
          </span>
          <span
            v-else
            v-html="item[header.id].text"
          >
          </span>
        </td>
      </slot>
    </tr>
    </tbody>
    <tfoot>
    <tr v-if="colSelectionMode">
      <td
        v-for="(header, headerKey) in variableHeaders"
        :key="`colSelection${headerKey}`"
        class="colSelectTD"
      >
        <button
          :class="{
            toShow: colSelectionMode && !header.toHide,
            toHide: colSelectionMode && header.toHide,
            'colSelectButtons': true
          }"
          @click="$set(variableHeaders[headerKey], 'toHide', !header.toHide)"
        >
          <i class="fa fa-check"></i>
          <template v-if="header.toHide === true">
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
        :colspan="variableHeaders.length"
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
        list: [],
        variableHeaders: [], // Used only to be updated (thanks to sorting for example)
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
      computedText() {
        const toReturn = {};
        Object.keys(this.text).forEach((key) => {
          toReturn[key] = this.text[key];
          toReturn[key] = toReturn[key].replace('%filteredItemCount%', this.filteredItems.length);
          toReturn[key] = toReturn[key].replace('%itemCount%', this.items.length);
        });
        return toReturn;
      },
    },
    watch: {
      filteredItems() {
        this.totalFilteredItems = this.filteredItems.length;
        this.totalItems = this.items.length;
      },
      headers() {
        this.variableHeaders = [];
        this.updateHeaders();
      }
    },
    mounted() {
    },
    created() {
      this.updateHeaders();
    },
    methods: {
      updateHeaders() {
          const newVariableHeaders = [];
          const addedIds = [];

          // Variable headers that are currently displayed are treated first (for the order)
          // All hidden headers come after, no matter what
          if (this.variableHeaders.length === 0
            && this.headers.length > 0
            && this.$cookie.get(this.cookieIdentifier)
          ) {
            const cookieData = JSON.parse(this.$cookie.get(this.cookieIdentifier));

            cookieData.forEach((data) => {
              const header = this.headers.find(header => header.id === data.id);

              if (header) {
                header.hidden = !!data.hidden; // !! is not an error
                header.toHide = header.hidden;

                if (!header.hidden) { // Add to the stack only if they are displayed
                  newVariableHeaders.push(header);
                  addedIds.push(header.id);
                }
              }
            });
          } else {
            this.variableHeaders.forEach((header) => { // Could be read as "currently displayed"
              if (this.colSelectionMode || !header.hidden) {
                newVariableHeaders.push(header);
                addedIds.push(header.id);
              }
            });
          }

        // Could be read as "currently not displayed"
        this.headers.forEach((header) => {
          if ((this.colSelectionMode || !header.hidden) && !addedIds.includes(header.id)) {
            newVariableHeaders.push(header);
          }
        });

        this.variableHeaders = newVariableHeaders;
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
        this.updateHeaders();
      },
      cancelColumnSelection() {
        this.variableHeaders.forEach((header) => {
          header.toHide = header.hidden;
        });
        this.colSelectionMode = false;
        this.updateHeaders();
      },
      validateColumnSelection() {
        this.variableHeaders.forEach((header) => {
          header.hidden = header.toHide;
        });
        this.colSelectionMode = false;
        this.updateHeaders();
        this.saveHeadersToCookie();
      },
      saveHeadersToCookie() {
        const cookieToSave = [];
        const addedIds = [];
        // Variable headers that are currently displayed are treated first (for the order)
        // All hidden headers come after, no matter what
        this.variableHeaders.forEach((header) => { // Could be read as "currently displayed"
          if (this.colSelectionMode || !header.hidden) {
            cookieToSave.push({
              id: header.id,
              hidden: header.hidden,
            });
            addedIds.push(header.id);
          }
        });

        this.headers.forEach((header) => {
          if (!addedIds.includes(header.id)) {
            cookieToSave.push({
              id: header.id,
              hidden: header.hidden,
            })
          }
        });
        this.$cookie.set(this.cookieIdentifier, JSON.stringify(cookieToSave), Infinity);
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
    filters:{
      removeHTML(content) {
          return content.replace(/(<([^>]+)>)/ig,''); // Remove HTML tags
      }
    }
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
  tr.textFilter td {
    border-top: 2px solid darkgrey;
  }
  tr.textFilter input {
    width: 100%;
    border: 1px solid lightgrey;
    font-size: 12px;
    outline: none;
    margin:-2px;
    padding: 2px
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
