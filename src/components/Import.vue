<template lang="pug">
div
  h3 Import performance
  p(class='subtitle') Import performance entries from external sources

  div(class='row')
    div(class='col')
      div(class='btn-group' role='group')
        b-dropdown(variant='outline-dark' v-if='timesheet' :text='"Timesheet: " + timesheet.display_label')
          b-dropdown-item(v-for='timesheet in timesheets' @click='selectTimesheet(timesheet)') {{ timesheet.display_label }}

    div(class='col-auto')
      div(class='btn-group mr-2')
          b-button(variant='outline-dark' v-if='invalidPerformances.length > 0' :pressed='showInvalidPerformances' v-on:click='toggleInvalidPerformancesWidget()')
            i(class='fa' :class='{"fa-chevron-circle-up": showInvalidPerformances, "fa-chevron-circle-down": !showInvalidPerformances}') &nbsp;
            strong Invalid entries found:&nbsp;
            | {{ invalidPerformances.length }}
      

      div(class='btn-group')
        button(class='btn btn-outline-primary' v-if='performances && performances.length' @click='executeImport()') Import

  div(v-if='invalidPerformances.length > 0 && showInvalidPerformances')
    hr
    table(class='table table-striped table-bordered')
      thead
        tr
          th Date
          th Duration
          th Description
          th Redmine entry
          th Error
  
      tbody
        tr(v-for='(performance, i) in invalidPerformances')
          td {{ performance.date | moment('DD/MM/YYYY') }}
          td {{ performance.duration | round }} hours
          td
            vue-markdown(class='rendered-markdown' :source='performance.description')
          td {{ performance.redmine_id }}
          td {{ performance.invalid_reason}}

  hr

  table(class='table table-striped table-bordered' v-if='performances && performances.length')
    thead
      tr
        th Date
        th Duration
        th Contract
        th Description
        th Type
        th
          b-form-checkbox(v-model='selectAll' class='my-0') Import?

    tbody
      tr(v-for='(performance, i) in performances')
        td {{ performance.date | moment('DD/MM/YYYY') }}
        td {{ performance.duration | round }} hours
        td {{ contracts[performance.contract].display_label }}
        td
          vue-markdown(class='rendered-markdown' :source='performance.description')
        template(v-if='!performance.id')
          td
            select(class='form-control form-control-sm col' v-model='performance.performance_type')
              option(
                v-for='performance_type in ((contracts[performance.contract].performance_types && contracts[performance.contract].performance_types.length) ? contracts[performance.contract].performance_types : performanceTypes)'
                :value='performance_type.id'
              ) {{ performance_type.display_label }}
          td
            b-form-checkbox(v-model='performancesToImport[i]' class='col') &nbsp;
        template(v-else)
          td(colspan='2') Imported {{ performance.updated ? '(with different values)' : ''}}

  div(class='alert alert-info text-center' v-else) {{ emptyMessage }}
</template>

<script>
import store from '../store';
import moment from 'moment';
import * as types from '../store/mutation-types'
import VueMarkdown from 'vue-markdown'

