<template>
  <div ref="imageContainer" class="image-ui relative">
    <div class="flex justify-center items-center h-full w-full">
      <slot v-if="state.loading" name="loading">
        <div
          class="absolute h-full rounded-lg bg-input-dark w-full animate-pulse"
        ></div>
      </slot>
      <slot v-else-if="state.error" name="error">
        <v-mdi name="mdi-image-outline" class="text-gray-300"></v-mdi>
      </slot>
      <div
        v-else
        :style="{
          backgroundImage: `url(${src})`,
          backgroundSize: contain ? 'contain' : 'cover',
        }"
        v-bind="{ role: alt ? 'img' : null, 'aria-label': alt }"
        class="h-full absolute top-0 left-0 w-full bg-no-repeat bg-center"
      >
        <slot></slot>
      </div>
    </div>
  </div>
</template>
<script>
import { ref, shallowReactive, onMounted } from 'vue';
import { useIntersect } from '@/composable/intersect';

export default {
  props: {
    src: {
      type: String,
      default: '',
    },
    alt: {
      type: String,
      default: '',
    },
    lazy: Boolean,
    contain: Boolean,
  },
  emits: ['error', 'load'],
  setup(props, { emit }) {
    const intersect = useIntersect();

    const imageContainer = ref(null);
    const state = shallowReactive({
      loading: true,
      error: false,
    });

    function handleImageLoad() {
      state.loading = false;
      state.error = false;

      emit('load', true);
    }
    function handleImageError() {
      state.loading = false;
      state.error = true;

      emit('error', true);
    }
    function loadImage() {
      const image = new Image();

      image.onload = () => handleImageLoad(image);
      image.onerror = handleImageError;
      image.src = props.src;
    }

    const observer = new IntersectionObserver((entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          const { target } = entry;
          loadImage();
          observer.unobserve(target);
        }
      });
    });

    onMounted(() => {
      if (props.lazy) {
        intersect.observe(imageContainer.value, loadImage);
      } else {
        loadImage();
      }
    });

    return {
      state,
      imageContainer,
    };
  },
};
</script>
