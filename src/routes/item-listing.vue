<template>
  <div
    :class="{ 'no-results': noResults || emptyCollection }"
    class="item-listing">
    <v-header-bar :breadcrumb="links" info-toggle>
      <search-filter
        v-if="selection.length === 0"
        v-show="!emptyCollection"
        :filters="preferences.filters || []"
        :search-query="preferences.search_query || ''"
        :fields="fieldNames"
        :placeholder="resultCopy"
        @filter="updateListingPreferences('filters', $event)"
        @search="updateListingPreferences('search_query', $event)"
        @clearFilters="clearFilters" />
      <template slot="buttons">
        <v-header-button
          v-if="selection.length > 1"
          key="edit"
          icon="mode_edit"
          color="warning"
          :label="$t('batch_edit')"
          @click="batchEdit" />
        <v-header-button
          v-if="selection.length"
          key="delete"
          icon="close"
          color="danger"
          :label="$t('delete')"
          @click="remove" />
        <v-header-button
          icon="add"
          key="add"
          color="action"
          :label="$t('new')"
          @click="add" />
      </template>
    </v-header-bar>

    <loader v-if="loading || hydrating" area="content" transparent :z-index="9" />

    <v-info-sidebar>
      <template slot="system">
        <label for="listing">{{ $t('view_type') }}</label>
        <v-select
          id="listing"
          :options="listingNames"
          :value="viewType"
          name="listing"
          @input="updateListingPreferences('view_type', $event)" />
      </template>

      <div class="sidebar-content">
        <v-listing-options
          v-if="hydrating === false"
          :id="viewType"
          :collection="collection"
          :primary-key-field="primaryKeyField"
          :fields="fields"
          :items="items"
          :options="viewOptions"
          :loading="loading"
          :query="preferences.view_query"
          :selection="selection"
          link="__link__"
          @query="updateListingPreferences('view_query', $event)"
          @select="selection = $event"
          @input="updateListingPreferences('view_options', $event)" />
        <button
          :class="{ active: isUserPreference }"
          :disabled="!isUserPreference"
          class="reset-preferences"
          @click="resetPreferences">{{ $t('reset_preferences') }}</button>
      </div>
    </v-info-sidebar>

    <v-error
      v-if="!hydrating && noResults"
      :title="$t('no_results')"
      :body="$t('no_results_body')"
      icon="search" />

    <v-error
      v-if="!hydrating && emptyCollection"
      :title="$t('empty_collection')"
      :body="$t('empty_collection_body')"
      icon="web_asset" />

    <v-listing
      v-else-if="!hydrating && !noResults && items.length > 0"
      :id="viewType"
      :primary-key-field="primaryKeyField"
      :fields="fields"
      :items="items"
      :options="viewOptions"
      :selection="selection"
      :query="preferences.view_query || {}"
      :link="readonly ? null : '__link__'"
      @select="selection = $event"
      @input="updateListingPreferences('view_options', $event)"
      @query="updateListingPreferences('view_query', $event)" />

    <v-prompt
      v-if="bookmarkModal"
      v-model="bookmarkTitle"
      :message="$t('name_bookmark')"
      @confirm="saveBookmark"
      @cancel="bookmarkModal = false" />

    <v-confirm
      v-if="batchDeleteModal"
      :message="$tc('batch_delete_confirm', selection.length, { count: selection.length })"
      :confirm-text="$t('delete')"
      @confirm="batchDelete"
      @cancel="batchDeleteModal = false" />
  </div>
</template>

<script>
import formatFilters from "../helpers/format-filters";
import SearchFilter from "../components/search-filter/search-filter.vue";
import VError from "../components/error.vue";

