<template>
  <div id="app">
    <section class="section container">
      <div class="row">
        <div class="col-md-3">
          <multiselect v-model="reposValue" :options="reposOptions" :multiple="true" :close-on-select="false" :clear-on-select="false" :hide-selected="true" :preserve-search="true" placeholder="All Repos" label="full_name" track-by="full_name" @input="updateIssues" selectLabel="">
            <template slot="tag" slot-scope="props">
              <span class="custom__tag">
                <span>{{ props.option.full_name }}</span>
                <span class="custom__remove" @click="props.remove(props.option)">❌ </span>
              </span>
            </template>
          </multiselect>
        </div>
        <div class="col-md-3">
          <multiselect v-model="assigneesValue" :options="assigneesOptions" :multiple="true" :close-on-select="false" :clear-on-select="false" :hide-selected="true" :preserve-search="true" placeholder="All Assignees" label="login" track-by="login" @input="updateIssues" selectLabel="">
            <template slot="tag" slot-scope="props">
              <span class="custom__tag">
                <span>{{ props.option.login }}</span>
                <span class="custom__remove" @click="props.remove(props.option)">❌ </span>
              </span>
            </template>
          </multiselect>
        </div>
        <div class="col-md-3">
          <multiselect v-model="labelsValue" :options="labelsOptions" :multiple="true" :close-on-select="false" :clear-on-select="false" :hide-selected="true" :preserve-search="true" label="name" track-by="name" placeholder="All Labels" @input="updateIssues" selectLabel="">
            <template slot="tag" slot-scope="props">
              <span class="custom__tag">
                <span>{{ props.option.name}}</span>
                <span class="custom__remove" @click="props.remove(props.option)">❌ </span>
              </span>
            </template>
          </multiselect>
        </div>
        <div class="col-md-3">
          <multiselect v-model="milestonesValue" :options="milestonesOptions" :multiple="true" :close-on-select="false" :clear-on-select="false" :hide-selected="true" :preserve-search="true" label="title" track-by="title" placeholder="All Milestones" @input="updateIssues" selectLabel="">
            <template slot="tag" slot-scope="props">
              <span class="custom__tag">
                <span>{{ props.option.title }}</span>
                <span class="custom__remove" @click="props.remove(props.option)">❌ </span>
              </span>
            </template>
          </multiselect>
        </div>
      </div>
    </section>
    <br/>
    <Kanban :stages="stages" :blocks="issues" @update-block="updateIssueStatus">
      <div v-for="issue in issues" :slot="issue.id" :key="issue.id">
        <div v-if="issue.assignee">
          <a href="javascript:void(0)">
            <span class="badge badge-secondary" v-on:click="showModal(issue.id)">#{{issue.id}}</span>
          </a>
          <img class="avatar img-rounded" :src="issue.assignee.avatar_url" />
        </div>
        <div>
          <strong>{{ issue.title }}</strong>
        </div>
        <div class="issue-tag" v-if="issue.assignee">
          {{ issue.milestone.title }}
        </div>
        <div v-for="label in issue.labels" :key="label.title" class="issue-tag" :style="{ backgroundColor: '#'+label.color}" v-if="!stages.includes(label.name)">
          <span>{{label.name}}</span>
        </div><br/>
        <div class="text-right">
          <a class="badge badge-secondary" :href="issue.url.replace('/api/v1/repos', '').replace(/.$/, issue.number)" target="_blank">
            <span class="glyphicon glyphicon-link"></span>
          </a>
        </div>

      </div>
    </Kanban>
  </div>
</template>

<script>
import _ from 'lodash';
import Multiselect from 'vue-multiselect';
import http from './http-common';
import Kanban from './components/Kanban';

