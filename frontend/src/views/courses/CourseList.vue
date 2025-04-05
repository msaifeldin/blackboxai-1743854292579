<template>
  <div>
    <!-- Header -->
    <div class="flex justify-between items-center mb-6">
      <h1 class="text-2xl font-bold text-gray-900">
        {{ isStudent ? 'Available Courses' : 'Your Courses' }}
      </h1>
      <router-link
        v-if="isTeacher"
        to="/courses/create"
        class="btn-primary flex items-center"
      >
        <i class="fas fa-plus mr-2"></i>
        Create Course
      </router-link>
    </div>

    <!-- Filters -->
    <div class="bg-white shadow rounded-lg p-4 mb-6">
      <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
        <!-- Search -->
        <div>
          <label class="form-label" for="search">Search</label>
          <div class="relative">
            <input
              id="search"
              v-model="filters.search"
              type="text"
              class="form-input pl-10"
              placeholder="Search courses..."
            />
            <i class="fas fa-search absolute left-3 top-3 text-gray-400"></i>
          </div>
        </div>

        <!-- Semester -->
        <div>
          <label class="form-label" for="semester">Semester</label>
          <select
            id="semester"
            v-model="filters.semester"
            class="form-input"
          >
            <option value="">All Semesters</option>
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
            v-model="filters.year"
            class="form-input"
          >
            <option value="">All Years</option>
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
    </div>

    <!-- Course Grid -->
    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
      <div
        v-for="course in filteredCourses"
        :key="course.id"
        class="bg-white shadow rounded-lg overflow-hidden hover:shadow-lg transition-shadow duration-300"
      >
        <!-- Course Header -->
        <div class="p-6 border-b">
          <div class="flex justify-between items-start">
            <div>
              <h3 class="text-lg font-semibold text-gray-900">
                {{ course.title }}
              </h3>
              <p class="text-sm text-gray-500">{{ course.code }}</p>
            </div>
            <span
              class="px-2 py-1 text-xs font-medium rounded-full"
              :class="getStatusClass(course)"
            >
              {{ course.is_active ? 'Active' : 'Inactive' }}
            </span>
          </div>
        </div>

        <!-- Course Info -->
        <div class="p-6">
          <p class="text-gray-600 text-sm mb-4 line-clamp-2">
            {{ course.description }}
          </p>

          <div class="space-y-2 text-sm text-gray-500">
            <div class="flex items-center">
              <i class="fas fa-user-tie mr-2"></i>
              <span>{{ course.teacher?.username || 'No teacher assigned' }}</span>
            </div>
            <div class="flex items-center">
              <i class="fas fa-users mr-2"></i>
              <span>{{ course.enrolled_students_count }}/{{ course.max_students }} students</span>
            </div>
            <div class="flex items-center">
              <i class="fas fa-calendar mr-2"></i>
              <span>{{ course.semester }} {{ course.year }}</span>
            </div>
          </div>
        </div>

        <!-- Actions -->
        <div class="px-6 py-4 bg-gray-50 border-t flex justify-between items-center">
          <router-link
            :to="{ name: 'CourseDetail', params: { id: course.id }}"
            class="text-primary-600 hover:text-primary-700 font-medium text-sm"
          >
            View Details
          </router-link>
          
          <button
            v-if="isStudent"
            @click="handleEnrollment(course)"
            :disabled="loading === course.id"
            class="btn-primary text-sm"
            :class="{ 'opacity-75 cursor-not-allowed': loading === course.id }"
          >
            <i class="fas" :class="[isEnrolled(course.id) ? 'fa-times' : 'fa-plus', 'mr-1']"></i>
            {{ isEnrolled(course.id) ? 'Drop Course' : 'Enroll' }}
          </button>
        </div>
      </div>
    </div>

    <!-- Empty State -->
    <div
      v-if="filteredCourses.length === 0"
      class="text-center py-12"
    >
      <div class="text-gray-400 mb-4">
        <i class="fas fa-book-open text-6xl"></i>
      </div>
      <h3 class="text-lg font-medium text-gray-900">No courses found</h3>
      <p class="text-gray-500">Try adjusting your filters or check back later.</p>
    </div>
  </div>
</template>

<script>
export default {
  name: 'CourseList',

  data() {
    return {
      filters: {
        search: '',
        semester: '',
        year: ''
      },
      loading: null
    }
  },

  computed: {
    isStudent() {
      return this.$store.getters['auth/userRole'] === 'student'
    },

    isTeacher() {
      return this.$store.getters['auth/userRole'] === 'teacher'
    },

    courses() {
      return this.$store.getters['courses/allCourses']
    },

    enrollments() {
      return this.$store.getters['courses/enrolledCourses']
    },

    availableYears() {
      const currentYear = new Date().getFullYear()
      return Array.from({ length: 5 }, (_, i) => currentYear + i)
    },

    filteredCourses() {
      return this.courses.filter(course => {
        const matchesSearch = !this.filters.search ||
          course.title.toLowerCase().includes(this.filters.search.toLowerCase()) ||
          course.code.toLowerCase().includes(this.filters.search.toLowerCase())

        const matchesSemester = !this.filters.semester ||
          course.semester === this.filters.semester

        const matchesYear = !this.filters.year ||
          course.year === parseInt(this.filters.year)

        return matchesSearch && matchesSemester && matchesYear
      })
    }
  },

  async created() {
    await this.loadCourses()
  },

  methods: {
    async loadCourses() {
      try {
        await this.$store.dispatch('courses/fetchCourses')
        if (this.isStudent) {
          await this.$store.dispatch('courses/fetchEnrollments')
        }
      } catch (error) {
        console.error('Error loading courses:', error)
      }
    },

    getStatusClass(course) {
      return course.is_active
        ? 'bg-green-100 text-green-800'
        : 'bg-gray-100 text-gray-800'
    },

    isEnrolled(courseId) {
      return this.$store.getters['courses/isEnrolled'](courseId)
    },

    async handleEnrollment(course) {
      this.loading = course.id
      try {
        if (this.isEnrolled(course.id)) {
          const enrollment = this.enrollments.find(e => e.course.id === course.id)
          await this.$store.dispatch('courses/dropCourse', enrollment.id)
        } else {
          await this.$store.dispatch('courses/enrollInCourse', course.id)
        }
      } catch (error) {
        console.error('Error handling enrollment:', error)
      } finally {
        this.loading = null
      }
    }
  }
}
</script>