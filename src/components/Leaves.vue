<template lang="pug">
div(class='calendar')
  .row
    //- Upcoming leave
    .col-lg-6
      h3.text-md-center Upcoming leave
      p.text-md-center The first one upcoming, or currently active.

      .list-group-item.text-md-center(v-if='nearestLeave')
        p <strong> {{ nearestLeave.leave_type }} </strong>
        p {{ nearestLeave.description }}
        hr
        template(v-if='nearestLeave')
          | <strong>From:</strong> {{ nearestLeave.leave_start | moment('DD MMM YYYY - HH:mm') }}<br>
          | <strong>To:</strong> {{ nearestLeave.leave_end | moment('DD MMM YYYY - HH:mm') }}<br>
      p.alert.alert-info(v-else) 
        | No leaves coming up. Are you sure you don't need a break from all that hard work?

      hr

    //- Holidays
    .col-lg-6
      cmpHolidays

      hr

  .row
    //- All leaves
    .col-lg-6
      h3.text-md-center All Leaves
      p.text-md-center An overview of all your leaves.

      #allLeavesAccordion(v-for='(yg, year) in processedLeaves' role='tablist', aria-multiselectable='true')
        .card.card-top-blue
          .card-header(role='tab', :id='"leavesHeading-" + year', data-toggle='collapse', data-parent='#allLeavesAccordion', aria-expanded='false', :aria-controls='"allLeavesCollapse-" + year', :href='"#allLeavesCollapse-" + year')
            h4.card-title.text-md-center <strong>{{ year }}</strong>

          .collapse(role='tabpanel', :id='"allLeavesCollapse-" + year', :aria-labelledby='"leavesHeading-" + year')
            .card-block
              table.table
                tbody(v-for='(leaves, month) in yg')
                  tr
                    td &nbsp;
                    td
                      h5 <strong>{{ month | fullMonthString }}</strong>

                  template(v-for='leave in leaves')
                    tr
                      td <strong>{{ leave.leave_type }}</strong> 
                        .tag.float-md-right(v-bind:class='getTagStyleClass(leave)') 
                          .hidden-lg-down {{ leave.status }}
                          .hidden-xl-up.fa(:class='getStatusStyleClass(leave)')
                      td.text-md-right {{ leave.leave_start | moment('DD MMM - HH:mm') }}  → {{ leave.leave_end | moment('DD MMM - HH:mm') }}

    //- Pending leaves
    .col-lg-6
      h3.text-md-center(@click='showPendingLeaves = !showPendingLeaves') Pending leaves
        .btn.btn-default.pull-right
          i.fa(v-bind:class='[showPendingLeaves ? "fa-chevron-up" : "fa-chevron-down"]')
      p.text-md-center Still awaiting approval.

      div(v-if='showPendingLeaves')
        p.text-md-center(v-if='pendingLeaves.length === 0') <strong>No leaves awaiting approval.</strong> 
        #accordion(v-for='(leave, i) in sortLeaves(pendingLeaves)' role='tablist', aria-multiselectable='true')
          .card(:class='getRibbonStyleClass(leave)')
            .card-header.leave__header(role='tab', v-bind:id='"pendHeading-" + i', data-toggle='collapse', data-parent='#accordion', aria-expanded='false', v-bind:aria-controls='"pendCollapse-" + i', v-bind:href='"#pendCollapse-" + i')

              .row
                .col-sm-9
                  a  <strong>{{ leave.leave_type }}:</strong> {{ leave.description }}
                .col-sm-3 
                  .tag.float-md-right(:class='getTagStyleClass(leave)')
                    | {{ leave.status }}
                  
            .collapse(role='tabpanel', v-bind:id='"pendCollapse-" + i', v-bind:aria-labelledby='"pendHeading-" + i')
              .card-block.row
                .col-md-10
                  | <strong>From:</strong> {{ leave.leave_start | moment('DD MMM YYYY - HH:mm') }}<br>
                  | <strong>To:</strong> {{ leave.leave_end | moment('DD MMM YYYY - HH:mm') }}<br>
                .btn.btn-danger.col-md-2(@click='cancelPendingLeave(leave)')
                  i.fa.fa-ban.text-sm-center


</template>
<script>
import * as types from '../store/mutation-types';
import store from '../store';
import Holidays from './Holidays.vue';
import moment from 'moment';