export default {
  name: 'app',
  components: {
    Kanban,
    Multiselect,
  },
  data() {
    return {
      stages: ['backlog', 'in-progress', 'question', 'validation', 'done'],
      issues: [], // Show issues on board
      allIssues: [], // All issues user can access
      reposValue: [],
      reposOptions: [],
      milestonesValue: [],
      milestonesOptions: [],
      labelsValue: [],
      labelsOptions: [],
      assigneesValue: [],
      assigneesOptions: [],
    };
  },
  created() {
    this.token = 'f48b0e812b76f52e36cc04178308ad8c763f40f2';
    this.getKanbanData();
  },
  methods: {
    updateIssues() {
      // update list view of issues based on filters selected
      this.issues = this.allIssues;

      // update issues based on the repo filter if provided
      if (this.reposValue.length > 0) {
        this.issues = _.filter(this.issues, issue => _.some(this.reposValue, issue.repo));
        this.updateUrl({ repos: _.map(this.reposValue, 'full_name').join() });
      }

      // update issues based on the assignee filter if provided
      if (this.assigneesValue.length > 0) {
        this.issues = _.filter(this.issues, (issue) => {
          const assigned = !_.isEmpty(issue.assignee);
          return assigned && _.some(this.assigneesValue, issue.assignee);
        });

        // add assignees to url
        this.updateUrl({ assignees: _.map(this.assigneesValue, 'username').join() });
      }

      // update issues based on the label filter if provided
      if (this.labelsValue.length > 0) {
        this.issues = _.filter(this.issues, issue =>
          !_.isEmpty(_.intersectionBy(this.labelsValue, issue.labels, 'id')),
        );
        // add labels to url
        this.updateUrl({ labels: _.map(this.labelsValue, 'name').join() });
      }

      // update issues based on the milestones filter if provided
      if (this.milestonesValue.length > 0) {
        this.issues = _.filter(this.issues, issue =>
          !_.isEmpty(issue.milestone) && _.some(this.milestonesValue, issue.milestone),
        );
        // add milestones to url
        this.updateUrl({ milestones: _.map(this.milestonesValue, 'title').join() });
      }
    },
    updateIssueStatus(id, status) {
      const issue = _.find(this.issues, { id: Number(id) });
      const oldStatus = issue.status;
      issue.status = status;
      const url = `/repos/${issue.repo.full_name}/issues/${issue.number}`;
      if (oldStatus !== 'backlog' && oldStatus !== 'done') {
        const oldLabelID = _.find(issue.labels, { name: oldStatus }).id;
        const deleteUrl = `${url}/labels/${oldLabelID}?token=${this.token}`;
        http.request.delete(deleteUrl).then();
      }

      // Close if issue was moved to
      if (status === 'done') {
        const doneUrl = `${url}?token=${this.token}`;
        http.request.patch(doneUrl, { state: 'closed' }).then();
        return;
      }

      // Open if it was in done
      if (oldStatus === 'done') {
        const doneUrl = `${url}?token=${this.token}`;
        http.request.patch(doneUrl, { state: 'open' }).then();
      }

      // Add label of new state if it was not backlog
      if (status !== 'backlog') {
        const label = _.find(this.labelsOptions, { name: status });
        const addLabelUrl = `${url}/labels?token=${this.token}`;
        http.request.post(addLabelUrl, { labels: [label.id] }).then();
      }
    },
    updateUrl(queryObject) {
      this.$router.push({
        query: Object.assign({}, this.$route.query, queryObject),
      });
    },
    showModal(issueId) {
      this.$modal.show(String(issueId));
    },
    getKanbanData() {
      const reposUrl = `/repos/search?token=${this.token}&uid=2`;

      // add assignees to url
      const stagesQuery = this.$route.query.stages;
      if (!_.isEmpty(stagesQuery)) {
        this.stages = _.split([stagesQuery], ',');
      } else {
        this.updateUrl({ stages: _.join(this.stages) });
      }

      http.request.get(reposUrl).then(
        (response) => {
          this.reposOptions = response.data.data;
          _.forEach(this.reposOptions, (repo) => {
            const repoUrl = `/repos/${repo.full_name}`;
            // Get milestones for each repo and add it to milestonesOptions
            const milestonesUrl = `${repoUrl}/milestones?token=${this.token}`;
            http.request.get(milestonesUrl).then(
              (milestonesResponse) => {
                _.forEach(milestonesResponse.data, (milestone) => {
                  if (!_.find(this.milestonesOptions, { title: milestone.title })) {
                    this.milestonesOptions.push(milestone);
                  }
                });
              });

            // Get collaborators for each repo and add it to assigneesOptions
            const collaboratorsUrl = `${repoUrl}/collaborators?token=${this.token}`;
            http.request.get(collaboratorsUrl).then(
              (collaboratorsResponse) => {
                collaboratorsResponse.data.push(repo.owner);
                _.forEach(collaboratorsResponse.data, (collaborator) => {
                  if (!_.find(this.assigneesOptions, { id: collaborator.id })) {
                    this.assigneesOptions.push(collaborator);
                  }
                });
              });

            // Get labels of each repo and add it to labelsOptions
            const labelsUrl = `${repoUrl}/labels?token=${this.token}`;
            http.request.get(labelsUrl).then(
              (labelsResponse) => {
                _.forEach(labelsResponse.data, (label) => {
                  if (!_.find(this.labelsOptions, { title: label.title })) {
                    this.labelsOptions.push(label);
                  }
                });
              });

            // Get issues of each repo and add it to issuesOptions
            const issuesUrl = `${repoUrl}/issues?state=all&token=${this.token}`;
            http.request.get(issuesUrl).then(
              (issuesResponse) => {
                /* eslint-disable no-param-reassign */
                _.forEach(issuesResponse.data, (issue) => {
                  // Set repo and status on all issues
                  issue.repo = repo;
                  const labelObj = _.find(
                    issue.labels, label => _.includes(this.stages, label.name),
                  );
                  if (labelObj) {
                    issue.status = labelObj.name;
                  } else if (issue.state === 'open') {
                    issue.status = 'backlog';
                  } else {
                    issue.status = 'done';
                  }
                  this.allIssues.push(issue);
                });
                // Update the issues
                this.updateFilters();
                this.updateIssues();
              });
          });
        });
    },
    updateFilters() {
      // update assignee filters
      const assigneesQuery = _.split(this.$route.query.assignees);
      if (!_.isEmpty(assigneesQuery)) {
        this.assigneesValue = _.filter(this.assigneesOptions, assignee =>
          _.includes(assigneesQuery, assignee.login),
        );
      }
      // update milestone filters
      const milestonesQuery = _.split(this.$route.query.milestones);
      if (!_.isEmpty(milestonesQuery)) {
        this.milestonesValue = _.filter(this.milestonesOptions, milestone =>
          _.includes(milestonesQuery, milestone.title),
        );
      }
      // update label filters
      const labelsQuery = _.split(this.$route.query.labels);
      if (!_.isEmpty(labelsQuery)) {
        this.labelsValue = _.filter(this.labelsOptions, label =>
          _.includes(labelsQuery, label.name),
        );
      }
      // update repo filters
      const reposQuery = _.split(this.$route.query.repos);
      if (!_.isEmpty(reposQuery)) {
        this.reposValue = _.filter(this.reposOptions, repo =>
          _.includes(reposQuery, repo.full_name),
        );
      }
    },
  },
};
</script>

