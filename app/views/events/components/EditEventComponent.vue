<script>
import { mapGetters, mapMutations, mapActions } from 'vuex'
import _ from 'lodash'
import moment from 'moment'
import momentTimezone from 'moment-timezone'
import { HTML5_FMT_DATE_LOCAL, HTML5_FMT_TIME_LOCAL } from '../../../core/constants'
import { RRuleGenerator, rruleGeneratorModule } from 'vue2-rrule-generator'
import VueTimepicker from 'vue2-timepicker'
import MembersComponent from './MembersComponent'
import UserSearchComponent from './UserSearchComponent'
import gcApiHandler from '../../../core/social-handlers/GoogleCalendarHandler'

export default {
  name: 'EditEventComponent',
  props: {
    editType: {
      type: String,
      default: 'new'
    }
  },
  components: {
    'rrule-generator': RRuleGenerator,
    'user-search': UserSearchComponent,
    'time-picker': VueTimepicker
  },
  data () {
    return {
      isSuccess: false,
      inProgress: false,
      previewing: false,
      errorMessage: '',
      event: {},
      resetRRule: true // signal
    }
  },
  methods: {
    ...mapMutations('events', [
      'setEvent',
      'cancelPreviewEvent'
    ]),
    ...mapActions('events', [
      'saveEvent',
      'editEvent',
      'syncToGoogleFailed'
    ]),
    selectOwner (u) {
      Vue.set(this.event, 'owner', u._id)
      Vue.set(this.event, 'ownerName', u.name)
    },
    cancelPreviewOnCalendar () {
      this.cancelPreviewEvent()
      this.previewing = false
    },
    previewOnCalendar () {
      if (this.previewing) {
        this.cancelPreviewOnCalendar()
      } else {
        const tempEvent = _.cloneDeep(this.event)
        tempEvent._id = 'temp-event'
        tempEvent.name = 'preview-' + tempEvent.name
        tempEvent.instances = this.makeInstances(tempEvent)
        this.setEvent(tempEvent)
        this.previewing = true
      }
    },
    makeInstances (event) {
      const instance = {
        _id: 'temp-event-' + Math.random(),
        title: 'temp-' + event.name,
        owner: event.owner,
      }
      return this.rrule.all().map(d => {
        return Object.assign({}, instance, {
          startDate: d,
          endDate: new Date((+event.endDate) - (+event.startDate) + (+d))
        })
      })
    },
    addMember (m) {
      this.event.members.add(m)
    },
    syncToGoogleCalendar () {
      gcApiHandler.syncEventsToGC(this.event).then(res => {
        console.log('Synced to GC')
      }).catch(err => {
        console.log('Error syncing to GC:', err)
        this.syncToGoogleFailed(this.event._id).then(res => {
          console.log('Sync to GC failed')
        })
        noty({ text: 'Error syncing to Google Calendar', type: 'error' })
      })
    },
    async onFormSubmit () {
      this.inProgress = true
      this.cancelPreviewOnCalendar()
      this.event.type = 'online-classes'
      this.event.rrule = this.rrule.toString()
      if (!this.event.owner) {
        this.errorMessage = 'Must set an Owner'
        this.inProgress = false
        return
      }
      if (this.event.endDate <= this.event.startDate) {
        this.errorMessage = 'End date must be after start date'
        this.inProgress = false
        return
      }
      if (this.editType === 'new') {
        try {
          const res = await this.saveEvent(this.event)
          if (this.event.syncedToGC) {
            this.syncToGoogleCalendar()
          }
          this.$emit('save', res._id)
          this.inProgress = false
        } catch (err) {
          this.errorMessage = err.message
          setTimeout(() => {
            this.inProgress = false
          }, 3000)
        }
      } else {
        try {
          await this.editEvent(this.event)
          if (this.event.syncedToGC && !this.propsEvent?.syncedToGC) {
            this.syncToGoogleCalendar()
          }
          this.$emit('save', this.event._id)
          this.inProgress = false
        } catch (err) {
          this.errorMessage = err.message
        }
      }
    },
    eventUpdate () {
      const now = new Date()
      let date;
      if (this.clickedDate) {
        date = new Date(this.clickedDate).setHours(now.getHours())
      }
      const sDate = moment(date || now).set('minutes', 0).set('seconds', 0)
      if (this.editType === 'new') {
        this.event = {
          members: [],
          startDate: sDate.toDate(),
          endDate: sDate.clone().add(1, 'hours').toDate(),
          instances: []
        }
      } else {
        this.event = _.cloneDeep(this.propsEvent)
      }
      this.resetRRule = !this.resetRRule
    }
  },
  computed: {
    ...mapGetters({
      propsEvent: 'events/eventPanelEvent',
      clickedDate: 'events/eventPanelDate',
      rrule: 'rruleGenerator/rule'
    }),
    me () {
      return me
    },
    myTimeZone () {
      return momentTimezone.tz.guess()
    },
    _startDate: {
      get () {
        return moment(this.event.startDate).format(HTML5_FMT_DATE_LOCAL)
      },
      set (val) {
        // update startDate and endDate at the same time
        this.$set(this.event, 'startDate', moment(`${val} ${this._startTime}`).toDate())
        this.$set(this.event, 'endDate', moment(`${val} ${this._endTime}`).toDate())
      }
    },
    _startTime: {
      get () {
        return moment(this.event.startDate).format(HTML5_FMT_TIME_LOCAL)
      },
      set (val) {
        this.$set(this.event, 'startDate', moment(`${this._startDate} ${val}`).toDate())
      }
    },
    _endTime: {
      get () {
        return moment(this.event.endDate).format(HTML5_FMT_TIME_LOCAL)
      },
      set (val) {
        // use _startDate here since startDate and endDate share the date
        this.$set(this.event, 'endDate', moment(`${this._startDate} ${val}`).toDate())
      }
    },
    _endTimeHourRange () {
      if (this.event.startDate) {
        let date = this.event.startDate
        if (!(this.event.startDate instanceof Date)) {
          date = new Date(this.event.startDate)
        }
        return [[date.getHours(), 23]]
      } else {
        return [[0, 23]]
      }
    },
    _gcEmails: {
      get () {
        return (this.event.gcEmails || []).join('\n')
      },
      set (val) {
        this.$set(this.event, 'gcEmails', val.split(/\n|,/).map(e => e.trim()))
      }
    },
    rruleStart () {
      return moment(this.event.startDate).toDate()
    },
    rulePreviewTop6 () {
      return this.rrule.all((date, i) => i < 6).map(d => moment(d).format('ll'))
    }
  },
  created () {
    if (!this.$store.hasModule('rruleGenerator')) {
      this.$store.registerModule('rruleGenerator', rruleGeneratorModule)
    }
  },
  mounted () {
    this.eventUpdate()
  },
  watch: {
    propsEvent () {
      this.eventUpdate()
    }
  }
}
</script>

