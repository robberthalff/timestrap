<template>
  <div class="container">
    <div class="row py-2 mb-4 bg-light rounded">
      <div class="col-6">
        <template v-if="is_staff">
          <button
            id="export-report"
            class="btn btn-primary btn-sm mr-1"
            @click="exportReport">
            <icon
              :icon="['fas', 'download']"
              class="mr-1"/>
            Export Report
          </button>
          <select
            v-model="exportFormat"
            class="export-select custom-select form-control-sm w-25">
            <option value="csv">csv</option>
            <option value="xls">xls</option>
            <option value="tsv">tsv</option>
            <option value="ods">ods</option>
            <option value="json">json</option>
            <option value="yaml">yaml</option>
            <option value="html">html</option>
          </select>
        </template>
      </div>
      <div class="col-6">
        <button
          class="btn btn-secondary btn-sm float-right ml-2"
          @click.prevent
          @click="refresh">
          <icon
            :icon="['fas', 'sync']"
            class="mr-1"/>
          Sync
        </button>
        <pager
          :next="next"
          :previous="previous"
          @next-page="getEntries(next)"
          @previous-page="getEntries(previous)"/>
      </div>
    </div>

    <form
      name="report-filters"
      class="row mb-4 pt-3 pb-1 bg-light rounded report-filters"
      @submit.prevent
      @submit="getReport">
      <div class="col-sm-6">
        <div class="form-group">
          <label>User</label>
          <select2
            id="report-filter-user"
            v-model="user"
            :options="users"
            placeholder="User"
            allowclear/>
        </div>
        <div class="form-group">
          <label>Client</label>
          <select2
            id="report-filter-client"
            v-model="client"
            :options="clients"
            placeholder="Client"
            allowclear/>
        </div>
        <div class="form-group pb-4">
          <label>Project</label>
          <select2
            id="report-filter-project"
            v-model="project"
            :options="projects"
            placeholder="Project"
            allowclear/>
        </div>
        <div class="form-group">
          <div class="row">
            <div class="col-md-6">
              <select2
                id="report-sort-by"
                v-model="orderBy"
                :options="orderByOptions"
                selected="date"
                placeholder="Order by"/>
            </div>
            <div class="col-md-6">
              <select2
                id="report-order-dir"
                v-model="orderDir"
                :options="orderDirOptions"
                selected="desc"
                placeholder="Order direction"/>
            </div>
          </div>
        </div>
      </div>
      <div class="col-sm-6">
        <div class="form-group">
          <label>Task</label>
          <select2
            id="report-filter-task"
            v-model="task"
            :options="tasks"
            placeholder="Task"
            allowclear/>
        </div>
        <div class="form-group">
          <label>Min. Date</label>
          <datepicker
            id="report-filter-min-date"
            v-model="minDate"
            type="text"
            class="form-control form-control-sm date-input"
            placeholder="Min. date"
            allowclear/>
        </div>
        <div class="form-group pb-4">
          <label>Max. Date</label>
          <datepicker
            id="report-filter-max-date"
            v-model="maxDate"
            type="text"
            class="form-control form-control-sm date-input"
            placeholder="Max. date"
            allowclear/>
        </div>
        <button
          id="generate-report"
          type="submit"
          class="btn btn-primary btn-sm w-100">
          Generate Report
        </button>
      </div>
    </form>

    <div
      v-if="this.$perms.view_entry"
      id="entry-rows">
      <div class="mb-4">
        <div class="entry-rows rounded">
          <entry
            v-for="(entry, index) in entries"
            :entry="entry"
            :index="index"
            :key="entry.id"
            :editable="editable"/>
        </div>
      </div>

      <div class="row bg-success text-white py-2 mb-4 rounded">
        <div class="ml-auto col-sm-2 text-right">
          Subtotal<br>
          <strong>Total</strong>
        </div>
        <div class="col-sm-4">
          <icon
            :icon="['fas', 'clock']"
            class="mr-1"/>
          {{ $moment.duration(subtotal, 'hours').format('d[d] h[h] m[m]') }}<br>
          <icon
            :icon="['fas', 'clock']"
            class="mr-1"/>
          <strong>{{ $moment.duration(total, 'hours').format('d[d] h[h] m[m]') }}</strong>
        </div>
      </div>
    </div>

    <div class="row py-2 mb-4 bg-light rounded">
      <div class="col-12">
        <button
          class="btn btn-secondary btn-sm float-right ml-2"
          @click.prevent
          @click="refresh">
          <icon
            :icon="['fas', 'sync']"
            class="mr-1"/>
          Sync
        </button>
        <pager
          :next="next"
          :previous="previous"
          @next-page="getEntries(next)"
          @previous-page="getEntries(previous)"/>
      </div>
    </div>
  </div>
