<template>
  <f7-page id="active-workout">
    <!-- Navbar -->
    <f7-navbar
      :title="workout.name"
      large
      class="no-hairline no-shadow"
      inner-class="text-align-center"
    >
      <!-- Close Button -->
      <f7-link
        id="close-button"
        slot="left"
        @click="prematureExit"
      >
        <f7-icon material="close" />
      </f7-link>
    </f7-navbar>
    
    <!-- Toolbar -->
    <f7-toolbar
      v-show="portraitMode"
      bottom
      :inner="false"
      class="no-hairline no-shadow"
      style="height: auto"
    >
      <WorkoutProgress
        :current-round="currentRound"
        :total-rounds="workout.rounds"
        :elapsed-workout-time="elapsedWorkoutTime"
        :workout-percentage="workoutPercentage"
      />
    </f7-toolbar>
    
    <f7-block>
      <!-- Finish Workout Button -->
      <div
        v-if="done && numbersEntered"
      >
        <BumpSuggestions
          v-if="anyBumpSuggestions"
          :bump-suggestions="bumpSuggestions"
          :workout-id="workout.id"
        />
        <f7-button
          class="col big-button"
          large
          fill
          raised
          @click="exitWorkout"
        >
          Finish Workout
        </f7-button>
      </div>
      
      <!-- Rest Panel -->
      <RestPanel
        v-if="rest && !done"
        v-show="numbersEntered"
        :rest-time="currentExercise.rest"
        :on-end-rest="onEndRest"
        :next-exercise="nextExercise"
      />
      
      <!-- Exercise Panel -->
      <ExercisePanel
        :exercise="currentExercise"
        :rest="rest"
        :numbers-entered="numbersEntered"
        :on-end-set="onEndSet"
        :enter-numbers="enterNumbers"
      />
      
      <!-- Workout Progress (Landscape Mode) -->
      <WorkoutProgress
        v-if="!portraitMode"
        :current-round="currentRound"
        :total-rounds="workout.rounds"
        :elapsed-workout-time="elapsedWorkoutTime"
        :workout-percentage="workoutPercentage"
      />
    </f7-block>
  </f7-page>
</template>

<script>
import { f7Page, f7Navbar, f7Link, f7Icon, f7Block, f7Toolbar, f7Button } from 'framework7-vue'
import { mapMutations } from 'vuex'
import ExercisePanel from '../components/ExercisePanel.vue'
import RestPanel from '../components/RestPanel.vue'
import WorkoutProgress from '../components/WorkoutProgress'
import BumpSuggestions from '../components/BumpSuggestions'

let deviceready = false
document.addEventListener('deviceready', function () {
  deviceready = true
})

