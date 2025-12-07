<template>
  <div v-bind:class="dynamicClass">

  </div>
</template>
  
  <script>
  export default {
    
    data() {
      return {
        msg: '神奇的魔法',
        dynamicClass: 'active'
      }
    }
  }
  </script>