<style lang="scss">
@import './assets/kanban.scss';

$backlog: Grey;
$in-progress: #2A92BF;
$question: #D2B8A1;
$validation: #00B961;
$done: green;

* {
  box-sizing: border-box;
}

body {
  background: #33363D;
  -webkit-font-smoothing: antialiased;
}

.drag-column {
  &-backlog {
    .drag-column-header,
    .is-moved,
    .drag-options {
      background: $backlog;
    }
  }

  &-in-progress {
    .drag-column-header,
    .is-moved,
    .drag-options {
      background: $in-progress;
    }
  }

  &-question {
    .drag-column-header,
    .is-moved,
    .drag-options {
      background: $question;
    }
  }

  &-validation {
    .drag-column-header,
    .is-moved,
    .drag-options {
      background: $validation;
    }
  }

  &-done {
    .drag-column-header,
    .is-moved,
    .drag-options {
      background: $done;
    }
  }
}

.section {
  padding: 20px;
  text-align: center;
}
</style>

<style>
@import './assets/vue-multiselect.min.css';

.avatar {
  width: 30px;
  height: 30px;
  float: right;
}

.icon {
  width: 24px;
  height: 24px;
  float: right;
}

.issue-tag {
  border-radius: 2px;
  cursor: default;
  padding: 0 2px;
  font-size: 12px;
  display: inline-block;
  line-height: 14px;
  border: 1px solid #6c6c6c;
  max-width: 100%;
  margin: 1px 0 0 1px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  float: left;
}

.loader {
  position: absolute;
  left: 50%;
  top: 50%;
  z-index: 1;
  width: 150px;
  height: 150px;
  margin: -75px 0 0 -75px;
  border: 16px solid #f3f3f3;
  border-radius: 50%;
  border-top: 16px solid #3498db;
  width: 120px;
  height: 120px;
  -webkit-animation: spin 2s linear infinite;
  animation: spin 2s linear infinite;
}

@-webkit-keyframes spin {
  0% {
    -webkit-transform: rotate(0deg);
  }
  100% {
    -webkit-transform: rotate(360deg);
  }
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
</style>