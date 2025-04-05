<template>
  <div v-if="course">
    <!-- Course Header -->
    <div class="bg-white shadow rounded-lg p-6 mb-6">
      <div class="flex justify-between items-start">
        <div>
          <h1 class="text-2xl font-bold text-gray-900">
            {{ course.title }}
          </h1>
          <p class="text-gray-500">{{ course.code }}</p>
        </div>
        <div class="flex items-center space-x-4">
          <span
            class="px-3 py-1 text-sm font-medium rounded-full"
            :class="course.is_active ? 'bg-green-100 text-green-800' : 'bg-gray-100 text-gray-800'"
          >
            {{ course.is_active ? 'Active' : 'Inactive' }}
          </span>
          <button
            v-if="isStudent && !isEnrolled"
            @click="handleEnrollment"
            :disabled="loading"
            class="btn-primary"
          >
            <i class="fas fa-plus mr-2"></i>
            Enroll
          </button>
        </div>
      </div>

      <div class="mt-6 grid grid-cols-1 md:grid-cols-3 gap-6">
        <div>
          <h3 class="text-sm font-medium text-gray-500">Teacher</h3>
          <p class="mt-1 text-sm text-gray-900">{{ course.teacher?.username || 'No teacher assigned' }}</p>
        </div>
        <div>
          <h3 class="text-sm font-medium text-gray-500">Semester</h3>
          <p class="mt-1 text-sm text-gray-900">{{ course.semester }} {{ course.year }}</p>
        </div>
        <div>
          <h3 class="text-sm font-medium text-gray-500">Students</h3>
          <p class="mt-1 text-sm text-gray-900">{{ course.enrolled_students_count }}/{{ course.max_students }}</p>
        </div>
      </div>

      <div class="mt-6">
        <h3 class="text-sm font-medium text-gray-500">Description</h3>
        <p class="mt-1 text-sm text-gray-900">{{ course.description }}</p>
      </div>
    </div>

    <!-- Course Content Tabs -->
    <div class="bg-white shadow rounded-lg">
      <div class="border-b">
        <nav class="flex -mb-px">
          <button
            v-for="tab in tabs"
            :key="tab.id"
            @click="currentTab = tab.id"
            class="px-6 py-3 text-sm font-medium border-b-2 whitespace-nowrap"
            :class="[
              currentTab === tab.id
                ? 'border-primary-500 text-primary-600'
                : 'border-transparent text-gray-500 hover:text-gray-700 hover:border-gray-300'
            ]"
          >
            {{ tab.name }}
          </button>
        </nav>
      </div>

      <div class="p-6">
        <!-- Schedule Tab -->
        <div v-if="currentTab === 'schedule'">
          <div v-if="course.schedules.length > 0" class="space-y-4">
            <div
              v-for="schedule in course.schedules"
              :key="schedule.id"
              class="flex items-center p-4 bg-gray-50 rounded-lg"
            >
              <div class="flex-shrink-0 w-16 text-center">
                <span class="text-sm font-medium text-gray-900">{{ schedule.day }}</span>
              </div>
              <div class="ml-4">
                <p class="text-sm text-gray-900">
                  {{ formatTime(schedule.start_time) }} - {{ formatTime(schedule.end_time) }}
                </p>
                <p class="text-sm text-gray-500">Room: {{ schedule.room }}</p>
              </div>
            </div>
          </div>
          <div v-else class="text-center py-6 text-gray-500">
            No schedule available
          </div>
        </div>

        <!-- Assignments Tab -->
        <div v-if="currentTab === 'assignments'">
          <div v-if="course.assignments.length > 0" class="space-y-4">
            <div
              v-for="assignment in course.assignments"
              :key="assignment.id"
              class="border rounded-lg p-4"
            >
              <div class="flex justify-between items-start">
                <div>
                  <h4 class="text-lg font-medium text-gray-900">
                    {{ assignment.title }}
                  </h4>
                  <p class="mt-1 text-sm text-gray-500">
                    {{ assignment.description }}
                  </p>
                </div>
                <div class="text-right">
                  <p class="text-sm font-medium text-gray-900">
                    {{ assignment.total_points }} points
                  </p>
                  <p class="text-sm text-gray-500">
                    Due: {{ formatDate(assignment.due_date) }}
                  </p>
                </div>
              </div>
            </div>
          </div>
          <div v-else class="text-center py-6 text-gray-500">
            No assignments available
          </div>

          <!-- Add Assignment Button (Teacher Only) -->
          <div v-if="isTeacher" class="mt-6">
            <button
              @click="showAddAssignment = true"
              class="btn-primary"
            >
              <i class="fas fa-plus mr-2"></i>
              Add Assignment
            </button>
          </div>
        </div>

        <!-- Students Tab -->
        <div v-if="currentTab === 'students'">
          <div v-if="course.enrolled_students.length > 0" class="space-y-4">
            <div
              v-for="student in course.enrolled_students"
              :key="student.id"
              class="flex items-center justify-between p-4 bg-gray-50 rounded-lg"
            >
              <div class="flex items-center">
                <div class="flex-shrink-0">
                  <span class="inline-flex items-center justify-center h-10 w-10 rounded-full bg-primary-100 text-primary-600">
                    {{ student.username.charAt(0).toUpperCase() }}
                  </span>
                </div>
                <div class="ml-4">
                  <p class="text-sm font-medium text-gray-900">
                    {{ student.username }}
                  </p>
                  <p class="text-sm text-gray-500">
                    {{ student.email }}
                  </p>
                </div>
              </div>
              <div v-if="isTeacher">
                <router-link
                  :to="{ name: 'GradeDetail', params: { id: student.id }}"
                  class="text-primary-600 hover:text-primary-700"
                >
                  View Grades
                </router-link>
              </div>
            </div>
          </div>
          <div v-else class="text-center py-6 text-gray-500">
            No students enrolled
          </div>
        </div>
      </div>
    </div>

    <!-- Add Assignment Modal -->
    <div
      v-if="showAddAssignment"
      class="fixed inset-0 bg-gray-500 bg-opacity-75 flex items-center justify-center"
    >
      <div class="bg-white rounded-lg shadow-xl max-w-lg w-full mx-4">
        <div class="p-6">
          <h3 class="text-lg font-medium text-gray-900 mb-4">Add Assignment</h3>
          <form @submit.prevent="handleAddAssignment">
            <div class="space-y-4">
              <div>
                <label class="form-label" for="title">Title</label>
                <input
                  id="title"
                  v-model="newAssignment.title"
                  type="text"
                  class="form-input"
                  required
                />
              </div>
              <div>
                <label class="form-label" for="description">Description</label>
                <textarea
                  id="description"
                  v-model="newAssignment.description"
                  class="form-input"
                  rows="3"
                  required
                ></textarea>
              </div>
              <div>
                <label class="form-label" for="dueDate">Due Date</label>
                <input
                  id="dueDate"
                  v-model="newAssignment.due_date"
                  type="datetime-local"
                  class="form-input"
                  required
                />
              </div>
              <div>
                <label class="form-label" for="points">Total Points</label>
                <input
                  id="points"
                  v-model.number="newAssignment.total_points"
                  type="number"
                  min="0"
                  step="0.1"
                  class="form-input"
                  required
                />
              </div>
            </div>
            <div class="mt-6 flex justify-end space-x-3">
              <button
                type="button"
                @click="showAddAssignment = false"
                class="btn-secondary"
              >
                Cancel
              </button>
              <button
                type="submit"
                :disabled="assignmentLoading"
                class="btn-primary"
              >
                {{ assignmentLoading ? 'Adding...' : 'Add Assignment' }}
              </button>
            </div>
          </form>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'CourseDetail',

  props: {
    id: {
      type: String,
      required: true
    }
  },

  data() {
    return {
      loading: false,
      currentTab: 'schedule',
      showAddAssignment: false,
      assignmentLoading: false,
      newAssignment: {
        title: '',
        description: '',
        due_date: '',
        total_points: 0
      },
      tabs: [
        { id: 'schedule', name: 'Schedule' },
        { id: 'assignments', name: 'Assignments' },
        { id: 'students', name: 'Students' }
      ]
    }
  },

  computed: {
    course() {
      return this.$store.getters['courses/courseById'](this.id)
    },

    isStudent() {
      return this.$store.getters['auth/userRole'] === 'student'
    },

    isTeacher() {
      return this.$store.getters['auth/userRole'] === 'teacher'
    },

    isEnrolled() {
      return this.$store.getters['courses/isEnrolled'](this.id)
    }
  },

  async created() {
    await this.loadCourse()
  },

  methods: {
    async loadCourse() {
      try {
        await this.$store.dispatch('courses/fetchCourse', this.id)
      } catch (error) {
        console.error('Error loading course:', error)
      }
    },

    formatTime(time) {
      return new Date(`2000-01-01T${time}`).toLocaleTimeString([], {
        hour: '2-digit',
        minute: '2-digit'
      })
    },

    formatDate(date) {
      return new Date(date).toLocaleDateString([], {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
      })
    },

    async handleEnrollment() {
      this.loading = true
      try {
        await this.$store.dispatch('courses/enrollInCourse', this.id)
      } catch (error) {
        console.error('Error enrolling in course:', error)
      } finally {
        this.loading = false
      }
    },

    async handleAddAssignment() {
      this.assignmentLoading = true
      try {
        await this.$store.dispatch('courses/createAssignment', {
          ...this.newAssignment,
          course: this.id
        })
        this.showAddAssignment = false
        this.newAssignment = {
          title: '',
          description: '',
          due_date: '',
          total_points: 0
        }
        await this.loadCourse()
      } catch (error) {
        console.error('Error adding assignment:', error)
      } finally {
        this.assignmentLoading = false
      }
    }
  }
}
</script>