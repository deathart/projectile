<template>
  <div class="issue-card">
    <div class="flex items-center mb-6">
      <ui-select v-model="state.activeFilter" placeholder="Filter by">
        <option v-for="filter in filters" :key="filter" :value="filter">
          {{ filter }}
        </option>
      </ui-select>
      <ui-button v-tooltip="'Reload'" icon class="ml-2" @click="reloadIssues">
        <v-mdi name="mdi-reload"></v-mdi>
      </ui-button>
      <div class="flex-grow"></div>
      <ui-input
        v-model="state.search"
        prepend-icon="mdi-magnify"
        placeholder="Search..."
      ></ui-input>
    </div>
    <div v-if="state.status === 'loading'" class="text-center my-4">
      <ui-spinner size="36"></ui-spinner>
    </div>
    <div
      v-else
      class="grid grid-cols-2 gap-2 max-h-96 scroll overflow-auto mb-8"
    >
      <div
        v-for="issue in issues"
        :key="issue.id"
        :class="{
          'bg-opacity-5 bg-gray-100': state.selectedIssues.includes(issue.id),
        }"
        class="
          p-4
          rounded-lg
          border
          hover:bg-gray-100 hover:bg-opacity-5
          cursor-pointer
          transition
        "
        @click="toggleSelectIssue(issue.id)"
      >
        <div class="flex items-center text-gray-300 mb-2">
          <v-mdi
            :name="
              issue.pull_request
                ? 'mdi-source-pull'
                : 'mdi-record-circle-outline'
            "
            :class="[
              issue.state === 'closed' ? 'text-red-400' : 'text-green-500',
            ]"
            size="20"
            class="mr-1"
          >
          </v-mdi>
          <span class="text-sm text-overflow flex-1 pr-2">
            {{ issue.user.login }}#{{ issue.number }}
          </span>
          <v-mdi
            v-if="state.selectedIssues.includes(issue.id)"
            name="mdi-check"
            class="text-primary"
            size="20"
          ></v-mdi>
        </div>
        <p class="leading-tight line-clamp text-gray-100">{{ issue.title }}</p>
      </div>
    </div>
    <ui-button
      class="w-36"
      variant="primary"
      :disabled="state.selectedIssues.length === 0"
      @click="addCard"
    >
      Add ({{ state.selectedIssues.length }})
    </ui-button>
  </div>
</template>
<script>
import { onMounted, reactive, computed } from 'vue';
import { useRoute } from 'vue-router';
import Card from '@/models/card';

export default {
  props: {
    repository: {
      type: String,
      default: '',
    },
    boardId: {
      type: String,
      default: '',
    },
  },
  emits: ['save'],
  setup(props, { emit }) {
    const route = useRoute();

    const filters = ['all', 'open', 'closed'];
    const state = reactive({
      issues: [],
      search: '',
      status: 'idle',
      selectedIssues: [],
      activeFilter: 'all',
    });

    const issues = computed(() =>
      state.issues.filter(
        (issue) =>
          (state.activeFilter === 'all'
            ? true
            : issue.state === state.activeFilter) &&
          issue.title
            .toLocaleLowerCase()
            .includes(state.search.toLocaleLowerCase())
      )
    );

    function toggleSelectIssue(id) {
      const index = state.selectedIssues.indexOf(id);

      if (index === -1) state.selectedIssues.push(id);
      else state.selectedIssues.splice(index, 1);
    }
    function addCard() {
      if (!props.boardId || state.selectedIssues.length === 0) return;

      let maxOrder =
        (Card.query().where('boardId', props.boardId).max('order') || 0) - 1;

      const data = state.selectedIssues.map((id) => ({
        order: (maxOrder += 1),
        type: 'issue',
        boardId: props.boardId,
        data: issues.value.find((issue) => issue.id === id) || {},
      }));

      Card.insert({ data }).then(() => {
        emit('save');
      });
    }
    async function fetchIssues() {
      try {
        state.status = 'loading';

        const apiURL = props.repository.replace(
          'github.com/',
          'api.github.com/repos/'
        );

        const response = await fetch(apiURL + '/issues?state=all&per_page=100');
        const limit = +response.headers.get('x-ratelimit-remaining');

        if (limit === 0 || !response.ok) return [];

        const data = await response.json();

        state.status = 'idle';

        return data;
      } catch (error) {
        console.error(error);
        state.status = 'error';
        return [];
      }
    }
    async function reloadIssues() {
      state.loading = true;

      try {
        state.issues = await fetchIssues();
        state.loading = false;

        localStorage.setItem(
          `api_${route.params.id}`,
          JSON.stringify(state.issues)
        );
      } catch (error) {
        state.loading = false;
      }
    }

    onMounted(async () => {
      const cacheKey = `api_${route.params.id}`;

      try {
        const lsData = localStorage.getItem(cacheKey);
        let data = lsData ? JSON.parse(lsData) : null;

        if (!data) {
          data = await fetchIssues();
        }

        state.issues = data;
        state.loading = false;

        localStorage.setItem(cacheKey, JSON.stringify(data));
      } catch (error) {
        state.loading = false;
      }
    });

    return {
      state,
      issues,
      filters,
      addCard,
      reloadIssues,
      toggleSelectIssue,
    };
  },
};
</script>
