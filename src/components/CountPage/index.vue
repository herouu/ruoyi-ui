<template>
  <div>
    <button v-on:click="count++">You clicked me {{ count }} times.</button>
  </div>
</template>

<script>
export default {
  name: "CountPage",
  props: {
    count: Number
  },
  data: function () {
    return {
      count: 0
    }
  },
}
</script>

<style scoped>

</style>