export default {
  name: 'import',

  mixins: [
  ],

  components: {
    VueMarkdown
  },

  data() {
    return {
      timesheet: null,
      selectAll: false,
      performances: [],
      performancesToImport: [],
      showInvalidPerformances: false,
      invalidPerformances: [],
      emptyMessage: "No importable performances found. 😞"
    }
  },

  created: function() {
    new Promise((resolve, reject) => {
      if (!store.getters.active_contracts) {
        store.dispatch(types.NINETOFIVER_RELOAD_CONTRACTS).then(() => resolve())
      } else {
        resolve()
      }
    })

    new Promise((resolve, reject) => {
      if (!store.getters.contract_users) {
        store.dispatch(types.NINETOFIVER_RELOAD_CONTRACT_USERS).then(() => resolve())
      } else {
        resolve()
      }
    })

    new Promise((resolve, reject) => {
      if (!store.getters.performance_types) {
        store.dispatch(types.NINETOFIVER_RELOAD_PERFORMANCE_TYPES).then(() => resolve())
      } else {
        resolve()
      }
    })

    new Promise((resolve, reject) => {
      if (!store.getters.open_timesheets) {
        store.dispatch(types.NINETOFIVER_RELOAD_TIMESHEETS).then(() => resolve())
      } else {
        resolve()
      }
    }).then(() => {
      if (store.getters.open_timesheets.length) {
        this.timesheet = store.getters.open_timesheets[0]
      }
    })
  },

  computed: {
    timesheets: function() {
      if (store.getters.open_timesheets) {
        return store.getters.open_timesheets.filter(timesheet => {
          return timesheet.status == 'active'
        })
      }
    },

    contracts: function() {
      if (store.getters.active_contracts) {
        let contracts = {}

        store.getters.active_contracts.forEach(contract => {
          contracts[contract.id] = contract
        })

        return contracts
      }
    },

    contractUsers: function() {
      if (store.getters.contract_users) {
        let contractUsers = []

        store.getters.contract_users.forEach(contractUser => {
          if (!contractUsers[contractUser.contract.id]) {
            contractUsers[contractUser.contract.id] = []
          }

          contractUsers[contractUser.contract.id].push(contractUser)
        })

        return contractUsers
      }
    },

    performanceTypes: function() {
      if (store.getters.performance_types) {
        return store.getters.performance_types
      }
    }
  },

  watch: {
    selectAll: function(state) {
      this.performances.forEach((value, idx) => {
        this.performancesToImport[idx] = !value.id ? state : false
      })
    },

    timesheet: function() {
      this.reloadPerformances()
    }
  },

  methods: {
    selectTimesheet: function(timesheet) {
      this.timesheet = timesheet
    },

    reloadPerformances: function() {
      let from = moment([this.timesheet.year, this.timesheet.month - 1])
      let until = moment(from).endOf('month')

      store.dispatch(types.NINETOFIVER_API_REQUEST, {
        path: '/imports/performances/',
        params: {
          from: from.format('YYYY-MM-DD'),
          until: until.format('YYYY-MM-DD')
        }
      }).then((res) => {
        let allPerformances = res.data.results
        let importablePerformances = []
        let unimportablePerformances = []

        allPerformances.forEach(performance => {
          if (performance.valid) {
            performance.performance_type = ((this.contracts[performance.contract].performance_types && this.contracts[performance.contract].performance_types.length) ? this.contracts[performance.contract].performance_types : this.performanceTypes)[0].id
            performance.contract_role = this.contractUsers[performance.contract][0].contract_role.id
            importablePerformances.push(performance)
          } else {
            unimportablePerformances.push(performance)
          }
        })
        this.performances = importablePerformances
        this.invalidPerformances = unimportablePerformances
        this.performancesToImport = Array(this.performances.length).fill(false)
      }).catch((error) => {
        if (error.body && error.body.message && error.status == 400) {
          this.emptyMessage = error.body.message
        }
      })
    },

    executeImport: function() {
      let promises = []

      this.performances.forEach((performance, idx) => {
        if (!this.performancesToImport[idx]) {
          return
        }

        promises.push(store.dispatch(types.NINETOFIVER_API_REQUEST, {
          path: '/performances/',
          method: 'POST',
          body: {
            'contract': performance.contract,
            'description': performance.description,
            'date': performance.date,
            'duration': performance.duration,
            'performance_type': performance.performance_type,
            'contract_role': performance.contract_role,
            'redmine_id': performance.redmine_id,
            'type': 'ActivityPerformance',
          }
        }))
      })

      Promise.all(promises).then(() => {
        this.reloadPerformances()
      })
    },

    toggleInvalidPerformancesWidget: function() {
      this.showInvalidPerformances = !this.showInvalidPerformances
    }
  },

  filters: {
    round: function(value) {
      return Math.round(value * 100) / 100
    }
  }
}
</script>
