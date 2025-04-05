<template>
  <div>
    <div class="bg-white shadow rounded-lg p-6">
      <!-- Header -->
      <div class="mb-6">
        <h1 class="text-2xl font-bold text-gray-900">Create New Course</h1>
        <p class="mt-1 text-sm text-gray-500">
          Fill in the course details below to create a new course.
        </p>
      </div>

      <!-- Course Form -->
      <form @submit.prevent="handleSubmit" class="space-y-6">
        <!-- Basic Information -->
        <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
          <!-- Course Title -->
          <div>
            <label class="form-label" for="title">Course Title</label>
            <input
              id="title"
              v-model="form.title"
              type="text"
              class="form-input"
              required
            />
          </div>

          <!-- Course Code -->
          <div>
            <label class="form-label" for="code">Course Code</label>
            <input
              id="code"
              v-model="form.code"
              type="text"
              class="form-input"
              required
            />
          </div>

          <!-- Credits -->
          <div>
            <label class="form-label" for="credits">Credits</label>
            <input
              id="credits"
              v-model.number="form.credits"
              type="number"
              min="0"
              class="form-input"
              required
            />
          </div>

          <!-- Max Students -->
          <div>
            <label class="form-label" for="maxStudents">Maximum Students</label>
            <input
              id="maxStudents"
              v-model.number="form.max_students"
              type="number"
              min="1"
              class="form-input"
              required
            />
          </div>

          <!-- Semester -->
          <div>
            <label class="form-label" for="semester">Semester</label>
            <select
              id="semester"
              v-model="form.semester"
              class="form-input"
              required
            >
              <option value="FALL">Fall</option>
              <option value="SPRING">Spring</option>
              <option value="SUMMER">Summer</option>
            </select>
          </div>

          <!-- Year -->
          <div>
            <label class="form-label" for="year">Year</label>
            <select
              id="year"
              v-model="form.year"
              class="form-input"
              required
            >
              <option
                v-for="year in availableYears"
                :key="year"
                :value="year"
              >
                {{ year }}
              </option>
            </select>
          </div>
        </div>

        <!-- Description -->
        <div>
          <label class="form-label" for="description">Description</label>
          <textarea
            id="description"
            v-model="form.description"
            rows="4"
            class="form-input"
            required
          ></textarea>
        </div>

        <!-- Schedule Section -->
        <div>
          <div class="flex justify-between items-center mb-4">
            <h3 class="text-lg font-medium text-gray-900">Course Schedule</h3>
            <button
              type="button"
              @click="addSchedule"
              class="text-sm text-primary-600 hover:text-primary-500"
            >
              <i class="fas fa-plus mr-1"></i>
              Add Time Slot
            </button>
          </div>

          <div class="space-y-4">
            <div
              v-for="(schedule, index) in form.schedules"
              :key="index"
              class="flex items-center space-x-4 bg-gray-50 p-4 rounded-lg"
            >
              <!-- Day -->
              <div class="w-1/4">
                <label class="form-label" :for="'day-' + index">Day</label>
                <select
                  :id="'day-' + index"
                  v-model="schedule.day"
                  class="form-input"
                  required
                >
                  <option value="MON">Monday</option>
                  <option value="TUE">Tuesday</option>
                  <option value="WED">Wednesday</option>
                  <option value="THU">Thursday</option>
                  <option value="FRI">Friday</option>
                </select>
              </div>

              <!-- Start Time -->
              <div class="w-1/4">
                <label class="form-label" :for="'start-' + index">Start Time</label>
                <input
                  :id="'start-' + index"
                  v-model="schedule.start_time"
                  type="time"
                  class="form-input"
                  required
                />
              </div>

              <!-- End Time -->
              <div class="w-1/4">
                <label class="form-label" :for="'end-' + index">End Time</label>
                <input
                  :id="'end-' + index"
                  v-model="schedule.end_time"
                  type="time"
                  class="form-input"
                  required
                />
              </div>

              <!-- Room -->
              <div class="w-1/4">
                <label class="form-label" :for="'room-' + index">Room</label>
                <input
                  :id="'room-' + index"
                  v-model="schedule.room"
                  type="text"
                  class="form-input"
                  required
                />
              </div>

              <!-- Remove Button -->
              <button
                type="button"
                @click="removeSchedule(index)"
                class="text-red-600 hover:text-red-500"
              >
                <i class="fas fa-trash"></i>
              </button>
            </div>
          </div>
        </div>

        <!-- Error Message -->
        <div v-if="error" class="text-red-600 text-sm">
          {{ error }}
        </div>

        <!-- Form Actions -->
        <div class="flex justify-end space-x-3">
          <router-link
            to="/courses"
            class="btn-secondary"
          >
            Cancel
          </router-link>
          <button
            type="submit"
            :disabled="loading"
            class="btn-primary"
          >
            {{ loading ? 'Creating Course...' : 'Create Course' }}
          </button>
        </div>
      </form>
    </div>
  </div>
</template>

<script>
export default {
  name: 'CourseCreate',

  data() {
    const currentYear = new Date().getFullYear()
    return {
      form: {
        title: '',
        code: '',
        description: '',
        credits: 3,
        max_students: 30,
        semester: 'FALL',
        year: currentYear,
        schedules: []
      },
      loading: false,
      error: null,
      availableYears: Array.from({ length: 5 }, (_, i) => currentYear + i)
    }
  },

  methods: {
    addSchedule() {
      this.form.schedules.push({
        day: 'MON',
        start_time: '',
        end_time: '',
        room: ''
      })
    },

    removeSchedule(index) {
      this.form.schedules.splice(index, 1)
    },

    async handleSubmit() {
      this.loading = true
      this.error = null

      try {
        const course = await this.$store.dispatch('courses/createCourse', this.form)
        
        // Create schedules for the course
        if (this.form.schedules.length > 0) {
          const schedulePromises = this.form.schedules.map(schedule =>
            this.$store.dispatch('courses/addSchedule', {
              courseId: course.id,
              schedule
            })
          )
          await Promise.all(schedulePromises)
        }

        this.$router.push({
          name: 'CourseDetail',
          params: { id: course.id }
        })
      } catch (error) {
        this.error = error.response?.data?.detail || 'Failed to create course'
      } finally {
        this.loading = false
      }
    }
  }
}
</script>