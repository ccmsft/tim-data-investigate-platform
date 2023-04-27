<template>
  <v-navigation-drawer
    ref="drawer"
    absolute
    permanent
    left
    :width="width"
    :expand-on-hover="isCollapsed"
    :mini-variant.sync="mini"
  >
    <!-- The menu items at the top of the navigation drawer -->
    <v-toolbar
      width="100%"
      height="36px"
      density="compact"
    >
      <NewQueryButton
        text
        icon
        tile
      >
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
        :disabled="tree.length === 0"
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
        :title="isCollapsed ? pinButton.title : collapseButton.title"
        @click="onClickPinOrCollapse"
      >
        <v-icon>{{ isCollapsed ? pinButton.icon : collapseButton.icon }}</v-icon>
      </v-btn>
    </v-toolbar>
    <v-divider />
    <!-- The list of queries -->
    <v-treeview-independent-parent
      ref="componentTreeView"
      v-model="tree"
      :items="getRootDisplayComponents"
      :class="{ 'closed-tree': isCollapsed && mini }"
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
      style="width: max-content"
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
        <div style="width=max-content">
          <span>{{ isDraft(item) ? "[draft] " : isNew(item) ? "[new] " : "" }}</span>
          <span :class="isNew(item) ? 'font-weight-medium' : ''">
            {{ item.title }}</span>
        </div>
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
      width: 275,
      minWidth: 200,
      maxWidth: 600,
      isCollapsed: false,
      mini: null,
      tree: [],
      prevTree: [],
      activeUuid: null,
      collapseButton: {
        icon: 'mdi-chevron-left',
        title: 'Collapse pane',
      },
      pinButton: {
        icon: 'mdi-pin-outline',
        title: 'Pin pane',
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
    isNew: () => (item) => (
      item.state.rowCount !== null
      && !item.state.isVisited
      && item.state.error === null
    ),
    isDraft: () => (item) => (
      item.state.rowCount === null
      && !item.state.isExecuting
      && item.state.error === null
    ),
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
    this.navDrawerBorder = this.$refs.drawer.$el.querySelector(
      '.v-navigation-drawer__border',
    );
    this.navDrawerBorder.style.backgroundColor = 'light-grey';
    this.navDrawerBorder.style.width = '3px';
    this.navDrawerBorder.style.cursor = 'ew-resize';
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
      console.log('onClickPinCollapse');
      this.isCollapsed = !this.isCollapsed;
      if (this.isCollapsed) {
        this.navDrawerBorder.style.width = '1px';
        this.navDrawerBorder.style.cursor = '';
        const evt = new MouseEvent('mouseleave', {
          view: window,
          bubbles: true,
          cancelable: true,
          clientX: 20,
        });
        this.$refs.drawer.$el.dispatchEvent(evt);
      } else {
        this.navDrawerBorder.style.width = '3px';
        this.navDrawerBorder.style.cursor = 'ew-resize';
      }
    },
    async onClickReloadTemplates() {
      await this.reloadQueries();
    },
    setEvents() {
      const { minWidth } = this;
      const { maxWidth } = this;
      const el = this.$refs.drawer.$el;
      const drawerBorder = el.querySelector('.v-navigation-drawer__border');
      const direction = el.classList.contains('v-navigation-drawer--right')
        ? 'right'
        : 'left';
      function resize(e) {
        document.body.style.cursor = 'ew-resize';
        let f = direction === 'right'
          ? document.body.scrollWidth - e.clientX
          : e.clientX;
        if (f < minWidth) {
          f = minWidth;
        } else if (f > maxWidth) {
          f = maxWidth;
        }
        el.style.width = `${f}px`;
      }
      drawerBorder.addEventListener(
        'mousedown',
        (e) => {
          e.preventDefault();
          el.style.transition = 'initial';
          document.addEventListener('mousemove', resize, false);
        },
        false,
      );
      document.addEventListener(
        'mouseup',
        () => {
          el.style.transition = '';
          this.width = el.style.width;
          document.body.style.cursor = '';
          document.removeEventListener('mousemove', resize, false);
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
