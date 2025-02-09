<script>
import Pagination from './Pagination.vue'
import Bool from './Bool.vue'
import Choice from './Choice.vue'
import RevisionDiffs from './RevisionDiffs.vue'

export default {
  mounted () {
    this.load_issues()
  },
  data () {
    // Set default to since one month back
    let since = new Date()
    since.setMonth(since.getMonth() - 1)
    since = since.toISOString().substring(0, 10)

    return {
      since,
      publishable: 'true',
      choices: {
        publishable: [
          {
            name: 'All states',
            value: 'all'
          },
          {
            name: 'Publishable',
            value: 'true'
          },
          {
            name: 'Not publishable',
            value: 'false'
          }
        ]
      }
    }
  },
  methods: {
    load_issues (name, evt) {
      // Use directly set value from v-model
      if (name !== undefined) {
        this.$set(this, name, evt ? evt.value : null)
      }

      const payload = { ...this.$route.params, publishable: this.publishable, since: this.since }
      this.$store.dispatch('load_check_issues', payload)
    }
  },
  components: {
    Bool,
    Choice,
    Pagination,
    RevisionDiffs
  },
  computed: {
    issues () {
      if (!this.$store.state.check_issues) {
        return []
      }
      return this.$store.state.check_issues.results
    }
  }
}
</script>

<template>
  <div>
    <h1 class="title">Check {{ $route.params.analyzer }} / {{ $route.params.check }}</h1>
    <h2 class="subtitle">On repository {{ $route.params.repository }}</h2>
    <Pagination :api_data="$store.state.check_issues" name="issues" store_method="load_check_issues"></Pagination>

    <div class="field">
      <label class="label">Issues since:</label>
      <div class="control">
        <input class="input" type="date" v-model="since" v-on:change="load_issues()" />
      </div>
    </div>

    <table class="table is-fullwidth" v-if="issues">
      <thead>
        <tr>
          <th>Revisions (diffs)</th>
          <th>Path</th>
          <th>Line</th>
          <td><Choice :choices="choices.publishable" name="publishable" v-on:new-choice="load_issues('publishable', $event)"/></td>
          <th>Message</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="issue in issues">
          <td>
            <RevisionDiffs :diffs=issue.diffs />
          </td>
          <td class="mono">{{ issue.path }}</td>
          <td>{{ issue.line }}</td>
          <td>
            <p>
              <span v-if="issue.level == 'error'" class="tag is-danger">Error</span>
              <span v-else-if="issue.level == 'warning'" class="tag is-warning">Warning</span>
              <span v-else class="tag is-dark">{{ issue.level }}</span>
            </p>
            <Bool :value="issue.publishable" name="Publishable" />
            <Bool :value="issue.in_patch" name="In Patch" />
            <Bool :value="issue.new_for_revision" name="New for revision" />
          </td>
          <td>
            <pre>{{ issue.message }}</pre>
          </td>
        </tr>
      </tbody>
    </table>
    <div class="notification is-info" v-else>Loading check issues...</div>
    <div class="notification is-info" v-if="issues && issues.length == 0">No issues found with these filters</div>
  </div>
</template>

<style>
.mono{
  font-family: monospace;
}
.is-nowrap{
  white-space: nowrap;
}
</style>
