<template>
  <component
    :is="tag"
    role="button"
    class="ui-button relative transition"
    :class="[
      color ? color : variants[variant],
      icon ? 'p-2' : 'py-2 px-4',
      circle ? 'rounded-full' : 'rounded-lg',
      { 'opacity-70': disabled, 'pointer-events-none': loading || disabled },
    ]"
    v-bind="{ disabled: loading || disabled, ...$attrs }"
  >
    <span
      class="flex justify-center h-full items-center"
      :class="{ 'opacity-25': loading }"
    >
      <slot></slot>
    </span>
    <div v-if="loading" class="button-loading">
      <ui-spinner
        :color="variant === 'default' ? 'text-primary' : 'text-white'"
      ></ui-spinner>
    </div>
  </component>
</template>
<script>
import UiSpinner from './Spinner.vue';

export default {
  components: { UiSpinner },
  props: {
    icon: Boolean,
    disabled: Boolean,
    loading: Boolean,
    circle: Boolean,
    color: {
      type: String,
      default: '',
    },
    tag: {
      type: String,
      default: 'button',
    },
    variant: {
      type: String,
      default: 'default',
    },
  },
  setup() {
    const variants = {
      default: 'bg-gray-100 bg-opacity-5 hover:bg-opacity-10',
      primary: 'bg-primary text-white hover:bg-blue-600',
      danger: 'bg-red-500 text-white hover:bg-red-600',
    };

    return {
      variants,
    };
  },
};
</script>
<style>
.button-loading {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
</style>