export default {
  name: "item-listing",
  components: {
    SearchFilter,
    VError
  },
  props: {
    collection: {
      type: String,
      required: true
    },
    readonly: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      items: [],
      meta: null,
      error: null,
      loading: false,
      hydrating: true,
      selection: [],
      bookmarkTitle: "",
      bookmarkModal: false,
      batchDeleteModal: false
    };
  },
  computed: {
    noResults() {
      if (this.hydrating || this.loading) return false;

      // Don't show no-results if there aren't any items in the db to begin with
      if (this.emptyCollection) return false;

      return (this.items && this.items.length === 0) || false;
    },
    emptyCollection() {
      if (this.hydrating || this.loading) return false;

      return (this.meta && this.meta.total_count === 0) || false;
    },
    currentBookmark() {
      if (this.hydrating) return null;

      const bookmarks = this.$store.state.bookmarks;
      const preferences = {
        collection: this.preferences.collection,
        search_query: this.preferences.search_query,
        filters: this.preferences.filters,
        view_options: this.preferences.view_options,
        view_type: this.preferences.view_type,
        view_query: this.preferences.view_query
      };

      const currentBookmark = bookmarks.filter(bookmark => {
        const bookmarkPreferences = {
          collection: bookmark.collection,
          search_query: bookmark.search_query,
          filters: bookmark.filters,
          view_options: bookmark.view_options,
          view_type: bookmark.view_type,
          view_query: bookmark.view_query
        };

        return this.$lodash.isEqual(bookmarkPreferences, preferences);
      })[0];

      return currentBookmark || null;
    },
    fields() {
      const stateFields =
        (this.$store.state.fields &&
          this.$store.state.fields[this.collection]) ||
        {};

      return this.$lodash.keyBy(
        Object.values(stateFields)
          .filter(field => {
            if (field.interface === "primary-key") return true;

            if (field.hidden_list === true || field.hidden_list === 1) {
              return false;
            }

            return true;
          })
          .map(field => ({
            ...field,
            name: this.$t(`fields-${this.collection}-${field.field}`)
          })),
        "field"
      );
    },
    fieldNames() {
      if (!this.fields) return {};
      const translatedNames = {};
      Object.keys(this.fields).forEach(name => {
        translatedNames[name] = this.$t(`fields-${this.collection}-${name}`);
      });
      return translatedNames;
    },
    links() {
      if (this.collection === "directus_users") {
        return [
          {
            name: this.$t("user_directory"),
            path: "/users"
          }
        ];
      }
      if (this.collection === "directus_files") {
        return [
          {
            name: this.$t("file_library"),
            path: "/files"
          }
        ];
      }
      if (this.collection === "directus_activity") {
        return [
          {
            name: this.$t("activity"),
            path: "/activity"
          }
        ];
      }
      return [
        {
          name: this.$t("collections"),
          path: "/collections"
        },
        {
          name: this.$t(`collections-${this.collection}`),
          path: `/collections/${this.collection}`
        }
      ];
    },
    listingNames() {
      if (!this.$store.state.extensions.listings) return {};
      const translatedNames = {};
      Object.keys(this.$store.state.extensions.listings).forEach(id => {
        translatedNames[id] = this.$store.state.extensions.listings[id].name;
      });
      return translatedNames;
    },
    itemParams() {
      let params = {
        fields: "*.*", // by default, we fetch all items + 1 level of relational items
        meta: "total_count"
      };

      Object.assign(params, this.preferences.view_query || {});

      if (this.preferences.search_query) {
        params.q = this.preferences.search_query;
      }

      if (this.preferences.filters && this.preferences.filters.length > 0) {
        params = {
          ...params,
          ...formatFilters(this.preferences.filters)
        };
      }

      if (params.fields) {
        // Make sure all selected fields are retrieved one level deep (to be able to show relational
        //  items)
        params.fields = params.fields
          .split(",")
          .map(field => (field.endsWith(".*") ? field : `${field}.*`));

        // Make sure the primaryKey is always fetched
        if (params.fields.includes(this.primaryKeyField) === false) {
          params.fields.push(this.primaryKeyField);
        }
      }

      return params;
    },
    preferences() {
      return this.$store.state.listingPreferences[this.collection] || {};
    },
    isUserPreference() {
      return (
        (this.preferences &&
          this.preferences.user === this.$store.state.currentUser.id) ||
        false
      );
    },
    primaryKeyField() {
      const primaryKeyField = this.$lodash.find(
        this.fields,
        { interface: "primary-key" } // TODO: check for data type instead of interface
      );

      return primaryKeyField && primaryKeyField.field;
    },
    resultCopy() {
      if (this.loading) return this.$t("loading");
      if (!this.meta) return "";

      const count = this.items && this.items.length;

      const isFiltering =
        !this.$lodash.isEmpty(this.preferences.filters) ||
        (!this.$lodash.isNil(this.preferences.search_query) &&
          this.preferences.search_query.length > 0);

      return isFiltering
        ? this.$tc("item_count_filter", count, { count })
        : this.$tc("item_count", count, { count });
    },
    viewType() {
      return this.preferences.view_type || "tabular";
    },
    viewOptions() {
      return (
        (this.preferences.view_options &&
          this.preferences.view_options[this.viewType]) ||
        {}
      );
    }
  },
  watch: {
    collection: "hydrate",

    /**
     * If the search_query, filters, or view_query changes: fetch the fresh items
     */
    preferences: {
      deep: true,
      handler(newVal, oldVal) {
        if (this.$lodash.isEmpty(oldVal) || this.$lodash.isEmpty(newVal))
          return;

        if (
          this.$lodash.isEqual(newVal.search_query, oldVal.search_query) ===
            false ||
          this.$lodash.isEqual(newVal.filters, oldVal.filters) === false ||
          this.$lodash.isEqual(newVal.view_query, oldVal.view_query) === false
        ) {
          this.getItems();
        }
      }
    }
  },
  created() {
    this.hydrate();
  },
  methods: {
    /**
     * Fetch component crucial information before rendering
     */
    hydrate() {
      this.hydrating = true;
      this.error = null;
      this.items = [];
      this.meta = null;

      Promise.all([
        this.$store.dispatch("getFields", this.collection),
        this.$store.dispatch("getListingPreferences", this.collection)
      ])
        .then(() => {
          this.getItems();
        })
        .then(() => {
          this.hydrating = false;
        })
        .catch(error => {
          this.error = error;
          this.hydrating = false;
        });
    },

    /**
     * Fetch (new) items
     */
    getItems() {
      this.error = null;
      this.loading = true;

      return this.$api
        .getItems(this.collection, this.itemParams)
        .then(res => {
          this.meta = res.meta;
          return res.data;
        })
        .then(items => {
          if (
            this.collection === "directus_users" ||
            this.collection === "directus_files"
          ) {
            this.items = items.map(item => ({
              ...item,
              __link__: `/${this.collection.substring(9)}/${
                item[this.primaryKeyField]
              }`
            }));
            return;
          }

          if (this.readonly) {
            this.items = items;
            return;
          }

          this.items = items.map(item => ({
            ...item,
            __link__: `/collections/${this.collection}/${
              item[this.primaryKeyField]
            }`
          }));
        })
        .then(() => {
          this.loading = false;
        })
        .catch(error => {
          this.error = error;
          this.loading = false;
        });
    },

    /**
     * Update a single column in directus_collection_presets mapped to the current collection
     */
    updateListingPreferences(field, value) {
      let val = value;
      if (Array.isArray(value) && value.length === 0) val = null;
      let info = null;

      /*
       * If the view options are saved, nest them in the currently-in-use view_type to prevent
       *   the settings of other view types to be overriden. This allows the user to switch back and
       *   forth between list views, without losing settings between them.
       */
      if (field === "view_options") {
        info = {
          collection: this.collection,
          updates: {
            view_options: {
              ...(this.preferences.view_options || {}),
              [this.viewType]: val
            }
          }
        };
      } else {
        info = {
          collection: this.collection,
          updates: {
            [field]: val
          }
        };
      }
      this.$store.dispatch("setListingPreferences", info);
    },

    resetPreferences() {
      this.$store.dispatch("resetPreferences", { collection: this.collection });
    },

    batchEdit() {
      let route = `/collections/${this.collection}/${this.selection.join()}`;
      if (
        this.collection === "directus_users" ||
        this.collection === "directus_files"
      ) {
        route = `/${this.collection.substring(9)}/${this.selection.join()}`;
      }
      this.$router.push(route);
    },

    remove() {
      if (this.selection.length === 1) {
        this.deleteSingle(this.selection[0]);
      } else {
        this.batchDeleteModal = true;
      }
    },

    batchDelete() {
      const primaryKeys = this.selection;

      this.selection = [];
      this.items = this.items.filter(
        item => primaryKeys.includes(item[this.primaryKeyField]) === false
      );

      this.batchDeleteModal = false;

      this.$api.deleteItems(this.collection, primaryKeys).catch(console.error);
    },

    deleteSingle(primaryKey) {
      this.selection = this.selection.filter(
        itemPrimaryKey => itemPrimaryKey !== primaryKey
      );
      this.items = this.items.filter(
        item => item[this.primaryKeyField] !== primaryKey
      );

      this.$api.deleteItem(this.collection, primaryKey).catch(console.error);
    },

    add() {
      let route = `/collections/${this.collection}/+`;
      if (
        this.collection === "directus_users" ||
        this.collection === "directus_files"
      ) {
        route = `/${this.collection.substring(9)}/+`;
      }
      this.$router.push(route);
    },

    saveBookmark() {
      const preferences = { ...this.preferences };
      preferences.user = this.$store.state.currentUser.id;
      preferences.title = this.bookmarkTitle;
      delete preferences.id;
      delete preferences.group;

      if (!preferences.collection) {
        preferences.collection = this.collection;
      }

      this.$store
        .dispatch("saveBookmark", preferences)
        .then(() => {
          this.bookmarkModal = false;
        })
        .catch(console.error);
    },

    clearFilters() {
      this.updateListingPreferences("filters", null);
      this.updateListingPreferences("search_query", null);
    }
  }
};
</script>

