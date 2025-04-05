<template>
  <div>
    <!-- Header -->
    <div class="flex justify-between items-center mb-6">
      <h1 class="text-2xl font-bold text-gray-900">
        {{ isStudent ? 'Your Grades' : 'Student Grades' }}
      </h1>
    </div>

    <!-- Filters -->
    <div v-if="!isStudent" class="bg-white shadow rounded-lg p-4 mb-6">
      <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
        <!-- Course Filter -->
        <div>
          <label class="form-label" for="courseFilter">Course</label>
          <select
            id="courseFilter"
            v-model="filters.courseId"
            class="form-input"
          >
            <option value="">All Courses</option>
            <option
              v-for="course in teachingCourses"
              :key="course.id"
              :value="course.id"
            >
              {{ course.title }}
            </option>
          </select>
        </div>

        <!-- Student Search -->
        <div>
          <label class="form-label" for="studentSearch">Search Student</label>
          <input
            id="studentSearch"
            v-model="filters.search"
            type="text"
            class="form-input"
            placeholder="Search by name or ID..."
          />
        </div>
      </div>
    </div>

    <!-- Grades Table -->
    <div class="bg-white shadow rounded-lg overflow-hidden">
      <div class="overflow-x-auto">
        <table class="min-w-full divide-y divide-gray-200">
          <thead class="bg-gray-50">
            <tr>
              <th
                v-for="header in tableHeaders"
                :key="header.key"
                class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider"
              >
                {{ header.label }}
              </th>
            </tr>
          </thead>
          <tbody class="bg-white divide-y divide-gray-200">
            <tr v-for="grade in filteredGrades" :key="grade.id">
              <!-- Course Info -->
              <td class="px-6 py-4 whitespace-nowrap">
                <div class="text-sm font-medium text-gray-900">
                  {{ grade.course.title }}
                </div>
                <div class="text-sm text-gray-500">
                  {{ grade.course.code }}
                </div>
              </td>

              <!-- Student Info (Teacher View) -->
              <td v-if="!isStudent" class="px-6 py-4 whitespace-nowrap">
                <div class="text-sm font-medium text-gray-900">
                  {{ grade.student.username }}
                </div>
                <div class="text-sm text-gray-500">
                  ID: {{ grade.student.id }}
                </div>
              </td>

              <!-- Assignment Grades -->
              <td class="px-6 py-4">
                <div class="space-y-2">
                  <div
                    v-for="assignmentGrade in grade.assignment_grades"
                    :key="assignmentGrade.id"
                    class="text-sm"
                  >
                    <div class="font-medium text-gray-900">
                      {{ assignmentGrade.assignment.title }}
                    </div>
                    <div class="text-gray-500">
                      {{ assignmentGrade.points }}/{{ assignmentGrade.assignment.total_points }}
                      ({{ calculatePercentage(assignmentGrade.points, assignmentGrade.assignment.total_points) }}%)
                    </div>
                  </div>
                </div>
              </td>

              <!-- Final Grade -->
              <td class="px-6 py-4 whitespace-nowrap">
                <span
                  class="px-2 py-1 text-sm font-medium rounded-full"
                  :class="getGradeClass(grade.final_grade)"
                >
                  {{ formatGrade(grade.final_grade) }}
                </span>
              </td>

              <!-- Actions -->
              <td class="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">
                <router-link
                  :to="{ name: 'GradeDetail', params: { id: grade.id }}"
                  class="text-primary-600 hover:text-primary-900"
                >
                  View Details
                </router-link>
              </td>
            </tr>

            <!-- Empty State -->
            <tr v-if="filteredGrades.length === 0">
              <td
                :colspan="tableHeaders.length"
                class="px-6 py-10 text-center text-gray-500"
              >
                No grades found
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Course Statistics -->
    <div v-if="selectedCourseStats" class="mt-6 bg-white shadow rounded-lg p-6">
      <h2 class="text-lg font-medium text-gray-900 mb-4">Course Statistics</h2>
      <div class="grid grid-cols-1 md:grid-cols-4 gap-4">
        <div class="bg-gray-50 rounded-lg p-4">
          <div class="text-sm text-gray-500">Class Average</div>
          <div class="mt-1 text-2xl font-semibold text-gray-900">
            {{ formatGrade(selectedCourseStats.average) }}
          </div>
        </div>
        <div class="bg-gray-50 rounded-lg p-4">
          <div class="text-sm text-gray-500">Total Students</div>
          <div class="mt-1 text-2xl font-semibold text-gray-900">
            {{ selectedCourseStats.total_students }}
          </div>
        </div>
        <div class="bg-gray-50 rounded-lg p-4 col-span-2">
          <div class="text-sm text-gray-500 mb-2">Grade Distribution</div>
          <div class="flex items-center space-x-4">
            <div
              v-for="(count, grade) in selectedCourseStats.grade_distribution"
              :key="grade"
              class="text-center"
            >
              <div class="text-sm font-medium text-gray-900">{{ grade }}</div>
              <div class="text-sm text-gray-500">{{ count }}</div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'GradeList',

  data() {
    return {
      filters: {
        courseId: '',
        search: ''
      },
      selectedCourseStats: null
    }
  },

  computed: {
    isStudent() {
      return this.$store.getters['auth/userRole'] === 'student'
    },

    teachingCourses() {
      return this.$store.getters['courses/allCourses']
    },

    grades() {
      return this.$store.getters['grades/allGrades']
    },

    tableHeaders() {
      const headers = [
        { key: 'course', label: 'Course' }
      ]

      if (!this.isStudent) {
        headers.push({ key: 'student', label: 'Student' })
      }

      headers.push(
        { key: 'assignments', label: 'Assignments' },
        { key: 'final', label: 'Final Grade' },
        { key: 'actions', label: '' }
      )

      return headers
    },

    filteredGrades() {
      return this.grades.filter(grade => {
        const matchesCourse = !this.filters.courseId ||
          grade.course.id === this.filters.courseId

        const matchesSearch = !this.filters.search ||
          grade.student.username.toLowerCase().includes(this.filters.search.toLowerCase())

        return matchesCourse && matchesSearch
      })
    }
  },

  watch: {
    'filters.courseId': {
      async handler(courseId) {
        if (courseId) {
          await this.loadCourseStatistics(courseId)
        } else {
          this.selectedCourseStats = null
        }
      }
    }
  },

  async created() {
    await this.loadData()
  },

  methods: {
    async loadData() {
      try {
        await this.$store.dispatch('grades/fetchGrades')
        if (!this.isStudent) {
          await this.$store.dispatch('courses/fetchCourses')
        }
      } catch (error) {
        console.error('Error loading grades:', error)
      }
    },

    async loadCourseStatistics(courseId) {
      try {
        this.selectedCourseStats = await this.$store.dispatch(
          'grades/fetchCourseStatistics',
          courseId
        )
      } catch (error) {
        console.error('Error loading course statistics:', error)
      }
    },

    calculatePercentage(points, total) {
      return ((points / total) * 100).toFixed(1)
    },

    formatGrade(grade) {
      return grade ? `${grade.toFixed(1)}%` : 'N/A'
    },

    getGradeClass(grade) {
      if (!grade) return 'bg-gray-100 text-gray-800'
      
      if (grade >= 90) return 'bg-green-100 text-green-800'
      if (grade >= 80) return 'bg-blue-100 text-blue-800'
      if (grade >= 70) return 'bg-yellow-100 text-yellow-800'
      if (grade >= 60) return 'bg-orange-100 text-orange-800'
      return 'bg-red-100 text-red-800'
    }
  }
}
</script>