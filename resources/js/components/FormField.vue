<template>
  <component :dusk="field.attribute" :is="field.fullWidth ? 'FullWidthField' : 'default-field'" :field="field"
    :errors="errors" :show-help-text="showHelpText" full-width-content>
    <template #field>

      <div class="flex flex-row gap-4">
        <div class="w-1/2" v-if="searchable">
          <input v-model="searchName" @keyup="onSearchKeyUp" placeholder="Search by course/lesson name or label"
            type="text" class="w-full form-control form-input form-control-bordered search-style" />
        </div>
        <div class="w-1/2" v-if="hasActiveTermFilter">
          <Multiselect v-model="selectedTerm" :options="availableTerms" placeholder="Filter by Active Term"
            track-by="value" :searchable="true" />
        </div>
      </div>


      <!-- Main groups -->
      <div ref="flexibleFieldContainer">
        <form-nova-flexible-content-group v-for="(group, groupIndex) in mainGroups"
          :dusk="field.attribute + '-' + groupIndex" :key="group.key" :field="field" :group="group" :index="groupIndex"
          :resource-name="resourceName" :resource-id="resourceId" :errors="errors" :mode="mode"
          @move-up="moveUp(group.key)" @move-down="moveDown(group.key,groupIndex)" @remove="remove(group.key)" />
      </div>

      <!-- Load more button -->
      <div class="load-more-container" v-if="showLoadMoreButton">
        <div @click="loadMore" class="btn btn-default btn-primary">
          Load More
        </div>
      </div>

      <!-- Divider and last group -->
      <hr v-if="showLoadMoreButton" class="last-divider mb-3" />
        <form-nova-flexible-content-group v-if="lastGroup" :dusk="field.attribute + '-last'" :key="lastGroup.key"
        :field="field" :group="lastGroup" :index="filteredGroupsFull.length - 1" :resource-name="resourceName"
        :resource-id="resourceId" :errors="errors" :mode="mode" @remove="remove(lastGroup.key)" :moveUpStatus="false" :moveDownStatus="false" />
 


      <component :layouts="layouts" :is="field.menu.component" :field="field" :limit-counter="limitCounter"
        :limit-per-layout-counter="limitPerLayoutCounter" :errors="errors" :resource-name="resourceName"
        :resource-id="resourceId" @addGroup="addGroup($event)" />
    </template>
  </component>
</template>

<script>
import FullWidthField from "./FullWidthField";
import Sortable from "sortablejs";
import { DependentFormField, HandlesValidationErrors, mapProps, } from "laravel-nova";
import Group from "../group";
import Multiselect from "@vueform/multiselect";