export default {
  
  components: {
    WorkoutProgress,
    ExercisePanel,
    RestPanel,
    BumpSuggestions,
    f7Page,
    f7Navbar,
    f7Link,
    f7Icon,
    f7Block,
    f7Toolbar,
    f7Button
  },
  
  data: () => ({
    portraitMode: true,
    workout: {},
    exerciseSequence: [],
    startTime: null,
    now: null,
    endTime: null,
    currentExerciseIndex: 0,
    currentRound: 1,
    lastCompletedSetTime: null,
    workoutTimer: null,
    rest: false,
    restCountdown: false,
    numbersEntered: false,
    bumpSuggestions: {},
    done: false
  }),
  
  computed: {
    currentExercise () {
      return this.exerciseSequence[this.currentExerciseIndex]
    },
    nextExercise () {
      return this.exerciseSequence[this.currentExerciseIndex + 1]
    },
    workoutPercentage () {
      return Math.round((this.currentExerciseIndex + (this.rest ? 1 : 0)) / this.exerciseSequence.length * 100)
    },
    firstExerciseOfWorkout () {
      return this.currentRound === 1 && this.currentExerciseIndex === 0
    },
    firstExerciseOfRound () {
      return this.currentExerciseIndex % this.workout.exercises.length === 0
    },
    lastExerciseOfWorkout () {
      return this.currentExerciseIndex === this.exerciseSequence.length - 1
    },
    completed () {
      return this.$store.state.completed
    },
    elapsedWorkoutTime () {
      const ms = (this.endTime || this.now) - this.startTime
      let secs = Math.round(ms / 1000)
      const mins = Math.floor(secs / 60)
      secs -= (mins * 60)
      return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`
    },
    anyBumpSuggestions () {
      return Object.keys(this.bumpSuggestions).length > 0
    }
  },
  
  created: function () {
    this.workout = this.$store.state.workouts.filter(w => w.id === this.$f7route.params.workoutId)[0]
    this.exerciseSequence = []
    for (let i = 0; i < this.workout.rounds; i++) {
      this.exerciseSequence.push(...this.workout.exercises)
    }
    this.startTime = Date.now()
    this.now = Date.now()
    this.workoutTimer = setInterval(this.incrementTimeElapsed, 1000)
    
    if (deviceready) {
      this.portraitMode = screen.orientation.type.includes('portrait')
      window.addEventListener('orientationchange', () => {
        this.portraitMode = screen.orientation.type.includes('portrait')
        if (!this.portraitMode) {
          document.querySelector('#active-workout > .page-content').scroll({
            top: 70,
            left: 0
          })
        }
      })
    }
    if (window && window.plugins && window.plugins.insomnia) {
      window.plugins.insomnia.keepAwake()
    }
    if (navigator && navigator.notification) {
      navigator.notification.beep(1)
    }
  },
  
  methods: {
    
    ...mapMutations([
      'startActiveWorkout',
      'addCompletedSet'
    ]),
    
    // Handle completed set
    onEndSet () {
      // Record completed time of exercise
      this.lastCompletedSetTime = Date.now()
      
      // Mark that we're resting
      this.rest = true
      this.restCountdown = true
      
      // If this is the last exercise of the workout
      if (this.lastExerciseOfWorkout) {
        // Mark that we're done and record the end time of the workout
        this.done = true
        this.endTime = Date.now()
      }
    },
    
    enterNumbers (weight, reps) {
      // Initialize the workout record if this is the first exercise
      if (this.firstExerciseOfWorkout) {
        this.startActiveWorkout({
          workoutId: this.workout.id,
          startTime: this.startTime
        })
      }
      
      // Record completed set
      this.addCompletedSet({
        workoutId: this.workout.id,
        workoutStartTime: this.startTime,
        exerciseName: this.currentExercise.name,
        round: this.currentRound,
        weight,
        reps,
        completedTime: this.lastCompletedSetTime
      })
      
      if (weight > this.currentExercise.weight || reps > this.currentExercise.reps) {
        if (this.currentExercise.name in this.bumpSuggestions) {
          const setIndex = this.bumpSuggestions[this.currentExercise.name].sets.findIndex(set => set.weight === weight && set.reps === reps)
          if (setIndex > -1) {
            this.bumpSuggestions[this.currentExercise.name].sets[setIndex].rounds.push(this.currentRound)
          } else {
            this.bumpSuggestions[this.currentExercise.name].sets.push({
              weight,
              reps,
              rounds: [this.currentRound]
            })
          }
        } else {
          this.$set(this.bumpSuggestions, this.currentExercise.name, {
            current: {
              weight: this.currentExercise.weight,
              reps: this.currentExercise.reps
            },
            sets: [{
              weight,
              reps,
              rounds: [this.currentRound]
            }]
          })
        }
      }
      
      this.numbersEntered = true
      if (!this.restCountdown) {
        this.startNextSet()
      }
    },
    
    onEndRest () {
      this.restCountdown = false
      if (navigator && navigator.notification) {
        navigator.notification.beep(1)
      }
      if (this.numbersEntered) {
        this.startNextSet()
      }
    },
    
    incrementTimeElapsed () {
      if (this.done) {
        clearInterval(this.workoutTimer)
      } else {
        this.now = Date.now()
      }
    },
    
    startNextSet () {
      this.numbersEntered = false
      
      // Increment the currentExerciseIndex
      this.currentExerciseIndex++
      if (this.firstExerciseOfRound) {
        this.currentRound += 1
      }
      
      // End the rest
      this.rest = false
    },
    
    prematureExit () {
      this.$f7.dialog.confirm(
        'Are you sure that you wish to exit the workout? All sets recorded so far will be saved',
        'Exit Workout',
        () => {
          this.exitWorkout()
        }
      )
    },
    
    exitWorkout () {
      if (window && window.plugins) {
        window.plugins.insomnia.allowSleepAgain()
      }
      this.$f7router.navigate('/workout/' + this.workout.id)
    }
  }
}
</script>

<style scoped>
#active-workout {
  background-color: #0A1344
}
</style>