<style lang="scss" scoped>
.item-listing:not(.no-results) {
  padding-bottom: var(--page-padding-bottom);
}

.bookmark {
  margin-left: 10px;
  opacity: 0.4;
  transition: opacity var(--fast) var(--transition);

  &:hover {
    opacity: 1;
  }

  i {
    font-size: 24px;
    height: 20px;
  }
}

.bookmark.active {
  opacity: 1;

  i {
    color: var(--accent);
  }
}

.bookmark-name {
  color: var(--accent);
  margin-left: 5px;
  margin-top: 3px;
  font-size: 0.77em;
  line-height: 1.1;
  font-weight: 700;
  text-transform: uppercase;
}

.sidebar-content {
  padding-bottom: 30px;
}

.reset-preferences {
  color: var(--light-gray);
  transition: var(--fast) var(--transition);
  text-align: center;
  font-size: 11px;
  position: fixed;
  bottom: 0px;
  right: 0;
  padding: 5px 0;
  background-color: var(--white);
  width: calc(var(--nav-sidebar-width) - 1px);
  transform: translateY(100%) scaleX(0.9);

  &:hover,
  .user-is-tabbing &:focus {
    color: var(--dark-gray);
  }

  &.active {
    transform: translateY(0) scaleX(1);
  }
}
</style>
