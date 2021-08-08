<template>
  <!-- Form core custome for vuetify by Mune-dev - please copy full content when use order source -->
  <v-form ref="formData">
    <v-container grid-list-md pt-4>
      <v-layout :class="cls" row wrap>
        <v-flex
          v-for="key in Object.keys(formData)"
          :key="key"
          :class="formData[key].class"
        >
          <component
            :is="
              require(`~/components/${
                formData[key].pathComp
                  ? `comps/${formData[key].pathComp}`
                  : 'comps'
              }/${formData[key].comp}`).default
            "
            v-if="formData[key].comp"
            :opt="formData[key].opt"
            :item="formData[key]"
            :more-data="moreData"
            @outdata="$emit('outdata', $event)"
          />
        </v-flex>
      </v-layout>
    </v-container>
  </v-form>
</template>

<script>
export default {
  props: {
    cls: {
      type: String,
      default: ''
    },
    formData: {
      type: Object,
      default() {
        return {}
      }
    },
    moreData: {
      type: Object,
      default() {
        return {}
      }
    }
  }
}
</script>
