<template>
    <b-card style="">
        <template v-slot:header>
            <b-pagination
                id="schedule-paginator"
                class="mb-1"
                align="fill"
                size="sm"
                first-number
                last-number
                v-model="currentPage"
                :total-rows="filteredAndSortedSchedules.length"
                :per-page="currentView"
                aria-controls="calendar">
            </b-pagination>
            <b-nav>
                <b-nav-item
                    v-b-tooltip.hover title="Change view"
                    @click="changeView">
                    <font-awesome-icon
                        size="sm"
                        :icon="viewIcons[currentView]"
                    />
                </b-nav-item>

                <!-- Optional quarter selector -->
                <b-nav-item-dropdown
                    class="filter-drop-down"
                    text="Quarter" v-if="showQuarterSelector">
                    <b-dropdown-header id="dropdown-header-label">
                        <small>Filter by quarter</small>
                    </b-dropdown-header>
                    <b-form-radio-group
                      v-model="currentQuarter"
                      :options="quarters"
                      :disabled="quarters.length == 0"
                      text-field="name"
                      value-field="quarter"
                      v-on:change="e => confirmUserSelection(e, currentQuarter, 'currentQuarter')"
                      class="ml-3">
                  </b-form-radio-group>
                </b-nav-item-dropdown>

                <b-nav-item-dropdown
                    class="filter-drop-down"
                    text="Days">
                    <b-dropdown-header id="dropdown-header-label">
                        <small>Show schedules with the selected days.</small>
                    </b-dropdown-header>
                    <b-dropdown-form>
                        <b-form-checkbox-group
                            id="daySelector"
                            v-model="filtering.selected"
                            :options="filtering.options"
                            name="daySelector"
                            switch
                            stacked
                        ></b-form-checkbox-group>
                        <b-dropdown-divider/>
                        <b-form-checkbox
                            v-model="filtering.allSelected"
                            :indeterminate="filtering.options.length != filtering.selected.length"
                            aria-describedby="daySelector"
                            aria-controls="daySelector"
                            @change="toggleDays">
                            {{ filtering.allSelected ? 'Deselect All' : 'Select All' }}
                        </b-form-checkbox>
                    </b-dropdown-form>
                </b-nav-item-dropdown>

                <b-nav-item-dropdown
                    class="filter-drop-down"
                    text="Time">
                        <b-dropdown-header id="dropdown-header-label">
                        <small>Show schedules within this time range.</small>
                    </b-dropdown-header>
                    <b-dropdown-form class="timeslider">
                        <vue-slider
                            v-model="filtering.selectedTimes"
                            :data="timeRangeForFiltering"
                            :adsorb="true"
                            :marks="false"
                            :tooltip-placement="'bottom'"
                            :tooltip="'always'"
                            :enable-cross="false"
                            :tooltip-formatter="formatSliderToolTip"/>
                    </b-dropdown-form>
                </b-nav-item-dropdown>

                <b-nav-item-dropdown
                    class="filter-drop-down"
                    text="Sort">
                    <b-dropdown-header id="dropdown-header-label">
                        <small>Sort the schedules by one of the following.</small>
                    </b-dropdown-header>
                    <b-dropdown-form>
                        <b-form-radio-group
                            v-model="sorting.attributes.selected"
                            :options="sorting.attributes.options"
                            name="scheduleSort"
                            switch
                            stacked>
                        </b-form-radio-group>
                    </b-dropdown-form>
                </b-nav-item-dropdown>

                <!-- Button that links to /user -->
                <b-nav-item
                    v-b-tooltip.hover title="Show favorite schedules">
                    <router-link
                    v-if="showFavoritesButton"
                    :to="{name:'user'}">
                      <font-awesome-icon
                          id="showFavoritedSchedules"
                          icon="heart"
                          size="sm"
                          style="color: #ED0303;">
                        </font-awesome-icon>
                    </router-link>

                    <router-link
                    v-else
                    :to="{name:'home'}">
                      <font-awesome-icon
                        icon="heart"
                        size="sm"
                        :id="'favorite-icon'+_uid"
                        style="color: #FFC7C7;">
                      </font-awesome-icon>
                      </router-link>
                </b-nav-item>



                <b-nav-item class="ml-auto"
                    v-b-tooltip.hover title="Reset filters">
                    <font-awesome-icon
                        id="resetFiltersAndSorters"
                        icon='undo'
                        size="sm"
                        @click="resetFiltersAndSorters"/>
                </b-nav-item>

                <b-nav-item
                    v-b-tooltip.hover title="Help">
                    <font-awesome-icon
                        v-b-modal.filteringHelpModal
                        id="help-circle"
                        icon='info-circle'
                        size="sm"/>
                </b-nav-item>

                <b-modal
                   id="filteringHelpModal"
                   title="Filtering and sorting schedules"
                   scrollable>
                   <p class="my-4">Once you have selected your classes, you can use this toolbar to find your favorite schedules.</p>
                   <p class="my-4">
                       Click the <strong>Days</strong> dropdown to hide schedules that have classes on Fridays (or any other day). Use the <strong>Time</strong> dropdown to hide schedules that have early or late classes.
                       Finally, you can <strong>Sort</strong> your schedules to see which have the least gaps between classes, or by the start time.
                   </p>
                   <p class="my-4">
                       When you find a schedule that you like, press <strong>Save</strong>. You can see all your saved schedules in your dashboard (top right corner).
                   </p>
               </b-modal>

            </b-nav>
        </template>
        <b-alert
            v-if="scheduleSavedStatus == 'successful'"
            :show="(scheduleSavedStatus == 'successful') ? 3 : 0"
            dismissible
            fade
            variant="success"
            @dismissed="scheduleSavedStatus = null">
            Schedule saved! Click My Schedules in the top right to view it.
        </b-alert>
        <b-alert
            v-if="(filteredAndSortedSchedules.length == 0 && lastUsedClassSections.length != 0 && schedules.length != 0) ? true : false"
            :show="(filteredAndSortedSchedules.length == 0 && lastUsedClassSections.length != 0 && schedules.length != 0) ? true : false"
            variant="danger">
            No schedules match your selections. Try selecting more days or more times.
        </b-alert>
        <b-alert
          v-if="(schedules.length == 0 && showFavoritesButton == true) ? true : false"
          :show="(schedules.length == 0 && showFavoritesButton == true) ? true : false"
          variant="info"
          class="text-center">
          Your schedules will appear here. <br>
          Use the course selectors to add courses that you want to include in your schedules.
        </b-alert>
        <b-alert
          v-if="(schedules.length == 0 && showFavoritesButton == false) ? true : false"
          :show="(schedules.length == 0 && showFavoritesButton == false) ? true : false"
          variant="info"
          class="text-center">
          You haven't saved any schedules yet! Try generating some on the Home page
          and save them by clicking the heart icon on each schedule. Your saved schedules will be shown here.
        </b-alert>
        <b-alert
            v-if="(scheduleSavedStatus == 'failed')"
            :show="(scheduleSavedStatus == 'failed') ? 3 : 0"
            dismissible
            fade
            variant="danger"
            @dismissed="scheduleSavedStatus = null">
            There was a problem saving your schedule.
        </b-alert>
        <b-alert
            v-if="selectedClassSectionsHaveUpdated"
            variant="warning"
            :show="selectedClassSectionsHaveUpdated">
            Your class selections have changed. Refreshing your schedules...
        </b-alert>
        <ListView
          :schedule="this.filteredAndSortedSchedules.slice((this.currentPage-1)*this.currentView, this.currentPage*this.currentView)"
          v-if="currentView == 10"
          :showEditButton="showEditButton"
          :courses="$store.getters.selectedCourses">
        </ListView>
        <ScheduleView
          :schedules="schedulesToRender"
          :numShow="currentView"
          :showEditButton="showEditButton"
          v-else>
        </ScheduleView>
    </b-card>
