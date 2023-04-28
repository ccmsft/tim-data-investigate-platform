<template>
  <v-navigation-drawer
    ref="drawer"
    absolute
    permanent
    left
    :style="navDrawerStyle"
    :expand-on-hover="navDrawer.isCollapsed"
    :mini-variant.sync="navDrawer.isMini"
  >
    <!-- The menu items at the top of the navigation drawer -->
    <v-toolbar width="100%" height="36px" density="compact">
      <NewQueryButton text icon tile>
        <v-icon>mdi-plus</v-icon>
      </NewQueryButton>
      <v-divider vertical style="height: 18px" />
      <v-btn
        text
        icon
        tile
        title="Reload templates"
        @click="onClickReloadTemplates"
      >
        <v-icon>mdi-refresh</v-icon>
      </v-btn>
      <v-divider vertical style="height: 18px" />
      <v-btn
        :disabled="treeView.tree.length === 0"
        text
        icon
        tile
        title="Remove selected"
        @click="onClickRemoveSelected"
      >
        <v-icon>mdi-delete</v-icon>
      </v-btn>
      <v-spacer />
      <v-btn
        text
        icon
        tile
        float-right="true"
        :title="
          navDrawer.isCollapsed
            ? navDrawer.pinButton.title
            : navDrawer.collapseButton.title
        "
        @click="onClickPinOrCollapse"
      >
        <v-icon>{{
          navDrawer.isCollapsed
            ? navDrawer.pinButton.icon
            : navDrawer.collapseButton.icon
        }}</v-icon>
      </v-btn>
    </v-toolbar>
    <v-divider />
    <!-- The list of queries -->
    <v-treeview-independent-parent
      ref="componentTreeView"
      v-model="treeView.tree"
      :items="getRootDisplayComponents"
      :class="{ 'closed-tree': navDrawer.isCollapsed && navDrawer.isMini }"
      :active="active"
      item-key="componentUuid"
      item-text="title"
      active-class="v-treeview-node--active active-query"
      selection-type="leaf"
      open-all
      activatable
      hoverable
      dense
      selectable
      style="width: auto; min-width: max-content"
      @update:active="updateActive"
    >
      <template #prepend="{ item, open, active }">
        <v-progress-circular
          v-if="item.state.isExecuting"
          indeterminate
          color="primary"
          :size="16"
          :width="2"
        />
        <v-icon v-else-if="item.state.error" color="error">
          mdi-alert
        </v-icon>
        <v-badge
          v-else-if="item.state.rowCount !== null && item.state.rowCount >= 0"
          :color="
            item.state.isVisited
              ? 'grey'
              : item.state.rowCount === 0
              ? 'warning'
              : 'info'
          "
          :content="
            item.state.rowCount > 9 ? '9+' : item.state.rowCount.toString()
          "
          overlap
        >
          <v-icon :class="active ? 'primary--text' : ''">
            {{ open ? "mdi-folder-open" : "mdi-folder" }}
          </v-icon>
        </v-badge>
        <template v-else>
          <v-icon style="opacity: 0.5" :class="active ? 'primary--text' : ''">
            {{ open ? "mdi-folder-open" : "mdi-folder" }}
          </v-icon>
        </template>
      </template>
      <template #label="{ item }">
        <span>{{
          isDraft(item) ? "[draft] " : isNew(item) ? "[new] " : ""
        }}</span>
        <span :class="isNew(item) ? 'font-weight-medium' : ''">
          {{ item.title }}
        </span>
      </template>
    </v-treeview-independent-parent>
  </v-navigation-drawer>
</template>

<script>
import { mapActions, mapGetters } from 'vuex';
import eventBus from '@/helpers/eventBus';
import {
  createNewQueryComponent,
  createNewTemplateQueryComponent,
} from '@/helpers/displayComponent';
import NewQueryButton from '@/components/NewQueryButton.vue';
import VTreeviewIndependentParent from '@/components/VTreeviewIndependentParent.vue';

