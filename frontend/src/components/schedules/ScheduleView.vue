<template>
    <b-row>
      <b-col
        v-for="schedule in schedules"
        :key="schedule.name"
        cols="12"
        :md="numColumns">
        <Schedule
          :schedule="schedule"
          :courses="$store.getters.selectedCourses"
          :showEditButton="showEditButton"
          class="mb-4"
        ></Schedule>
      </b-col>
    </b-row>
</template>

<script>
//replace with new component
import Schedule from "./Schedule.vue";

export default {
  props: {
    //number of schedules to show (1,2,4)
    numShow: {
      //type: Number,
      required: true,
      default: 4,
    },
    courses: {
      type: Array,
      required: false,
    },
    schedules: {
      type: Array,
      required: false,
    },
    showEditButton: {
        type: Boolean,
        default: false
    }
  },
  components: {
    Schedule,
  },
  computed: {
    scheduleClass: function () {
      return "schedule-" + this.numShow;
    },
    numColumns: function () {
      switch (this.numShow) {
        case "1": return "12";
        case "2": return "6";
        case "4": return "6";
        default: return "12";
      }
    }
  },
};
</script>

<style>

</style>