<template>
  <div>
    <div class="tab-label">
      {{ $t('events.edit_event_tab_desc') }}
    </div>
    <form
      class="edit-event-form"
      @submit.prevent="onFormSubmit"
    >
      <div class="from-group">
        <label for="name"> {{ $t('events.name') }}</label>
        <input
          v-model="event.name"
          name="name"
          class="form-control"
          type="text"
        >
      </div>
      <div class="from-group">
        <label for="description"> {{ $t('events.description') }}</label>
        <input
          v-model="event.description"
          name="description"
          class="form-control"
          type="text"
        >
      </div>
      <div class="from-group">
        <label for="owner"> {{ $t('events.owner') }}</label>
        <user-search
          :permissions="'onlineTeacher'"
          :value="event.ownerName"
          :no-results="'No online teachers found'"
          :placeholder="'Search email or username for an online-teacher'"
          @select="selectOwner"
        />
      </div>
      <div class="from-group">
        <label for="startDate"> {{ $t('events.start_date') }}</label>
        <div class="input-label">
          {{ $t('events.timezone_tips') + myTimeZone }}
        </div>
        <input
          v-model="_startDate"
          type="date"
          class="form-control"
          name="startDate"
        >
      </div>
      <div class="from-group">
        <label for="timeRange"> {{ $t('events.time_range') }}</label>
        <div>
          <time-picker format="HH:mm" :minute-interval="10" v-model="_startTime" />
          <span>-</span>
          <time-picker
            format="HH:mm"
            :minute-interval="10"
            :hour-range="_endTimeHourRange"
            v-model="_endTime"
          />
        </div>
      </div>

      <rrule-generator
        v-if="editType === 'new' || event.rrule"
        :start="rruleStart"
        :option="{showStart: false}"
        :rrule="event.rrule"
        :resetRRule="resetRRule"
      />

      <div class="form-group">
        <label for="meetingLink"> {{ $t('events.meetingLink') }}</label>
        <div>
          <input
            v-model="event.meetingLink"
            type="text"
            class="form-control"
            name="meetingLink"
            placeholder="input your zoom link here"
          >
        </div>
      </div>
      <div class="form-group" v-if="true || me.useGoogleCalendar()">
        <label for="importedToGC"> {{ $t(`events.sync${propsEvent?.syncedToGC ? 'ed' : ''}_to_google`) }}</label>
        <div class="input-label">
          {{ $t('events.sync_to_google_desc') }}
        </div>
        <input
          v-model="event.syncedToGC"
          type="checkbox"
          class="form-control"
          name="sync"
          :disabled="propsEvent?.syncedToGC"
        >
      </div>
      <div class="form-group" v-if="event.syncedToGC">
        <label for="gcEmails">{{$t('events.google_calendar_attendees')}} </label>
        <textarea
          v-model="_gcEmails"
          class="form-control gcEmails"
          name="gcEmails"
          placeholder="List emails here to get notification by google calendar, split by newline or comma"
        />
      </div>
      <div class="form-group pull-right">
        <span
          v-if="isSuccess"
          class="success-msg"
        >
          {{ $t('teacher.success') }}
        </span>
        <span
          v-if="errorMessage"
          class="error-msg"
        >
          {{ errorMessage }}
        </span>
        <button
          class="btn btn-success btn-lg"
          type="submit"
          :disabled="inProgress"
        >
          {{ $t('common.submit') }}
        </button>
      </div>
    </form>
    <div class="preview">
      <input
        class="btn btn-success btn-lg"
        type="button"
        :value="previewing ? 'Cancel' : 'Preview'"
        @click="previewOnCalendar"
      >
      <div
        v-for="date in rulePreviewTop6"
        :key="date"
      >
        {{ date }}
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.gcEmails {
  height: 10em;
}
.error-msg {
  color: red;
}
.tab-label, .input-label {
  font-size: 15px;
  color: rgba(128, 128, 128, 0.7);
}
</style>
