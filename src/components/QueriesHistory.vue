<template>
  <div class="mt-lg-5">
    <h5>Query History</h5>
    <small>Your last 10 SQL queries</small>
    <b-list-group class="scroll-code">
      <b-list-group-item class="d-flex justify-content-between align-items-center" v-for="query in userQueriesList" :key="query.queryExecutionId">
        <code class="code">{{query.queryString}}</code>
        <input type="hidden" :id="query.queryExecutionId.split('-')[0]" :value="query.queryString">
        <b-button size="sm" variant="outline-info" @click="copyTestingCode(query.queryExecutionId.split('-')[0])">Copy</b-button>
      </b-list-group-item>
    </b-list-group>
  </div>
</template>

<script>
import { Analytics } from '@aws-amplify/analytics';
import eventBus from "@/event";
import {AthenaClient} from "@aws-sdk/client-athena";
import {Auth} from "@aws-amplify/auth";
import queryDao from "@/data/queryDAO";
import awsconfig from "@/aws-exports";
export default {
  name: "QueriesHistory",
  async created() {
    eventBus.$on('refreshCredentials', async (credentials) => {
      this.client = new AthenaClient({credentials: credentials, region: awsconfig.aws_project_region})
    })
    const credentials = await Auth.currentCredentials()
    this.client = new AthenaClient({credentials: credentials, region: awsconfig.aws_project_region})
    this.currentUser = await Auth.currentUserInfo()
    let self = this
    queryDao.sessionQuerySubscription(this.currentUser.username).subscribe( {
      next(sessionQuery) {
        self.userQueriesList.unshift(sessionQuery.value.data.onCreateSessionQuery)
      }
    })
    let sessionQuerysResponse = await queryDao.getSessionQueries(this.currentUser.username)
    this.userQueriesList = sessionQuerysResponse.data.listSessionQuerys.items
  },
  async mounted() {

  },
  data() {
    return {
      userQueriesList: [],
      client: null,
      currentUser: null
    }
  },
  methods: {
    copyTestingCode (id) {
      let testingCodeToCopy = document.querySelector(`#${CSS.escape(id)}`)
      Analytics.record({name: 'copyHistoryQuery', attributes: {historyId: id}})
      testingCodeToCopy.setAttribute('type', 'text')
      testingCodeToCopy.select()
      let copiedQuery = testingCodeToCopy.value
      try {
        let successful = document.execCommand('copy');
        let msg = successful ? 'successful' : 'unsuccessful';
        this.$bvToast.toast(copiedQuery, {
          title: `Query copied ${msg}`,
          toaster: 'b-toaster-top-right',
          solid: true,
        })
      } catch (err) {
        alert('Oops, unable to copy');
      }
      /* unselect the range */
      testingCodeToCopy.setAttribute('type', 'hidden')
      window.getSelection().removeAllRanges()
    },
  },
}
</script>

<style scoped>
.code {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  border: 1px solid #fff;
}
.scroll-code {
  overflow-y: scroll;
  height: 250px;
}
</style>
