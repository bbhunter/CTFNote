<template>
  <q-table
    :rows="tasks"
    :columns="columns"
    :pagination="pagination"
    hide-pagination
    flat
    dense
  >
    <template #body="props">
      <q-tr
        v-show="isTaskVisible(props.row)"
        style="height: 45px"
        :props="props"
      >
        <task-menu :task="props.row" :context-menu="true" />

        <q-td
          key="tags"
          style="padding-left: 12px"
          :props="props"
          class="no-click"
        >
          <task-tags-list :tags="props.row.assignedTags" />
        </q-td>

        <q-td key="solved" :props="props" auto-width>
          <q-badge v-if="props.row.solved" color="positive">
            <q-icon name="flag" />
          </q-badge>
        </q-td>

        <q-td key="title" :props="props" align="left">
          <ctf-note-link name="task" :ctf="ctf" :task="props.row">
            {{ props.row.title }}
          </ctf-note-link>
        </q-td>

        <q-td key="description" :props="props">
          <div v-if="props.row.solved" class="blur q-py-xs">
            {{ props.row.flag }}
          </div>
          <div v-else class="q-py-xs">{{ props.row.description || '…' }}</div>
        </q-td>

        <q-td key="players" :props="props">
          <task-player-list :task="props.row" />
        </q-td>

        <q-td key="actions" :props="props" class="no-click" auto-width>
          <task-btn-group
            dense
            group
            show-open-task-btn
            :ctf="ctf"
            :task="props.row"
          />
        </q-td>
      </q-tr>
    </template>
  </q-table>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { Ctf, Task } from 'src/ctfnote/models';
import ctfnote from 'src/ctfnote';
import TaskBtnGroup from './TaskBtnGroup.vue';
import TaskTagsList from './TaskTagsList.vue';
import TaskPlayerList from './TaskPlayerList.vue';
import TaskMenu from './TaskMenu.vue';
import { injectStrict } from 'src/ctfnote/utils';
import CtfNoteLink from '../Utils/CtfNoteLink.vue';
import keys from '../../injectionKeys';

function col(name: string, opt = {}) {
  return {
    name,
    field: name,
    label: name,
    align: 'left',
    ...opt,
    headerStyle: 'text-transform: capitalize;',
  };
}

export default defineComponent({
  components: {
    TaskBtnGroup,
    TaskTagsList,
    TaskPlayerList,
    TaskMenu,
    CtfNoteLink,
  },
  props: {
    ctf: { type: Object as () => Ctf, required: true },
    tasks: { type: Array as () => Task[], required: true },
  },
  setup() {
    const columns = [
      col('tags', { sortable: true }),
      col('solved', { sortable: false, label: '' }),
      col('title', { sortable: true }),
      col('description', { label: 'description / flag' }),
      col('players', { align: 'center' }),
      col('actions', {
        label: '',
        align: 'right',
      }),
    ];
    const pagination = {
      rowsPerPage: 0,
    };
    const me = ctfnote.me.injectMe();
    const team = ctfnote.profiles.injectTeam();
    return {
      columns,
      pagination,
      me,
      team,
      isTaskVisible: injectStrict(keys.isTaskVisible),
    };
  },
});
</script>

<style scoped>
.q-table td,
.q-table th {
  white-space: normal;
}
</style>
