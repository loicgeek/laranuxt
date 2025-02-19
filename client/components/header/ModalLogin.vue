<template>
  <div class="bg-white dark:bg-gray-800 py-8 px-4 sm:px-10">
    <div class="grid grid-cols-2 gap-3">
      <div>
        <push-button class="w-full justify-center" @click="login('google')">
          <icon
            v-if="loading.google"
            icon="gg:spinner-two"
            class="w-6 h-6 text-indigo-600 animate-spin"
          />
          <Icon
            v-else
            icon="flat-color-icons:google"
            class="w-6 h-6"
          />
        </push-button>
      </div>
      <div>
        <push-button class="w-full justify-center" @click="login('facebook')">
          <icon
            v-if="loading.facebook"
            icon="gg:spinner-two"
            class="w-6 h-6 text-indigo-600 animate-spin"
          />
          <icon
            v-else
            icon="logos:facebook"
            class="w-6 h-6"
          />
        </push-button>
      </div>
    </div>
    <dividing-copy>Or continue with</dividing-copy>
    <label class="mt-6 block text-sm font-medium leading-5 text-gray-700" for="login_email">Email address</label>
    <div class="mt-1 relative rounded-md shadow-sm">
      <div class="absolute inset-y-0 left-0 pl-3 flex items-center pointer-events-none">
        <icon icon="mdi:envelope" class="w-5 h-5 text-gray-400" />
      </div>
      <input
        id="login_email"
        ref="input"
        v-model="email"
        class="form-input appearance-none block w-full px-3 py-2 pl-10 border border-gray-300 rounded-md placeholder-gray-400 focus:outline-none focus:shadow-outline-blue focus:border-blue-300 transition duration-150 ease-in-out sm:text-sm sm:leading-5"
        :readonly="loading.attempt"
        placeholder="email@address.com"
        type="email"
        required="required"
        @keydown.esc="modalOff"
        @keydown.enter="attempt"
      >
    </div>
    <div class="mt-6">
      <span class="block w-full rounded-md shadow-sm">
        <push-button theme="indigo" class="w-full justify-center" @click="attempt">
          Sign in / Register
          <div v-if="loading.attempt" class="absolute inset-y-0 right-0 pr-3 flex items-center pointer-events-none">
            <icon-spinner class="w-5 h-5" primary="text-indigo-700" secondary="text-indigo-200" />
          </div>
        </push-button>
      </span>
    </div>
  </div>
</template>

<script lang="ts" setup>
interface OauthResult {
  token: string
  user: models.User
  provider: string
  error?: string
}

const emit = defineEmits(['off'])
const { $axios, $toast, $auth, app } = useContext()
const email = ref('')
const loading = reactive({
  attempt: false,
  google: false,
} as Record<string, boolean>)
onMounted(() => { if (process.browser) messageHandler(true) })
onBeforeUnmount(() => { if (process.browser) messageHandler(false) })

async function attempt (): Promise<void> {
  loading.attempt = true
  try {
    await $axios.post('/attempt', { email: email.value })
  } catch (e) {
    loading.attempt = false
    return
  }
  $toast.show({
    type: 'success',
    title: 'Login E-mail Sent',
    message: `Login link sent to <b>${email.value}</b>`,
    timeout: 5,
  })
  email.value = ''
  loading.attempt = false
  emit('off')
}

function messageHandler (add: boolean): void {
  if (add) return window.addEventListener('message', handleMessage)
  return window.removeEventListener('message', handleMessage)
}

function handleMessage (event: { data: OauthResult }): void {
  if (event.data.user && event.data.token)
    oauthComplete(event.data)
  if (event.data.error)
    $toast.show({ type: 'danger', message: event.data.error })
}

function login (provider: 'facebook'|'google'): void {
  loading[provider] = true
  const width = 640
  const height = 660
  const left = window.screen.width / 2 - (width / 2)
  const top = window.screen.height / 2 - (height / 2)
  const win = window.open(`${$axios.defaults.baseURL}/redirect/${provider}`, 'Log In',
    `toolbar=no, location=no, directories=no, status=no, menubar=no, scollbars=no,
      resizable=no, copyhistory=no, width=${width},height=${height},top=${top},left=${left}`)
  const interval = setInterval(() => {
    if (win === null || win.closed) {
      clearInterval(interval)
      loading[provider] = false
    }
  }, 200)
}

const oauthComplete = async (result: OauthResult): Promise<void> => {
  loading[result.provider] = false
  await $auth.setUserToken(result.token)
  $toast.show({ type: 'success', message: 'Login Successful' })
  await app.router?.push('/home')
  emit('off')
}
</script>