export default {
  name: 'SideQueryTreeNew',
  components: {
    VTreeviewIndependentParent,
    NewQueryButton,
  },
  data() {
    return {
      navDrawer: {
        width: 275,
        minWidth: 200,
        maxWidth: 600,
        resizing: false,
        isCollapsed: false,
        isMini: false,
        border: null,
        position: null,
        collapseButton: {
          icon: 'mdi-chevron-left',
          title: 'Collapse pane',
        },
        pinButton: {
          icon: 'mdi-pin-outline',
          title: 'Pin pane',
        },
      },
      treeView: {
        tree: [],
        prevTree: [],
        activeUuid: null,
      },
    };
  },
  computed: {
    ...mapGetters('displayComponent', [
      'getRootDisplayComponents',
      'getParentDisplayComponent',
      'isComponent',
    ]),
    active() {
      return [this.activeUuid];
    },
    isNew: () => (item) => 
      item.state.rowCount !== null &&
      !item.state.isVisited &&
      item.state.error === null
    ,
    isDraft: () => (item) => 
      item.state.rowCount === null &&
      !item.state.isExecuting &&
      item.state.error === null
    ,
    navDrawerStyle() {
      if (!this.navDrawer.isCollapsed) {
        return {
          width: `${this.navDrawer.width}px`,
          minWidth: `${this.navDrawer.minWidth}px`,
          maxWidth: `${this.navDrawer.maxWidth}px`,
          transition: this.navDrawer.resizing ? 'initial' : '',
        };
      }
      return '';
    },
  },
  watch: {
    $route(to, from) {
      if (from.params.uuid !== to.params.uuid) {
        this.activeUuid = to.params.uuid;
      }
    },
  },
  mounted() {
    eventBus.$on('new:display-component', ({ uuid }) => {
      const parent = this.getParentDisplayComponent(uuid);
      if (parent) {
        this.$refs.componentTreeView.updateOpen(parent, true);
      }
    });
    eventBus.$on('update:kusto-results', ({ uuid }) => {
      // Results were complete on the active component
      if (uuid === this.activeUuid) {
        this.updateComponentState({ uuid, isVisited: true });
      }
    });
    this.loadAllDisplayComponents();
    this.navDrawer.border = this.$refs.drawer.$el.querySelector(
      '.v-navigation-drawer__border',
    );
    this.navDrawer.border.style.backgroundColor = 'light-grey';
    this.navDrawer.border.style.width = '3px';
    this.navDrawer.border.style.cursor = 'ew-resize';
    this.navDrawer.position = this.$refs.drawer.$el.classList.contains(
      'v-navigation-drawer--right',
    )
      ? 'right'
      : 'left';
    this.$emit('update:navWidth', this.navDrawer.width);
    this.setEvents();
  },
  methods: {
    ...mapActions('displayComponent', [
      'removeAllDisplayComponents',
      'updateComponentState',
      'loadAllDisplayComponents',
    ]),
    ...mapActions('queries', ['reloadQueries']),
    updateActive(newValue) {
      if (newValue.length === 1) {
        const uuid = newValue[0];
        this.updateComponentState({ uuid, isVisited: true });
        if (uuid !== this.activeUuid) {
          this.$router.push({ name: 'OpenTriage', params: { uuid } });
        }
      }
    },
    onClickRemoveSelected() {
      this.removeAllDisplayComponents({ uuidArray: this.tree });
      if (this.activeUuid && !this.isComponent(this.activeUuid)) {
        this.$router.replace('/');
      }
    },
    onClickNewQuery() {
      createNewQueryComponent('New query');
    },
    onClickNewView(queryTemplate) {
      createNewTemplateQueryComponent(
        queryTemplate.summary,
        queryTemplate,
        queryTemplate.getDefaultParams(),
        null,
        true,
      );
    },
    onClickPinOrCollapse() {
      this.navDrawer.isCollapsed = !this.navDrawer.isCollapsed;
      this.$emit('update:navCollapsed', this.navDrawer.isCollapsed);
      if (this.navDrawer.isCollapsed) {
        this.navDrawer.border.style.width = '1px';
        this.navDrawer.border.style.cursor = '';
        const evt = new MouseEvent('mouseleave', {
          view: window,
          bubbles: true,
          cancelable: true,
          clientX: 0,
        });
        this.$refs.drawer.$el.dispatchEvent(evt);
      } else {
        this.navDrawer.border.style.width = '3px';
        this.navDrawer.border.style.cursor = 'ew-resize';
        this.$emit('update:navWidth', this.navDrawer.width);
      }
    },
    async onClickReloadTemplates() {
      await this.reloadQueries();
    },
    resizeNavDrawer(event) {
      document.body.style.cursor = 'ew-resize';
      let width = this.navDrawer.direction === 'right'
        ? document.body.scrollWidth - event.clientX
        : event.clientX;
      if (width < this.navDrawer.minWidth) {
        width = this.navDrawer.minWidth;
      } else if (width > this.navDrawer.maxWidth) {
        width = this.navDrawer.maxWidth;
      }
      this.navDrawer.width = width;
      this.$emit('update:navWidth', this.navDrawer.width);
    },
    setEvents() {
      const navDrawerElement = this.$refs.drawer.$el;
      const drawerBorder = navDrawerElement.querySelector(
        '.v-navigation-drawer__border',
      );
      drawerBorder.addEventListener(
        'mousedown',
        (e) => {
          document.body.style.cursor = 'ew-resize';
          this.navDrawer.resizing = true;
          e.preventDefault();
          document.addEventListener('mousemove', this.resizeNavDrawer, false);
        },
        false,
      );
      document.addEventListener(
        'mouseup',
        () => {
          if (this.navDrawer.resizing) {
            this.navDrawer.resizing = false;
            document.body.style.cursor = '';
            document.removeEventListener('mousemove', this.resizeNavDrawer, false);
            document.removeEventListener('mouseup', this.resizeNavDrawer, false);
          }
        },
        false,
      );
    },
  },
};
</script>
<style>
.closed-tree {
}

.closed-tree .v-treeview-node__level,
.closed-tree .v-treeview-node__toggle--open,
.closed-tree .v-treeview-node__label,
.closed-tree .v-treeview-node__toggle,
.closed-tree .v-treeview-node__checkbox {
  display: none;
}

.v-treeview-node__label {
  cursor: pointer;
  white-space: break-spaces;
}

.v-list-item--dense,
.v-list--dense .v-list-item {
  min-height: 50px;
}

.v-treeview--dense .v-treeview-node__label {
  font-size: 0.8125rem;
  line-height: 1rem;
}

.active-query .v-treeview-node__label {
  pointer-events: none;
}
</style>