</template>


<script>
import {mapGetters, mapState, mapActions, mapMutations} from 'vuex';
import $ from 'jquery';

import Datepicker from '../datepicker.vue';
import Entry from '../entry.vue';
import Pager from '../pager.vue';
import Select2 from '../select2.vue';

import fetch from '../../fetch';


export default {
  components: {
    Datepicker,
    Entry,
    Pager,
    Select2,
  },
  data() {
    return {
      is_staff: timestrapConfig.USER.IS_STAFF,
      editable: false,
      next: null,
      previous: null,

      exportFormat: 'csv',

      orderByOptions: [
        {id: 'project__client__name', text: 'Client'},
        {id: 'date', text: 'Date'},
        {id: 'project__name', text: 'Project'},
        {id: 'task__name', text: 'Task'},
        {id: 'user__username', text: 'User'},
      ],
      orderDirOptions: [
        {id: 'asc', text: 'Ascending'},
        {id: 'desc', text: 'Descending'},
      ],

      orderBy: 'date',
      orderDir: 'desc',

      minDate: null,
      maxDate: null,
      user: null,
      client: null,
      project: null,
      task: null,
    };
  },
  computed: {
    ...mapState({
      entries: state => state.reports.all,
      total: state => state.reports.total,
      subtotal: state => state.reports.subtotal,
    }),
    ...mapGetters({
      tasks: 'tasks/getSelectTasks',
      projects: 'clients/getSelectProjects',
      clients: 'clients/getSelectClients',
      users: 'users/getSelectUsers',
    }),
  },
  mounted() {
    return this.getReport();
  },
  methods: {
    ...mapMutations('reports', [
      'setOrderBy',
      'setOrderDir',

      'setUser',
      'setClient',
      'setProject',
      'setTask',
      'setMinDate',
      'setMaxDate',
    ]),
    getReport() {
      this.setOrderBy(this.orderBy);
      this.setOrderDir(this.orderDir);

      this.setUser(this.user);
      this.setClient(this.client);
      this.setProject(this.project);
      this.setTask(this.task);
      this.setMinDate(this.minDate);
      this.setMaxDate(this.maxDate);
      this.$store.dispatch('reports/getReport');
    },
    exportReport() {
      let ordering = (this.orderDir == 'desc' ? '-' : '') + this.orderBy;
      if (this.orderBy != 'date') {
        ordering += ',-date';
      }
      const query = {
        ordering: ordering,
        user: this.user,
        project: this.project,
        project__client: this.client,
        min_date: this.minDate,
        max_date: this.maxDate,
        task: this.task,
        exportFormat: this.exportFormat,
      };
      document.location.href = timestrapConfig.CORE_URLS.REPORTS_EXPORT + '?' + $.param(query);
    },
    refresh() {
      return this.getEntries();
    },
  },
};
</script>


<style lang="scss">
.report-filters {
  label {
    font-weight: bold;
    font-size: .7em;
    letter-spacing: 1px;
    text-transform: uppercase;
  }
}
</style>
