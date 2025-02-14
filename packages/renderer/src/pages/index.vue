<template>
  <div
    class="container flex flex-col h-full z-40"
    @drop="handleDrop"
    @dragover.prevent
    @dragenter.prevent
  >
    <home-nav
      v-model:viewType="state.viewType"
      v-model:search="state.search"
    ></home-nav>
    <p v-if="allProjects.length === 0" class="my-12 text-gray-200 text-center">
      You have no project
    </p>
    <div v-else class="flex-1 overflow-auto scroll pb-5 px-5">
      <div v-if="projects.starred.length !== 0" class="mb-12">
        <p class="text-gray-300 mb-3">Starred</p>
        <home-projects
          :list-view="state.viewType === 'list'"
          :projects="projects.starred"
        ></home-projects>
      </div>
      <div v-if="projects.recent.length !== 0" class="mb-4">
        <p class="text-gray-300 mb-3">Recent</p>
        <home-projects
          :list-view="state.viewType === 'list'"
          :projects="projects.recent"
        ></home-projects>
      </div>
    </div>
  </div>
</template>
<route>
{
  name: 'home',
}
</route>
<script>
import {
  computed,
  shallowRef,
  shallowReactive,
  onMounted,
  onUnmounted,
  watch,
} from 'vue';
import { useStore } from 'vuex';
import { useRouter } from 'vue-router';
import { nanoid } from 'nanoid';
import { debounce } from '@/utils/helper';
import KeyboardNavigation from '@/utils/keyboardNavigation';
import Project from '@/models/project';
import HomeNav from '@/components/home/HomeNav.vue';
import HomeProjects from '@/components/home/HomeProjects.vue';

export default {
  components: { HomeNav, HomeProjects },
  setup() {
    const store = useStore();
    const router = useRouter();

    const { ipcRenderer, existsSync, path } = window.electron;

    const keyboardNavigation = shallowRef(null);
    const state = shallowReactive({
      search: '',
      viewType: 'list',
    });

    const allProjects = computed(() => Project.all());
    const projects = computed(() => {
      const projects = Project.query()
        .where(
          ({ name, isMonorepo }) =>
            !isMonorepo &&
            name.toLocaleLowerCase().includes(state.search.toLocaleLowerCase())
        )
        .orderBy('createdAt', 'desc')
        .get()
        .map((project) => ({
          ...project,
          isPathExist: existsSync(project.path),
        }));
      const filteredProjects = filterProjects(projects);

      return filteredProjects;
    });

    function filterProjects(projects) {
      const result = {
        recent: [],
        starred: [],
      };

      projects.forEach((project) => {
        result[project.starred ? 'starred' : 'recent'].push(project);
      });

      return result;
    }
    async function handleDrop({ dataTransfer }) {
      const projectPromises = Object.values(dataTransfer.items).map(
        async (item) => {
          try {
            const projects = [];
            const { name, path: projectDir } = item.getAsFile();
            const isPkgJsonExists = existsSync(
              path.join(projectDir, 'package.json')
            );
            const projectExists = Project.query()
              .where('path', projectDir)
              .exists();

            if (!isPkgJsonExists || projectExists) return;

            const id = nanoid();
            const repository = await ipcRenderer.callMain(
              'get:repository',
              projectDir
            );
            projects.push({
              id,
              name,
              repository,
              path: projectDir,
              createdAt: Date.now(),
            });

            const workspaces = await ipcRenderer.callMain(
              'get:workspaces',
              projectDir
            );

            workspaces.forEach((workspace) => {
              projects.push({
                ...workspace,
                rootId: id,
                isMonorepo: true,
                repository: projects[0]?.repository ?? '',
              });
            });

            return projects;
          } catch (error) {
            console.error(error);
          }
        }
      );
      Promise.allSettled(projectPromises).then((projects) => {
        projects.forEach(({ status, value }) => {
          if (status === 'fulfilled' && value) {
            Project.insert({ data: value });
          }
        });

        store.dispatch('saveToStorage', 'projects');
      });
    }
    function deleteProject(name, id) {
      const confirm = window.confirm(`Are you sure want to delete "${name}"?`);

      if (confirm) Project.delete(id);
    }

    const navigationBreakpoints = {
      default: 2,
      '(min-width: 1024px)': 3,
      '(min-width: 1280px)': 4,
    };

    watch(
      projects,
      debounce(() => {
        keyboardNavigation.value?.refresh();
      }, 200)
    );
    watch(
      () => state.viewType,
      debounce((value) => {
        keyboardNavigation.value?.setOptions({
          breakpoints:
            value === 'grid' ? navigationBreakpoints : { default: 1 },
        });
      }, 200)
    );

    onMounted(() => {
      state.viewType = localStorage.getItem('view-type') || 'list';

      keyboardNavigation.value = new KeyboardNavigation({
        itemSelector: '.project-card',
      });

      keyboardNavigation.value.on('keydown', ({ key }, activeItem) => {
        if (!activeItem) return;

        const { projectId, projectName } = activeItem.dataset;

        switch (key) {
          case 'Enter':
            router.push(`/project/${projectId}`);
            break;
          case 'Backspace':
          case 'Delete':
            deleteProject(projectName, projectId);
            break;
          default:
        }
      });

      setTimeout(() => {
        keyboardNavigation.value.refresh();
      }, 250);
    });
    onUnmounted(() => {
      keyboardNavigation.value.destroy();
    });

    return {
      state,
      projects,
      handleDrop,
      allProjects,
    };
  },
};
</script>