export default {
  name: 'leaves',

  data () {
    return {
      leaves: [],
      showAllLeaves: false,
      showPendingLeaves: true,
      loaded: false,
    }
  },

  components: {
    cmpHolidays: Holidays,
  },

  created: function () {
    this.getLeaves();
  },
  watch: {
    leaveTypes: function(leaveTypes, newLeaveTypes){
      this.getLeaves();
    },
  },
  computed: {

    leaveTypes: function() {
      if(store.getters.leave_types)
        return store.getters.leave_types;
    },

    acceptedLeaves: function() {
      return this.leaves.filter(x => {
        if(x.status === store.getters.leave_statuses[2])
          return x;
      });
    },

    pendingLeaves: function() {
      return this.leaves.filter(x => {
        if(x.status === store.getters.leave_statuses[0] && moment().isSameOrBefore(x.leave_end, 'date')){
          return x;
        }
      });
    },

    processedLeaves: function() {
      if(this.leaves){
        let procLeaves = {};
        
        let leaves = this.leaves.filter(x => {
          if(x.status !== store.getters.leave_statuses[0] 
            && x.status !== store.getters.leave_statuses[3]) {
              x.year = x.leave_start.year();
              x.month = x.leave_start.month() + 1;
              return x;
            }
        });
        
        leaves.forEach((leave) => {
          if(!procLeaves[leave.year])
            procLeaves[leave.year] = {};

          if(!procLeaves[leave.year][leave.month])
            procLeaves[leave.year][leave.month] = [];

          procLeaves[leave.year][leave.month].push(leave);
        });

        for(let year in procLeaves) {
          for(let month in procLeaves[year]) {
            this.sortLeaves(procLeaves[year][month]).reverse();
          }
        }

        return procLeaves;
      }
    },

    //Filters for upcoming leaves, sorts them & returns the first result
    nearestLeave: function(val) {
      let upcoming = this.acceptedLeaves.filter(x => {
        return moment().isSameOrBefore(x.leave_end, 'date')
      });

      return this.sortLeaves(upcoming).reverse()[0];
    },
  },

  filters: {

    // Takes a month integer and returns a month-format
    fullMonthString: function(val) {
      return moment().month(val-1).format('MMM');
    }
  },

  methods: {
    //Displays a toast with message
    showToast(text) {
      this.$toast(text, 
        { 
          id: 'performance-toast',
          horizontalPosition: 'right',
          verticalPosition: 'top',
          duration: 1000,
          transition: 'slide-down',
          mode: 'override'
        });
    },

    //Cancels the pending
    cancelPendingLeave: function(lv) {

      store.dispatch(
        types.NINETOFIVER_API_REQUEST,
        {
          path: '/my_leaves/' + lv.id + '/',
          method: 'DELETE'
        }
      ).then((response) => {
        if(response.status == 200 || response.status == 204) {
            this.getLeaves();
            this.showToast('Leave successfully deleted.');
          } else {
            console.log(response);
            this.showToast('Error deleting Leave. Console has more information.');
          }
      });
    },

    //Load all leaves
    getLeaves: function() {
      
      store.dispatch(types.NINETOFIVER_API_REQUEST, {
        path: '/my_leaves/'
      }).then((response) => {
        
        //Converts the start / end datetime from strings to actual JS datetimes
        response.data.results.forEach(lv => {
          lv.leavedate_set.forEach(ld => {
            ld.starts_at = moment(ld.starts_at, 'YYYY-MM-DD HH:mm:ss');
            ld.ends_at = moment(ld.ends_at, 'YYYY-MM-DD HH:mm:ss');
          });
          
          lv['leave_start'] = lv.leavedate_set[0].starts_at;
          lv['leave_end'] = lv.leavedate_set[lv.leavedate_set.length-1].ends_at;

          let lType = this.leaveTypes.find(x => { return x.id === lv.leave_type});
          lv['leave_type'] = lType ? lType.display_label : 'UNKNOWN';
        });

        this.leaves = response.data.results
      }, () => {
        this.loading = false
      });
    },

    //Sorts the leaves from furthest in the future to furthest  in the past
    sortLeaves: function(val) {
      return val.sort(function(a, b) {
        a = a.leave_start;
        b = b.leave_start;

        return a > b ? -1 : (a < b ? 1 : 0);
      });
    },

    //Get the style class of the ribbon based on the leave_status
    getRibbonStyleClass: function(leave) {
      let tempObj = {
        [store.getters.leave_statuses[2]]: 'card-top-green',
        [store.getters.leave_statuses[1]]: 'card-top-red',
      };

      return tempObj[leave.status] || 'card-top-blue';                           
    },

    //Get the style class of the tag based on the leave_status
    getTagStyleClass: function(leave) {
      let tempObj = {
        [store.getters.leave_statuses[2]]: 'tag-success',
        [store.getters.leave_statuses[1]]: 'tag-danger',
      };

      return tempObj[leave.status] || 'tag-primary';
    },

    //Get the style based on the status of the leave
    getStatusStyleClass: function(leave) {
      let tempObj = {
        [store.getters.leave_statuses[2]]: 'fa-check',
        [store.getters.leave_statuses[1]]: 'fa-times'
      };

      return tempObj[leave.status] || 'fa-question';
    }

  },
}
</script>

<style lang="less" scoped>
.leave__header {
  cursor: pointer;
}

</style>
