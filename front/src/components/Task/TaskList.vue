<template>
  <div>
    <div class="row q-mb-md q-col-gutter-sm">
      <div class="col col-md-3 col-sm-6 col-xs-11 col-grow">
        <q-input v-model="filter" filled dense placeholder="Search">
          <template #prepend>
            <q-icon name="search" />
          </template>
        </q-input>
      </div>
      <div class="col col-md-3 col-sm-6 col-xs-11 col-grow">
        <q-select
          v-model="tagFilter"
          :options="tags"
          filled
          dense
          options-dense
          use-chips
          multiple
          label="Filter by tag"
          emit-value
        >
          <template #prepend>
            <q-icon name="label" />
          </template>
          <template v-if="tagFilter.length" #append>
            <q-icon
              name="cancel"
              class="cursor-pointer"
              @click.stop="tagFilter = []"
            />
          </template>
        </q-select>
      </div>

      <div class="col col-auto">
        <q-checkbox v-model="hideSolved" label="Hide solved" class="q-mr-sm" />
        <q-checkbox v-model="myTasks" label="Show my tasks" />
      </div>

      <q-space />

      <div class="col col-auto q-ml-auto">
        <q-select
          v-model="displayMode"
          filled
          dense
          options-dense
          label="Display"
          style="width: 130px"
          :options="displayOptions"
        />
      </div>
    </div>

    <template v-if="sortedTasks.length">
      <task-cards
        v-if="useCard"
        :ctf="ctf"
        :tasks="sortedTasks"
        :display-mode="displayMode"
      />
      <task-table-dense
        v-else-if="displayMode == 'tabledense'"
        :ctf="ctf"
        :tasks="sortedTasks"
      />
      <task-table v-else :ctf="ctf" :tasks="sortedTasks" />
    </template>

    <div v-else class="text-center col">
      <div class="row q-gutter-md justify-center">
        <q-btn
          icon="add"
          label="Create Task"
          color="positive"
          @click="openCreateTaskDialog(ctf)"
        />
        <q-btn
          icon="flag"
          label="Import Task"
          color="secondary"
          @click="openImportTaskDialog"
        />
      </div>
    </div>

    <q-page-sticky position="top-right" :offset="[18, 8]">
      <q-fab
        class="ctfs-action-btn shadow-2"
        padding="10px"
        color="positive"
        icon="add"
        direction="down"
        vertical-actions-align="right"
        push
      >
        <q-fab-action
          color="positive"
          push
          icon="add"
          label="Create"
          @click="openCreateTaskDialog(ctf)"
        />
        <q-fab-action
          color="secondary"
          push
          icon="flag"
          label="Import "
          @click="openImportTaskDialog"
        />
        <q-fab-action
          color="accent"
          push
          icon="file_download"
          label="Export "
          @click="openExportTasksDialog"
        />
      </q-fab>
    </q-page-sticky>
  </div>
</template>

<script lang="ts">
import { Ctf, Task } from 'src/ctfnote/models';
import ctfnote from 'src/ctfnote';
import { useStoredSettings } from 'src/extensions/storedSettings';
import { defineComponent, ref, provide } from 'vue';
import TaskEditDialogVue from '../Dialogs/TaskEditDialog.vue';
import TaskImportDialogVue from '../Dialogs/TaskImportDialog.vue';
import TaskExportDialogVue from '../Dialogs/TaskExportDialog.vue';
import TaskCards from './TaskCards.vue';
import TaskTable from './TaskTable.vue';
import TaskTableDense from './TaskTableDense.vue';
import keys from '../../injectionKeys';
import { tagsSortFn } from 'src/ctfnote/tags';
import { Platform } from 'quasar';

const displayOptions = [
  'classic',
  'dense',
  'ultradense',
  'table',
  'tabledense',
] as const;

export type DisplayMode = (typeof displayOptions)[number];

export default defineComponent({
  components: { TaskCards, TaskTable, TaskTableDense },
  props: {
    ctf: { type: Object as () => Ctf, required: true },
  },
  setup() {
    const { makePersistant } = useStoredSettings();

    const me = ctfnote.me.injectMe();

    const filter = ref('');
    const tagFilter = ref<string[]>([]);
    const hideSolved = makePersistant('task-hide-solved', ref(false));
    const myTasks = makePersistant('task-my-tasks', ref(false));

    provide(keys.isTaskVisible, (task: Task) => {
      const needle = filter.value.toLowerCase();
      // Hide solved task if hideSolved == true
      if (hideSolved.value && task.solved) return false;

      if (
        myTasks.value &&
        me.value &&
        task.workOnTasks.filter(
          (w) => w.profileId == me.value?.profile.id && w.active,
        ).length == 0
      )
        return false;

      // Hide task if there is a filter and category not in filter
      const tFilter = tagFilter.value;
      if (
        tFilter.length &&
        !tFilter.some((t) => task.assignedTags.some((tt) => tt.tag == t))
      ) {
        return false;
      }

      // Filter using needle on title and description
      const fields = ['title', 'description'] as const;

      const checkField = (f: (typeof fields)[number]): boolean =>
        task[f]?.toLowerCase().includes(needle) ?? false;

      return fields.some((name) => checkField(name));
    });

    provide(keys.filterTag, (category: string) => {
      if (!tagFilter.value.includes(category)) {
        tagFilter.value.push(category);
      }
    });

    const chooseDisplayMode = () => (Platform.is.mobile ? 'classic' : 'table');

    return {
      displayMode: makePersistant(
        'task-display-mode',
        ref(chooseDisplayMode()),
      ),
      hideSolved,
      myTasks,
      filter,
      tagFilter,
      displayOptions,
      me,
    };
  },
  computed: {
    tasks() {
      return this.ctf.tasks;
    },
    useCard() {
      return this.displayMode != 'table' && this.displayMode != 'tabledense';
    },
    tags() {
      return [
        ...new Set(
          this.tasks.flatMap((t) => t.assignedTags.map((tt) => tt.tag)),
        ),
      ];
    },
    sortedTasks() {
      const tasks = this.tasks;
      return tasks
        .slice()
        .sort(tagsSortFn)
        .sort((a) => {
          return a.solved ? 1 : -1;
        });
    },
  },
  methods: {
    openCreateTaskDialog(ctf: Ctf) {
      this.$q.dialog({
        component: TaskEditDialogVue,
        componentProps: {
          ctfId: ctf.id,
        },
      });
    },
    openImportTaskDialog() {
      this.$q.dialog({
        component: TaskImportDialogVue,
        componentProps: {
          ctf: this.ctf,
        },
      });
    },
    openExportTasksDialog() {
      this.$q.dialog({
        component: TaskExportDialogVue,
        componentProps: {
          ctf: this.ctf,
        },
      });
    },
  },
});
</script>