export default {
  mixins: [HandlesValidationErrors, DependentFormField],

  props: {
    ...mapProps(["resourceName", "resourceId", "mode"]),
  },

  components: { FullWidthField, Multiselect },

  data() {
    return {
      order: [],
      groups: {},
      files: {},
      sortableInstance: null,
      visibleCount: this.field.initialItemsCount !== null
      ? Math.max(this.field.initialItemsCount - 1, 1)
      : null,

      selectedTerm: null,
      availableTerms: [
        { value: "ALL", label: "All" },
        { value: "FIRST_TERM", label: "First Term" },
        { value: "SECOND_TERM", label: "Second Term" }
      ],
      searchable: this.field.searchable || null,
      searchName: '',
    
    };
  },

  computed: {
    layouts() {
      return this.field.layouts || false;
    },

    orderedGroups() {
      return this.order.reduce((groups, key) => {
        groups.push(this.groups[key]);
        return groups;
      }, []);
    },
    filteredGroupsFull() {
      return this.orderedGroups.filter(group => {
        let termMatch = false;
        let nameMatch = false;
        let match = false;
        if (this.hasActiveTermFilter && this.selectedTerm) {
          termMatch = this.matchesSelectedTerm(group);
          match = true;
        }
        if (this.searchable && this.searchName) {
          nameMatch = this.matchesSearchName(group);
          match = true;
        }
        return match ? termMatch || nameMatch : true;
      });
    },
    mainGroups() {
      const groups = this.filteredGroupsFull;
  
      if (!(this.paginate())) {
        return groups;
      }
      return groups.slice(0, this.visibleCount);
    },
    lastGroup() {
      const groups = this.filteredGroupsFull;
      if (!this.paginate() || groups.length <= this.visibleCount) {
        return null;
      }
      return groups[groups.length - 1];
    },

    showLoadMoreButton() {
      return this.paginate() && (this.filteredGroupsFull.length > this.visibleCount + 1);
    },
    limitCounter() {
      if (
        this.field.limit === null ||
        typeof this.field.limit == "undefined"
      ) {
        return null;
      }

      return this.field.limit - Object.keys(this.groups).length;
    },
    limitPerLayoutCounter() {
      return this.layouts.reduce((layoutCounts, layout) => {
        if (layout.limit === null) {
          layoutCounts[layout.name] = null;

          return layoutCounts;
        }
        let count = Object.values(this.groups).filter(
          (group) => group.name === layout.name,
        ).length;

        layoutCounts[layout.name] = layout.limit - count;

        return layoutCounts;
      }, {});
    },
   
    hasActiveTermFilter() {
      return this.field.hasActiveTermFilter;
    }
  },
  watch: {
    searchName() {
      this.emitButtonState();
    },
    selectedTerm() {
      this.emitButtonState();
    },
    
  },

  methods: {
    paginate() {
      return this.field.paginate;
    },
    onSearchKeyUp() {
      this.filteredGroupsFull;
    },
    emitButtonState() {
      console.log(this.searchName,this.selectedTerm);
      
      if (this.searchName == "" && this.selectedTerm == null) {
        Nova.$emit('set-button-state', false);
      } else {
        Nova.$emit('set-button-state', true);
      }
     
    },
    loadMore() {
      this.visibleCount = Math.min(
      this.visibleCount + (this.field.paginationCount || 0),
      this.orderedGroups.length - 1
    );
    },
    matchesSelectedTerm(group) {
      const term = this.extractGroupTerm(group);
      return term === this.selectedTerm;
    },

    extractGroupTerm(group) {
      return group.fields.find(
        f => f.attribute.endsWith('__course_heading_active_term') ||
          f.attribute.endsWith('__lesson_active_term')
      )?.value;
    },
    matchesSearchName(group) {
      const search = this.searchName.toLowerCase();
      const nameString = group.fields
        .filter(f => f.attribute.endsWith('__course_heading_name') ||
          f.attribute.endsWith('__lesson_name') ||
          f.attribute.endsWith('__lesson_label'))
        .map(f => f.value || '')
        .join(' ')
        .toLowerCase();
      return nameString.includes(search);
    },

    /*
    * Set the initial, internal value for the field.
    */
    setInitialValue() {
      this.value = this.currentField.value || [];
      this.files = {};
      this.populateGroups();
      this.$nextTick(this.initSortable.bind(this));
    },

    /**
     * Fill the given FormData object with the field's internal value.
     */
    fill(formData) {
      let key, group;

      this.value = [];
      this.files = {};
      
      for (var i = 0; i < this.order.length; i++) {
        key = this.order[i];
        
        var isShown =[...this.mainGroups, this.lastGroup].find(group => group?.key === key);
       
        console.log(key,isShown);
        if(!isShown){
          continue ;
        }
        group = this.groups[key].serialize();

        // Only serialize the group's non-file attributes
        this.value.push({
          layout: group.layout,
          key: group.key,
          attributes: group.attributes,

        });

        // Attach the files for formData appending
        this.files = { ...this.files, ...group.files };
      }

      this.appendFieldAttribute(formData, this.currentField.attribute);
      formData.append(
        this.currentField.attribute,
        this.value.length ? JSON.stringify(this.value) : "",
      );

      // Append file uploads
      for (let file in this.files) {
        formData.append(file, this.files[file]);
      }

      this.$nextTick(this.initSortable.bind(this));
    },

    /**
     * Register given field attribute into the parsable flexible fields register
     */
    appendFieldAttribute(formData, attribute) {
      let registered = [];

      if (formData.has("___nova_flexible_content_fields")) {
        registered = JSON.parse(
          formData.get("___nova_flexible_content_fields"),
        );
      }

      registered.push(attribute);

      formData.set(
        "___nova_flexible_content_fields",
        JSON.stringify(registered),
      );
    },

    /**
     * Update the field's internal value.
     */
    handleChange(value) {
      this.value = value || [];
      this.files = {};
      
      this.populateGroups();
      
    },

    /**
     * Set the displayed layouts from the field's current value
     */
    populateGroups() {
  
      this.order.splice(0, this.order.length);
      this.groups = {};
      for (var i = 0; i < this.value.length; i++) {
        this.addGroup(
          this.getLayout(this.value[i].layout),
          this.value[i].attributes,
          this.value[i].key,
          this.currentField.collapsed,
          true
        );
      }
    },
   
    /**
     * Retrieve layout definition from its name
     */
    getLayout(name) {
      if (!this.layouts) return;
      return this.layouts.find((layout) => layout.name === name);
    },

    /**
     * Append the given layout to flexible content's list
     */
    addGroup(layout, attributes, key, collapsed, populate = false) {
      if (!layout) return;

      collapsed = collapsed || false;

      let fields = attributes || JSON.parse(JSON.stringify(layout.fields));


      let group = new Group(
          layout.name,
          layout.title,
          fields,
          this.currentField,
          key,
          collapsed,
        );

      this.groups[group.key] = group;
      this.order.push(group.key);
      if (this.paginate() && !populate) {
        this.visibleCount++;
      }
    },


    /**
     * Move a group up
     */
    moveUp(key) {
      let index = this.order.indexOf(key);

      if (index <= 0) return;

      this.order.splice(index - 1, 0, this.order.splice(index, 1)[0]);
    },

    /**
     * Move a group down
     */
    moveDown(key,shownIndex = false) {
      let index = this.order.indexOf(key);

      if (index < 0 || index >= this.order.length - 1) return;

      this.order.splice(index + 1, 0, this.order.splice(index, 1)[0]);

      if(shownIndex + 1 == this.mainGroups.length){
        this.visibleCount = Math.min(
        this.visibleCount + (this.field.paginationCount || 0),
        this.orderedGroups.length - 1
        );
      }
    },

    /**
     * Remove a group
     */
    remove(key) {
      let index = this.order.indexOf(key);

      if (index < 0) return;

      this.order.splice(index, 1);
      delete this.groups[key];
    },

    initSortable() {
      const containerRef = this.$refs["flexibleFieldContainer"];

      if (!containerRef || this.sortableInstance) {
        return;
      }

      this.sortableInstance = Sortable.create(containerRef, {
        ghostClass: "nova-flexible-content-sortable-ghost",
        dragClass: "nova-flexible-content-sortable-drag",
        chosenClass: "nova-flexible-content-sortable-chosen",
        direction: "vertical",
        handle: ".nova-flexible-content-drag-button",
        scrollSpeed: 5,
        animation: 500,
        onEnd: (evt) => {
          const item = evt.item;
          const key = item.id;
          const oldIndex = evt.oldIndex;
          const newIndex = evt.newIndex;

          if (newIndex < oldIndex) {
            this.moveUp(key);
          } else if (newIndex > oldIndex) {
            this.moveDown(key);
          }
        },
      });
    },
  }
};
</script>
<style lang="scss">
@import "@vueform/multiselect/themes/default.scss";

.load-more-container {
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  padding: 8px;
  transition: transform 0.2s ease-in-out;

  &:hover {
    transform: scale(1.1);
  }
}

.search-style {
  height: 41px;
  position: relative;
  margin: 0 auto;
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  box-sizing: border-box;
  cursor: pointer;
  outline: none;
  border: var(--ms-border-width, 1px) solid var(--ms-border-color, #D1D5DB);
  border-radius: var(--ms-radius, 4px);
  background: var(--ms-bg, #FFFFFF);
  font-size: var(--ms-font-size, 1rem);
  min-height: calc(2* var(--ms-border-width, 1px) + var(--ms-font-size, 1rem)* var(--ms-line-height, 1.375) + 2* var(--ms-py, 0.5rem));
}
</style>
