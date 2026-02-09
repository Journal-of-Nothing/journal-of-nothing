<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import { useRouter } from 'vue-router'
import { useAuth } from '../stores/auth'
import { fetchUserReviewOpinions, fetchUserSubmissions, updateUserPermissions } from '../services/supabaseApi'

const router = useRouter()
const { user, profile, signOut } = useAuth()
const { t } = useI18n()
const authUser = computed(() => user.value)
const adminTargetId = ref('')
const adminRole = ref<'author' | 'reviewer' | 'deputy_editor' | 'admin'>('author')
const adminCanSubmit = ref(true)
const adminCanReview = ref(true)
const adminCanComment = ref(true)
const adminMessage = ref('')
const isLoadingLists = ref(true)
const mySubmissions = ref<{ id: string; title: string; status: string; updated_at: string }[]>([])
const myReviews = ref<{ id: string; submission_id: string; status: string; decision: string | null; created_at: string }[]>([])

const isAdmin = computed(() => profile.value?.role === 'admin')

onMounted(async () => {
  if (!authUser.value) return
  const [submissionsRes, reviewsRes] = await Promise.all([
    fetchUserSubmissions(authUser.value.id),
    fetchUserReviewOpinions(authUser.value.id),
  ])

  mySubmissions.value = (submissionsRes.data as typeof mySubmissions.value) ?? []
  myReviews.value = (reviewsRes.data as typeof myReviews.value) ?? []
  isLoadingLists.value = false
})

const handleSignOut = async () => {
  await signOut()
  router.replace('/login')
}

const updatePermissions = async () => {
  adminMessage.value = ''
  if (!adminTargetId.value.trim()) {
    adminMessage.value = t('user.adminMissingId')
    return
  }
  const { error } = await updateUserPermissions({
    id: adminTargetId.value.trim(),
    role: adminRole.value,
    can_submit: adminCanSubmit.value,
    can_review: adminCanReview.value,
    can_comment: adminCanComment.value,
  })
  if (error) {
    adminMessage.value = error.message
    return
  }
  adminMessage.value = t('user.adminUpdated')
}
</script>

<template>
  <section class="mx-auto max-w-2xl space-y-6 rounded-lg border border-slate-200 bg-white p-6">
    <div>
      <h1 class="text-2xl font-semibold text-slate-900">{{ $t('user.title') }}</h1>
      <p class="text-sm text-slate-500">{{ $t('user.subtitle') }}</p>
    </div>

    <div class="rounded-md border border-slate-100 bg-slate-50 p-4">
      <p class="text-sm text-slate-500">{{ $t('user.currentUser') }}</p>
      <p class="mt-1 text-slate-900">
        {{ authUser?.user_metadata?.user_name || authUser?.email || $t('app.anonymous') }}
      </p>
    </div>

    <div class="rounded-md border border-slate-100 bg-slate-50 p-4 text-sm">
      <p class="text-slate-500">{{ $t('user.roleTitle') }}</p>
      <div class="mt-2 space-y-1 text-slate-700">
        <p>{{ $t('user.roleLabel', { role: profile?.role || $t('user.roleUnset') }) }}</p>
        <p>{{ $t('user.canSubmit', { value: profile?.can_submit ? $t('user.valueAllow') : $t('user.valueDeny') }) }}</p>
        <p>{{ $t('user.canReview', { value: profile?.can_review ? $t('user.valueAllow') : $t('user.valueDeny') }) }}</p>
        <p>{{ $t('user.canComment', { value: profile?.can_comment ? $t('user.valueAllow') : $t('user.valueDeny') }) }}</p>
      </div>
    </div>

    <div class="rounded-md border border-slate-100 bg-white p-4 text-sm">
      <p class="text-slate-700">{{ $t('user.mySubmissions') }}</p>
      <ul class="mt-3 space-y-2">
        <template v-if="isLoadingLists">
          <li v-for="index in 2" :key="index" class="flex items-center justify-between">
            <div class="skeleton-text h-4 w-32" />
            <div class="skeleton-text h-3 w-14" />
          </li>
        </template>
        <template v-else>
          <li v-for="item in mySubmissions" :key="item.id" class="flex items-center justify-between">
            <router-link class="text-slate-700 hover:text-slate-900" :to="`/submissions/${item.id}`">
              {{ item.title }}
            </router-link>
            <span class="text-xs text-slate-500">{{ item.status }}</span>
          </li>
          <li v-if="!mySubmissions.length" class="text-xs text-slate-400">{{ $t('user.emptySubmissions') }}</li>
        </template>
      </ul>
    </div>

    <div class="rounded-md border border-slate-100 bg-white p-4 text-sm">
      <p class="text-slate-700">{{ $t('user.myReviews') }}</p>
      <ul class="mt-3 space-y-2">
        <template v-if="isLoadingLists">
          <li v-for="index in 2" :key="index" class="flex items-center justify-between">
            <div class="skeleton-text h-4 w-32" />
            <div class="skeleton-text h-3 w-14" />
          </li>
        </template>
        <template v-else>
          <li v-for="item in myReviews" :key="item.id" class="flex items-center justify-between">
            <router-link class="text-slate-700 hover:text-slate-900" :to="`/submissions/${item.submission_id}/review-opinions`">
              {{ $t('review.defaultTitle') }} {{ item.id.slice(0, 6) }}
            </router-link>
            <span class="text-xs text-slate-500">{{ item.status }}</span>
          </li>
          <li v-if="!myReviews.length" class="text-xs text-slate-400">{{ $t('user.emptyReviews') }}</li>
        </template>
      </ul>
    </div>

    <div v-if="isAdmin" class="rounded-md border border-slate-100 bg-white p-4 text-sm">
      <p class="text-slate-700">{{ $t('user.adminTitle') }}</p>
      <div class="mt-3 space-y-3">
        <input
          v-model="adminTargetId"
          class="w-full rounded-md border border-slate-200 px-3 py-2 text-sm"
          :placeholder="$t('user.adminUserId')"
        />
        <select v-model="adminRole" class="w-full rounded-md border border-slate-200 px-3 py-2 text-sm">
          <option value="author">作者</option>
          <option value="reviewer">审稿人</option>
          <option value="deputy_editor">副主编</option>
          <option value="admin">管理员</option>
        </select>
        <div class="flex items-center gap-4">
          <label class="flex items-center gap-2">
            <input v-model="adminCanSubmit" type="checkbox" /> 投稿
          </label>
          <label class="flex items-center gap-2">
            <input v-model="adminCanReview" type="checkbox" /> 审稿
          </label>
          <label class="flex items-center gap-2">
            <input v-model="adminCanComment" type="checkbox" /> 评论
          </label>
        </div>
        <div class="flex items-center justify-between">
          <span class="text-xs text-slate-500">{{ adminMessage }}</span>
          <button
            class="rounded-md bg-slate-900 px-3 py-2 text-xs font-semibold text-white"
            @click="updatePermissions"
          >
            {{ $t('user.adminUpdate') }}
          </button>
        </div>
      </div>
    </div>

    <button
      class="rounded-md border border-slate-200 px-4 py-2 text-sm font-semibold text-slate-700 hover:border-slate-300"
      @click="handleSignOut"
    >
      {{ $t('user.signOut') }}
    </button>
  </section>
</template>
