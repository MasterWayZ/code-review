<script>
import mixins from './mixins.js'
import _ from 'lodash'
import Choice from './Choice.vue'

export default {
  mounted () {
    this.load_tasks('production')
  },
  methods: {
    load_tasks (channel) {
      this.$set(this, 'channel', channel)
      this.$store.commit('reset')
      this.$store.dispatch('load_index', { channel })
    }
  },
  components: {
    Choice
  },
  data: function () {
    return {
      channel: 'production',
      filters: {
        state: null,
        issues: null,
        revision: null,
        repository: null
      },
      choices: {
        issues: [
          {
            name: 'No issues',
            func: t => t.data.issues === undefined || t.data.issues === 0
          },
          {
            name: 'Has issues',
            func: t => t.data.issues && t.data.issues > 0
          },
          {
            name: 'Publishable issues',
            func: t => t.data.issues_publishable && t.data.issues_publishable > 0
          }
        ]
      }
    }
  },
  mixins: [
    mixins.date
  ],
  computed: {
    tasks () {
      let tasks = this.$store.state.tasks

      // Filter by repository
      if (this.filters.repository !== null) {
        tasks = _.filter(tasks, t => t.data.repository === this.filters.repository)
      }

      // Filter by states
      if (this.filters.state !== null) {
        tasks = _.filter(tasks, t => t.data.state === this.filters.state.key)
      }

      // Filter by issues
      if (this.filters.issues !== null) {
        tasks = _.filter(tasks, this.filters.issues.func)
      }

      // Filter by revision
      if (this.filters.revision !== null) {
        tasks = _.filter(tasks, t => {
          const payload = t.data.title + t.data.bugzilla_id + t.data.phid + t.data.diff_phid + t.data.id + t.data.diff_id
          return payload.toLowerCase().indexOf(this.filters.revision.toLowerCase()) !== -1
        })
      }

      // Sort by indexation date
      return tasks.sort((x, y) => {
        return new Date(y.data.indexed) - new Date(x.data.indexed)
      })
    },
    tasks_total () {
      return this.$store.state.tasks ? this.$store.state.tasks.length : 0
    },
    states () {
      const currentTasks = this.$store.state.tasks
      const states = currentTasks.reduce((states, task) => {
        if (states[task.data.state] === undefined) {
          states[task.data.state] = 0
        }
        states[task.data.state] += 1
        return states
      }, {})

      // Order states by their nb, and calc percents
      return Object.keys(states).map(state => {
        const nb = states[state]
        return {
          key: state,
          name: state.startsWith('error.') ? 'error: ' + state.substring(6) : state,
          nb,
          percent: currentTasks && currentTasks.length > 0 ? Math.round(nb * 100 / currentTasks.length) : 0
        }
      }).sort((x, y) => { return y.nb - x.nb })
    },
    repositories () {
      return []
    }
  }
}
</script>

