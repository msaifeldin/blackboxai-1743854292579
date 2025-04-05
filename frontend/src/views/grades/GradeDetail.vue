<template>
  <div v-if="grade">
    <!-- Header -->
    <div class="bg-white shadow rounded-lg p-6 mb-6">
      <div class="flex justify-between items-start">
        <div>
          <h1 class="text-2xl font-bold text-gray-900">
            {{ grade.course.title }}
          </h1>
          <p class="text-gray-500">{{ grade.course.code }}</p>
          <div class="mt-2 flex items-center">
            <span class="text-sm text-gray-500">Student:</span>
            <span class="ml-2 text-sm font-medium text-gray-900">
              {{ grade.student.username }}
            </span>
          </div>
        </div>
        <div class="text-right">
          <div class="text-sm text-gray-500">Final Grade</div>
          <span
            class="mt-1 inline-block px-3 py-1 text-lg font-medium rounded-full"
            :class="getGradeClass(grade.final_grade)"
          >
            {{ formatGrade(grade.final_grade) }}
          </span>
        </div>
      </div>
    </div>

    <!-- Assignment Grades -->
    <div class="bg-white shadow rounded-lg overflow-hidden mb-6">
      <div class="px-6 py-4 border-b">
        <h2 class="text-lg font-medium text-gray-900">Assignment Grades</h2>
      </div>

      <div class="divide-y divide-gray-200">
        <div
          v-for="assignmentGrade in grade.assignment_grades"
          :key="assignmentGrade.id"
          class="p-6"
        >
          <div class="flex justify-between items-start">
            <div>
              <h3 class="text-base font-medium text-gray-900">
                {{ assignmentGrade.assignment.title }}
              </h3>
              <p class="mt-1 text-sm text-gray-500">
                {{ assignmentGrade.assignment.description }}
              </p>
              <div class="mt-2 text-sm text-gray-500">
                Due: {{ formatDate(assignmentGrade.assignment.due_date) }}
              </div>
            </div>

            <div class="text-right">
              <div class="flex items-center space-x-4">
                <div v-if="isTeacher">
                  <button
                    @click="editAssignmentGrade(assignmentGrade)"
                    class="text-primary-600 hover:text-primary-700"
                  >
                    <i class="fas fa-edit"></i>
                  </button>
                </div>
                <div>
                  <div class="text-sm text-gray-500">Points</div>
                  <div class="text-lg font-medium text-gray-900">
                    {{ assignmentGrade.points }}/{{ assignmentGrade.assignment.total_points }}
                  </div>
                  <div class="text-sm text-gray-500">
                    {{ calculatePercentage(assignmentGrade.points, assignmentGrade.assignment.total_points) }}%
                  </div>
                </div>
              </div>
            </div>
          </div>

          <!-- Feedback -->
          <div v-if="assignmentGrade.feedback" class="mt-4">
            <div class="text-sm font-medium text-gray-700">Feedback</div>
            <p class="mt-1 text-sm text-gray-600">
              {{ assignmentGrade.feedback }}
            </p>
          </div>
        </div>

        <!-- Empty State -->
        <div v-if="grade.assignment_grades.length === 0" class="p-6 text-center text-gray-500">
          No assignments graded yet
        </div>
      </div>
    </div>

    <!-- Comments -->
    <div class="bg-white shadow rounded-lg overflow-hidden">
      <div class="px-6 py-4 border-b">
        <h2 class="text-lg font-medium text-gray-900">Comments</h2>
      </div>

      <div class="p-6">
        <!-- Comment List -->
        <div class="space-y-6">
          <div
            v-for="comment in grade.comments"
            :key="comment.id"
            class="flex space-x-3"
          >
            <div class="flex-shrink-0">
              <div class="h-10 w-10 rounded-full bg-primary-100 flex items-center justify-center">
                <span class="text-primary-600 font-medium">
                  {{ comment.author.username.charAt(0).toUpperCase() }}
                </span>
              </div>
            </div>
            <div class="flex-grow">
              <div class="text-sm">
                <span class="font-medium text-gray-900">
                  {{ comment.author.username }}
                </span>
                <span class="text-gray-500 ml-2">
                  {{ formatDate(comment.created_at) }}
                </span>
              </div>
              <div class="mt-1 text-sm text-gray-700">
                {{ comment.comment }}
              </div>
            </div>
          </div>
        </div>

        <!-- Add Comment -->
        <div class="mt-6">
          <form @submit.prevent="addComment" class="space-y-4">
            <div>
              <label class="form-label" for="comment">Add a comment</label>
              <textarea
                id="comment"
                v-model="newComment"
                rows="3"
                class="form-input"
                placeholder="Write your comment here..."
              ></textarea>
            </div>
            <div class="flex justify-end">
              <button
                type="submit"
                :disabled="!newComment.trim() || commentLoading"
                class="btn-primary"
              >
                {{ commentLoading ? 'Posting...' : 'Post Comment' }}
              </button>
            </div>
          </form>
        </div>
      </div>
    </div>

    <!-- Edit Grade Modal -->
    <div
      v-if="showEditModal"
      class="fixed inset-0 bg-gray-500 bg-opacity-75 flex items-center justify-center"
    >
      <div class="bg-white rounded-lg shadow-xl max-w-lg w-full mx-4">
        <div class="px-6 py-4 border-b">
          <h3 class="text-lg font-medium text-gray-900">
            Edit Grade - {{ selectedAssignment?.assignment.title }}
          </h3>
        </div>

        <form @submit.prevent="updateAssignmentGrade" class="p-6">
          <div class="space-y-4">
            <div>
              <label class="form-label" for="points">Points</label>
              <input
                id="points"
                v-model.number="editForm.points"
                type="number"
                min="0"
                :max="selectedAssignment?.assignment.total_points"
                step="0.1"
                class="form-input"
                required
              />
              <p class="mt-1 text-sm text-gray-500">
                Maximum points: {{ selectedAssignment?.assignment.total_points }}
              </p>
            </div>

            <div>
              <label class="form-label" for="feedback">Feedback</label>
              <textarea
                id="feedback"
                v-model="editForm.feedback"
                rows="3"
                class="form-input"
              ></textarea>
            </div>
          </div>

          <div class="mt-6 flex justify-end space-x-3">
            <button
              type="button"
              @click="showEditModal = false"
              class="btn-secondary"
            >
              Cancel
            </button>
            <button
              type="submit"
              :disabled="updateLoading"
              class="btn-primary"
            >
              {{ updateLoading ? 'Saving...' : 'Save Changes' }}
            </button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'GradeDetail',

  props: {
    id: {
      type: String,
      required: true
    }
  },

  data() {
    return {
      showEditModal: false,
      selectedAssignment: null,
      editForm: {
        points: 0,
        feedback: ''
      },
      newComment: '',
      commentLoading: false,
      updateLoading: false
    }
  },

  computed: {
    grade() {
      return this.$store.getters['grades/gradeById'](this.id)
    },

    isTeacher() {
      return this.$store.getters['auth/userRole'] === 'teacher'
    }
  },

  async created() {
    await this.loadGrade()
  },

  methods: {
    async loadGrade() {
      try {
        await this.$store.dispatch('grades/fetchGrade', this.id)
      } catch (error) {
        console.error('Error loading grade:', error)
      }
    },

    formatDate(date) {
      return new Date(date).toLocaleString()
    },

    formatGrade(grade) {
      return grade ? `${grade.toFixed(1)}%` : 'N/A'
    },

    calculatePercentage(points, total) {
      return ((points / total) * 100).toFixed(1)
    },

    getGradeClass(grade) {
      if (!grade) return 'bg-gray-100 text-gray-800'
      
      if (grade >= 90) return 'bg-green-100 text-green-800'
      if (grade >= 80) return 'bg-blue-100 text-blue-800'
      if (grade >= 70) return 'bg-yellow-100 text-yellow-800'
      if (grade >= 60) return 'bg-orange-100 text-orange-800'
      return 'bg-red-100 text-red-800'
    },

    editAssignmentGrade(assignment) {
      this.selectedAssignment = assignment
      this.editForm.points = assignment.points
      this.editForm.feedback = assignment.feedback
      this.showEditModal = true
    },

    async updateAssignmentGrade() {
      this.updateLoading = true
      try {
        await this.$store.dispatch('grades/submitAssignmentGrade', {
          assignmentId: this.selectedAssignment.id,
          gradeData: this.editForm
        })
        this.showEditModal = false
        await this.loadGrade()
      } catch (error) {
        console.error('Error updating grade:', error)
      } finally {
        this.updateLoading = false
      }
    },

    async addComment() {
      if (!this.newComment.trim()) return

      this.commentLoading = true
      try {
        await this.$store.dispatch('grades/addComment', {
          gradeId: this.id,
          comment: this.newComment
        })
        this.newComment = ''
        await this.loadGrade()
      } catch (error) {
        console.error('Error adding comment:', error)
      } finally {
        this.commentLoading = false
      }
    }
  }
}
</script>