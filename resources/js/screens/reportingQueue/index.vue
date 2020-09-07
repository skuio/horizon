<script type="text/ecmascript-6">
  import $ from 'jquery';
  import moment from 'moment-timezone';

  export default {
    /**
     * The component's data.
     */
    data()
    {
      return {
        tagSearchPhrase  : '',
        searchTimeout    : null,
        ready            : false,
        loadingNewEntries: false,
        hasNewEntries    : false,
        page             : 1,
        perPage          : 50,
        totalPages       : 1,
        jobs             : []
      };
    },

    /**
     * Prepare the component.
     */
    mounted()
    {
      document.title = "Horizon - Reporting Queue";

      this.loadJobs(this.$route.params.tag);

      this.refreshJobsPeriodically();
    },

    /**
     * Clean after the component is destroyed.
     */
    destroyed()
    {
      clearInterval(this.interval);
    },

    /**
     * Watch these properties for changes.
     */
    watch: {
      '$route'()
      {
        this.page = 1;

        this.loadJobs(this.$route.params.tag);
      },

      tagSearchPhrase()
      {
        clearTimeout(this.searchTimeout);
        clearInterval(this.interval);

        this.searchTimeout = setTimeout(() => {
          this.loadJobs();
          this.refreshJobsPeriodically();
        }, 500);
      }
    },

    methods: {
      /**
       * Load the jobs of the given tag.
       */
      loadJobs(starting = -1, refreshing = false)
      {
        if (!refreshing)
        {
          this.ready = false;
        }

        var tagQuery = this.tagSearchPhrase ? 'report=' + this.tagSearchPhrase + '&' : '';

        this.$http.get('/api/horizon/reporting-queue?' + tagQuery + 'page=' + this.page + '&limit=' + this.perPage)
          .then(response => {
            this.jobs = response.data.data;

            this.totalPages = response.data.last_page;

            this.ready = true;
          });
      },

      loadNewEntries()
      {
        this.jobs = [];

        this.loadJobs(-1, false);

        this.hasNewEntries = false;
      },

      /**
       * Refresh the jobs every period of time.
       */
      refreshJobsPeriodically()
      {
        this.interval = setInterval(() => {
          if (this.page != 1)
          {
            return;
          }

          this.loadJobs(-1, true);
        }, 3000);
      },

      /**
       * Load the jobs for the previous page.
       */
      previous()
      {
        this.page -= 1;

        this.loadJobs(
          (this.page - 2) * this.perPage
        );

        this.hasNewEntries = false;
      },

      /**
       * Load the jobs for the next page.
       */
      next()
      {
        this.page += 1;

        this.loadJobs(
          this.page * this.perPage
        );

        this.hasNewEntries = false;
      },

      cleanObject(obj)
      {
        Object.keys(obj).forEach((k) => (!obj[k] && obj[k] !== undefined) && delete obj[k]);

        return obj;
      },

      toDate(date)
      {
        return moment(date, 'YYYY-MM-DD\THH:mm:ssZZ').format('YYYY-MM-DD HH:mm:ss')
      },

      getObject(obj)
      {
        let result = '';

        Object.keys(obj).forEach((k) => {
          if (obj[k] && obj[k] !== undefined)
          {
            let value = obj[k];

            if (!Number.isInteger(value) && !isNaN(Date.parse(value)))
            {
              value = this.toDate(value);
            }
            result += '<li><small class="text-muted"><strong>' + k + '</strong><br>' + value + '</small></li>';
          }
        });

        return result;
      }
    }
  }
</script>

<template>
    <div>
        <div class="card">
            <div class="card-header d-flex align-items-center justify-content-between">
                <h5>Reporting Queue</h5>

                <input type="text" class="form-control" v-model="tagSearchPhrase" placeholder="Search Reports"
                       style="width:200px">
            </div>

            <div v-if="!ready"
                 class="d-flex align-items-center justify-content-center card-bg-secondary p-5 bottom-radius">
                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" class="icon spin mr-2 fill-text-color">
                    <path
                        d="M12 10a2 2 0 0 1-3.41 1.41A2 2 0 0 1 10 8V0a9.97 9.97 0 0 1 10 10h-8zm7.9 1.41A10 10 0 1 1 8.59.1v2.03a8 8 0 1 0 9.29 9.29h2.02zm-4.07 0a6 6 0 1 1-7.25-7.25v2.1a3.99 3.99 0 0 0-1.4 6.57 4 4 0 0 0 6.56-1.42h2.1z"></path>
                </svg>

                <span>Loading...</span>
            </div>


            <div v-if="ready && jobs.length == 0"
                 class="d-flex flex-column align-items-center justify-content-center card-bg-secondary p-5 bottom-radius">
                <span>There aren't any reports.</span>
            </div>

            <table v-if="ready && jobs.length > 0" class="table table-hover table-sm mb-0">
                <thead>
                <tr>
                    <th style="width:5%;">Link</th>
                    <th>Queued At</th>
                    <th>Event Type</th>
                    <th>Status</th>
                </tr>
                </thead>

                <tbody>
                <tr v-if="hasNewEntries" key="newEntries" class="dontanimate">
                    <td colspan="100" class="text-center card-bg-secondary py-1">
                        <small><a href="#" v-on:click.prevent="loadNewEntries" v-if="!loadingNewEntries">Load New
                            Entries</a></small>

                        <small v-if="loadingNewEntries">Loading...</small>
                    </td>
                </tr>

                <tr v-for="job in jobs" :key="job.id">
                    <td>
<!--                        <span :title="job.ReportType">{{job.ReportType.length > 35 ? job.ReportType.slice(0, 35) : job.ReportType}}</span><br>-->
                        <span :title="job.link_type">{{jobBaseName(job.link_type)}}</span><br>

                        <small class="text-muted">
                            {{jobBaseName(job.link_type)}}: {{job.link_id}}
                        </small>
                    </td>

                    <td class="table-fit">
                        {{ toDate(job.created_at) }}
                    </td>

                    <td class="table-fit">
                        {{ job.event_type }}
                    </td>

                    <td class="table-fit">

                        <span v-if="job.status === 'pending'" class="badge badge-secondary">Pending</span>

                        <span v-if="job.status === 'recording'" class="badge badge-info">Recording</span>

                        <span v-if="job.status === 'recorded'" class="badge badge-success">Recorded</span>

                        <span v-if="job.status === 'failed'" class="badge badge-danger">Failed</span>

                    </td>
                </tr>
                </tbody>
            </table>

            <div v-if="ready && jobs.length" class="p-3 d-flex justify-content-between border-top">
                <button @click="previous" class="btn btn-secondary btn-md" :disabled="page==1">Previous</button>
                <button @click="next" class="btn btn-secondary btn-md" :disabled="page>=totalPages">Next</button>
            </div>
        </div>

    </div>
</template>

<style>
    .badge{
        font-weight: 300;
    }
</style>