<template>
  <section>
    <div class="buttons has-addons is-pulled-right">
      <button v-on:click="load_tasks('production')" class="button is-small" :class="{'is-primary': channel == 'production'}">Production tasks</button>
      <button v-on:click="load_tasks('testing')" class="button is-small" :class="{'is-primary': channel == 'testing'}">Testing tasks</button>
    </div>

    <div class="states" >
      <div class="state columns" v-for="state in states">
        <div class="column is-one-third">
          <progress class="progress" :class="{'is-danger': state.key.startsWith('error') || state.key === 'killed', 'is-success': state.key == 'done', 'is-info': state.key != 'done' && !state.key.startsWith('error')}" :value="state.percent" max="100">{{ state.percent }}%</progress>
        </div>
        <div class="column is-one-third">
          <strong>{{ state.name }}</strong> - <span class="has-text-grey-light">{{ state.nb }}/{{ tasks_total }} tasks or {{ state.percent }}%</span>
        </div>
      </div>
    </div>

    <table class="table is-fullwidth">
      <thead>
        <tr>
          <td>#</td>
          <td>
            <input class="input" type="text" v-model="filters.revision" placeholder="Filter using phabricator, bugzilla Id or word, ..."/>
          </td>
          <td>
            <Choice :choices="repositories" name="repo" v-on:new-choice="filters.repository = $event"/>
          </td>
          <td>
            <Choice :choices="states" name="state" v-on:new-choice="filters.state = $event"/>
          </td>
          <td>
            <Choice :choices="choices.issues" name="issue" v-on:new-choice="filters.issues = $event"/>
          </td>
          <td>Indexed</td>
          <td>Actions</td>
        </tr>
      </thead>

      <tbody>
        <tr v-for="task in tasks">
          <td>
            <a class="mono" :href="'https://firefox-ci-tc.services.mozilla.com/tasks/' + task.taskId" target="_blank">{{ task.taskId }}</a>
          </td>

          <td>
            <p v-if="task.data.title">{{ task.data.title }}</p>
            <p class="has-text-danger" v-else>No title</p>
            <p>
              <small class="mono has-text-grey-light">{{ task.data.diff_phid}}</small> - diff {{ task.data.diff_id || 'unknown'     }}
            </p>
            <p>
              <small class="mono has-text-grey-light">{{ task.data.phid}}</small> - <router-link :to="{ name: 'revision', params: { revisionId: task.data.id }}">rev {{ task.data.id }}</router-link>
            </p>
          </td>

          <td>
            <span class="tag is-primary" v-if="task.data.repository == 'https://hg.mozilla.org/mozilla-central'">Mozilla Central</span>
            <span class="tag is-info" v-else-if="task.data.repository == 'https://hg.mozilla.org/projects/nss'">NSS</span>
            <span class="tag is-dark" v-else>{{ task.data.repository || 'Unknown'}}</span>
          </td>

          <td>
            <span class="tag is-light" v-if="task.data.state == 'started'">Started</span>
            <span class="tag is-info" v-else-if="task.data.state == 'cloned'">Cloned</span>
            <span class="tag is-info" v-else-if="task.data.state == 'analyzing'">Analyzing</span>
            <span class="tag is-primary" v-else-if="task.data.state == 'analyzed'">Analyzed</span>
            <span class="tag is-danger" v-else-if="task.data.state == 'killed'">
              Killed for timeout
            </span>
            <span class="tag is-danger" v-else-if="task.data.state == 'error'" :title="task.data.error_message">
              Error: {{ task.data.error_code || 'unknown' }}
            </span>
            <span class="tag is-success" v-else-if="task.data.state == 'done'">Done</span>
            <span class="tag is-black" v-else>Unknown</span>
          </td>

          <td :class="{'has-text-success': task.data.issues_publishable > 0}">

            <span v-if="task.data.issues_publishable > 0">{{ task.data.issues_publishable }}</span>
            <span v-else-if="task.data.issues_publishable == 0">{{ task.data.issues_publishable }}</span>
            <span v-else>-</span>
            / {{ task.data.issues }}
          </td>

          <td>
            <span :title="task.data.indexed">{{ task.data.indexed|since }} ago</span>
          </td>
          <td>
            <a class="button is-link" :href="task.data.url" target="_blank">Phabricator</a>
            <a v-if="task.data.bugzilla_id" class="button is-dark" :href="'https://bugzil.la/' + task.data.bugzilla_id" target="_blank">Bugzilla</a>
            <a class="button is-primary" :href="'https://firefox-ci-tc.services.mozilla.com/tasks/' + task.data.try_group_id" target="_blank">Try Tasks</a>
            <router-link v-if="task.data.issues > 0" :to="{ name: 'diff', params: { diffId : task.data.diff_id }}" class="button is-primary">Issues</router-link>
          </td>
        </tr>
      </tbody>
    </table>
  </section>
</template>

<style>
.mono{
  font-family: monospace;
}

div.states {
  margin-top: 1rem;
  margin-bottom: 2rem;
}

div.states div.column {
  padding: 0.2rem;
}

div.states div.column progress {
  margin-top: 0.3rem;
}

div.table input.input {
  display: inline-block;
}
</style>