</template>

<script>
import VueSlider from 'vue-slider-component';
import 'vue-slider-component/theme/default.css';
import ListView from './ListView.vue';
import ScheduleView from './ScheduleView.vue';
import { getQuarters } from '@/components/util/util-methods.js';

export default {
    name: 'SchedulePaginator',
    components: {
        VueSlider,
        ListView,
        ScheduleView
    },
    props: {
        schedules: {
             type: Array,
             required: true,
             default: () => []
        },
        showQuarterSelector: {
            type: Boolean,
            required: false,
            default: false,
        },
        showFavoritesButton: {
            type: Boolean,
            required: false,
            default: true,
        },
        showEditButton: {
            type: Boolean,
            default: false
        }
    },
    created: function() {
        // Register event hooks
        this.$eventHub.$on('generate-schedules', this.resetView); //Component is more reliable when set to initial view on this event

        this.quarters = this.getQuarters();

        // Use a setter to initialize selectedClassSectionsHaveUpdated to false?
    },
    data: function() {
        return {
            currentPage: 1,
            currentView: "4",
            lastUsedClassSections: [],
            errors: [],
            quarters: [],
            savingScheduleInProgress: false,
            scheduleSavedStatus: null,
            sorting: {
                attributes: {
                    selected: 'totalMinutesFromMidnight',
                    options: [
                        { text: 'Earliest start time', value: 'totalMinutesFromMidnight' },
                        { text: 'Smallest gaps', value: 'totalMinutesBetweenEvents' },
                    ]
                },
                order: {
                    selected: 'descending',
                    options: [
                        { text: 'Descending', value: 'descending' },
                        { text: 'Ascending', value: 'ascending' },
                    ]
                },
            },
            filtering: {
                selectedTimes: ['06:00', '19:00'],
                selected: ['MONDAY', 'TUESDAY', 'WEDNESDAY', 'THURSDAY', 'FRIDAY'],
                allSelected: true,
                options: [
                    { text: 'Monday', value: 'MONDAY' },
                    { text: 'Tuesday', value: 'TUESDAY' },
                    { text: 'Wednesday', value: 'WEDNESDAY' },
                    { text: 'Thursday', value: 'THURSDAY' },
                    { text: 'Friday', value: 'FRIDAY' },
                ]
            },
            sortingInProgress: false,
            viewIcons: {1: "calendar", 2: "columns", 4:"border-all", 10: "list"},
        }
    },
    computed: {
          schedulesToRender: function(){
            var end = ((this.currentPage-1)*this.currentView) + parseInt(this.currentView);
            var pass = this.filteredAndSortedSchedules.slice((this.currentPage-1)*this.currentView,end);
            return pass;
          },
          /**
           * Computed prop for the selectedQuarter.
           * Needed for optional quarter selector.
           */
          currentQuarter: {
            get: function() {
              return this.$store.state.selectedQuarter;
            },
            set: function(newQuarter) {
              this.$nextTick(() => this.$store.commit('setSelectedQuarter', newQuarter));
            }
          },
         timeRangeForFiltering: function() {
            let minTime = '01:00';
            let maxTime = '23:00';

            let intervals = [];

            if (this.schedules.length != 0) {
                this.schedules.forEach(schedule => {
                    minTime = (schedule.minTime < minTime) ? schedule.minTime : minTime;
                    maxTime = (schedule.maxTime > maxTime) ? schedule.maxTime : maxTime;
                });
            }

            maxTime = this.$date(maxTime, 'HH:mm').add(1, 'hour').startOf('hour');
            minTime = this.$date(minTime, 'HH:mm').subtract(1, 'hour').startOf('hour');

            while (minTime < maxTime) {
                intervals.push(minTime.format('HH:mm'));
                minTime = minTime.add(30, 'minute');
            }

            return intervals;
        },
        filteredAndSortedSchedules: function() {
            const sortBy = this.sorting.attributes.selected;
            this.schedules.map((x, index) => x.name = x.name ?  x.name : "Schedule "+index);
            return this.schedules
                .filter(sched => {
                    const withinDays = this.checkIfSubset(this.filtering.selected, sched.sortingAttributes.daysWithEvents);
                    const withinTime = (
                        sched.sortingAttributes.earliestBeginTime > this.filtering.selectedTimes[0] &&
                        sched.sortingAttributes.latestEndTime < this.filtering.selectedTimes[1]
                        )
                    return withinDays && withinTime;
                })
                .sort((a,b) => (a.sortingAttributes[sortBy] > b.sortingAttributes[sortBy]) ? 1 : -1 )
        },
        /**
         *
         */
        selectedClassSectionsHaveUpdated: function() {
            if (this.lastUsedClassSections.length == 0) { // For when the component is first created
                return false;
            } else {
                // (big) TODO: This should also check the contents
                return this.lastUsedClassSections.length != this.classSections.length;
            }
        },
        customEvents: function(){
          return this.$store.getters.selectedCustomEvents;
        },
        classSections: function(){
          return this.$store.getters.selectedClassSections;
        }
    },
    methods: {
      getQuarters: function() {
        return getQuarters();
      },
        //simple method for resetting view to default
        resetView: function(){
          this.currentView = "4";
        },
        //Method for changing shown view
        changeView: function() {
            let arr = Object.keys(this.viewIcons);
            this.currentView = arr[(arr.indexOf(this.currentView) + 1) % arr.length];
        },
        /**
         * Resets the selected days (in the filter). If any days selected, this method deselects all options. If none
         * are selected, this will select all options.
         */
        toggleDays: function(isChecked) {
            this.filtering.selected = isChecked ? this.filtering.options.map(a => a.value) : [];
        },
        /**
         * Resets the selections for the days filter, time filter, and sorter.
         */
        resetFiltersAndSorters() {
            this.filtering.selected = this.filtering.options.map(a => a.value);
            this.filtering.selectedTimes =  ['07:00', '19:00'];
            this.sorting.attributes.selected = this.sorting.attributes.options[0].value;
        },
        /**
         * Checks if "smaller" is a subset of "bigger". Assumes the elements of each array are unique.
         */
        checkIfSubset: function(bigger, smaller) {
            if (smaller.length > bigger.length) {
                return false;
            } else {
                return smaller.every(val => bigger.includes(val));
            }
        },
        formatSliderToolTip (val) {
            return this.$date(val, 'HH:mm').format('h:mm a');
        },
        /**
         * Intercepts changes in quarter/session selectors and prompts the user for confirmation.
         */
        confirmUserSelection(newVal, oldVal, selection) {
          const modalOptions = {
            title: 'Are you sure?',
            headerBgVariant: 'warning',
            okTitle: 'Continue'
          };

          if (this.$store.state.selectedCourses.length > 0 && newVal != oldVal) {
            this.$bvModal
              .msgBoxConfirm('Changing your quarter or session will clear your courses and schedules.', modalOptions)
              .then(ok => {
                if (ok) {
                  this[selection] = newVal;
                } else { // cancel
                  this[selection] = oldVal; // reset
                }
              })
          } else {
            this[selection] = newVal;
          }

          this.currentView = "1";

        },
    },
    watch: {
        'sorting.attributes.selected': function() {
            this.currentPage = 1;
        },
        'filtering.selected': function() {
            this.currentPage = 1;
        },
        schedules: function() {
          this.lastUsedClassSections = this.classSections;  // Set this as the last used set of events. This is used to check for changes later
        }
    },
}
</script>

<style scoped>

/deep/ .nav-link.dropdown-toggle {
    padding: 0.5rem;
}

.filter-drop-down {
    font-size: 0.85rem;
    padding-top: 0.25rem;
}

.card-header {
    padding: 0px 0px 6px 0px ;
}

.change-view-wrapper {
    border: 1px solid rgba(0, 0, 0, 0.125);
    border-radius: 0.25rem;
}

.timeslider {
    width: 400px;
}

.timeslider .b-dropdown-form:focus {
    outline: none !important;
}
</style>
