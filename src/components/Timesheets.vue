<template lang="pug">
div(class='row')
  div(class='col')
    h3 My timesheets
    p(class='subtitle') Overview of all open timesheets
    div(class='row')
      div(v-for="timesheet in openTimesheets" class="col-xl-6")
        TimesheetWidget(:timesheet='timesheet' v-on:submit='reloadTimesheets')

    div(v-if='closedTimesheets && closedTimesheets.length')
      p(class='subtitle') Recently closed timesheets
      div(class='row')
        div(v-for="timesheet in closedTimesheets" class="col-xl-6")
          TimesheetWidget(:timesheet='timesheet')
</template>

<script>
import store from '../store';
import * as types from '../store/mutation-types';
import TimesheetWidget from './widgets/TimesheetWidget.vue';

export default {
  name: 'timesheets',

  mixins: [
  ],

  components: {
    TimesheetWidget,
  },

  created: function() {
    new Promise((resolve, reject) => {
      if (!store.getters.timesheets) {
        store.dispatch(types.NINETOFIVER_RELOAD_TIMESHEETS).then(() => resolve())
      } else{
        resolve()
      }
    })
  },

  data: function() {
    return {
    }
  },

  computed: {
    openTimesheets: () => store.getters.open_timesheets,
    closedTimesheets: () => store.getters.closed_timesheets ? store.getters.closed_timesheets.slice(-2).reverse() : undefined,
  },

  methods: {
    reloadTimesheets: () => {
      store.dispatch(types.NINETOFIVER_RELOAD_TIMESHEETS)
    },
  }
}
</script>

<style>
</style>
