<template>
  <v-container class="">
    <v-responsive class="align-center fill-height text-center">
      <h1 class="my-2">Telegram Group Joiner</h1>

      <v-row justify="center" class="text-left">
        <v-col cols="12" md="8" lg="6" xl="4">
          <p class="mt-2">
            Use this tool to batch add a Telegram account (via its phone number) to multiple Telegram groups.
            Works for <u>public</u> and <u>private</u> groups; private groups require an invite link and not just the
            <code>t.me/c/1234</code> link.
          </p>
          <p class="mt-2">We advise you <strong>not to use your personal account</strong> for automations
            as telegram may block it (unlikely but possible).</p>
        </v-col>
      </v-row>

      <!-- authenticate -->
      <v-form>
        <v-row justify="center">
          <v-col cols="12" md="8" lg="6" xl="4">
            <h2 class="my-1">Step 1 - Authenticate</h2>
            <v-row justify="center" class="text-left mb-1">
              <v-col>
                <p class="my-2">Get telegram API keys from <a href="https://my.telegram.org/auth">my.telegram.org</a>
                  (see <a href="https://docs.telethon.dev/en/stable/basic/signing-in.html#signing-in">instructions</a>).
                  These keys are private, but they can be reused to add any account (phone number) to groups.
                </p>
              </v-col>
            </v-row>
            <!-- step 0 -->
            <v-text-field v-model="apiId" label="API ID" outlined placeholder="Enter API ID"
              :prepend-inner-icon="authFlowStep >= 1 ? 'mdi-check' : ''" />
            <v-text-field v-model="apiHash" label="API HASH" outlined placeholder="Enter API Hash"
              :prepend-inner-icon="authFlowStep >= 1 ? 'mdi-check' : ''" type="password" />
            <p class="my-2"
              v-if="authenticationState == 'authorizationStateWaitPhoneNumber' || authenticationState == 'authorizationStateWaitCode'">
              Expect to receive a code either via SMS or the telegram app with the code.</p>
            <p v-if="phoneNumber">Using phone number <strong>{{ phoneNumber }}</strong></p>
            <!-- step 1 -->
            <v-text-field v-if="authenticationState == 'authorizationStateWaitPhoneNumber'" v-model="phoneNumber"
              label="Phone number" outlined placeholder="Enter phone number from the account to join groups"
              :prepend-inner-icon="authFlowStep >= 2 ? 'mdi-check' : ''" @keypress.enter="nextStep()" />
            <!-- step 2 -->
            <v-text-field v-if="authenticationState == 'authorizationStateWaitCode'" v-model="confirmationCode"
              label="Confirmation code" outlined placeholder="Enter confirmation code"
              :prepend-inner-icon="authFlowStep >= 3 ? 'mdi-check' : ''" @keypress.enter="nextStep()" />
            <!-- step 3 -->
            <v-text-field v-if="authenticationState == 'authorizationStateWaitPassword'" v-model="password"
              label="password" type="password" outlined placeholder="Enter password"
              :prepend-inner-icon="authFlowStep >= 4 ? 'mdi-check' : ''" @keypress.enter="nextStep()" />

            <v-row class="text-left">
              <v-col cols="12">
                <v-alert v-if="errorMessage.length" class="my-4" type="error" title="something is wrong" variant="tonal">
                  {{ errorMessage }}
                </v-alert>
              </v-col>
            </v-row>
          </v-col>
        </v-row>

        <v-row justify="center" class="my-4">
          <v-btn color="primary" depressed large rounded :loading="loading" :disabled="authFlowStep == 4"
            @click="nextStep">
            {{ buttonState }}
          </v-btn>
        </v-row>
        <v-row justify="center" class="my-4">
          <v-btn color="red-accent-2" depressed size="small" rounded :loading="loading" @click="clearStorage">
            reset authentication
          </v-btn>
        </v-row>
        <small>state: <code>{{ authenticationState }}</code></small>
      </v-form>

      <!-- join groups -->
      <v-row v-if="authenticationState == 'authorizationStateReady'" class="ma-4 text-left mb-16">
        <v-divider class="mt-6"></v-divider>
        <v-row justify="center">
          <h2 class="mt-10 mb-4">Step 2 - Join Groups</h2>
        </v-row>
        <v-col cols="12">
          <v-row justify="center">
            <v-col cols="12" md="8" lg="6" xl="4">
              <p class="mt-2">
                For <strong>public groups</strong>, add the group's link <code>https://t.me/channel1</code> to the text
                area, <strong>1 per line</strong>.
              </p>
              <p class="mt-2">
                For <strong>private groups</strong>, add your group invites (eg:
                <code>https://t.me/joinchat/randomString</code>).
                To make the process faster, include the group id after <code>,</code> eg:
                <code>https://t.me/+randomString,123456789</code>.
              </p>
              <p class="mt-2"><strong>NB:</strong> some invite links may direct you to a bot that you then need to
                manually interact with to
                gain access to channels.</p>
              <v-checkbox v-model="muteJoinedGroups" label="mute joined chats" density="comfortable"
                color="primary"></v-checkbox>
            </v-col>
          </v-row>

          <v-row justify="center">
            <!-- <v-col cols="12" class="justify-center"> -->
            <v-btn class="mb-2" color="primary" depressed large rounded :loading="loading" @click="joinGroups">
              {{ 'Join ' + groups.length + ' Group' + (groups.length == 1 ? '' : 's') }}
            </v-btn>
            <v-progress-linear v-if="processedInvites > 0 && processedInvites != groups.length" v-model="processedInvites"
              :max="groups.length" color="light-blue" height="20" striped class="my-2">
              <template v-slot:default="{ value }">
                <strong>{{ processedInvites }} / {{ groups.length }}</strong>
              </template></v-progress-linear>
          </v-row>
          <v-row justify="center">
            <v-col cols="4">
              <v-alert v-if="processedInvites > 0 && processedInvites == groups.length" type="success"
                :title="`${processedInvites}/${groups.length} groups joined`"></v-alert>
            </v-col>
          </v-row>

          <v-row justify="center">
            <v-col cols="8" v-if="chatExceptions.size">
              <v-alert class="my-4" type="error" title="custom status messages detected" variant="tonal">
                To understand custom status messages see the <a
                  href="https://core.telegram.org/method/messages.importChatInvite">API documentation</a>.
                <br>Detected: {{ chatExceptions }}
              </v-alert>
            </v-col>
            <v-col cols="8" v-if="sleepSeconds > 0">
              <v-alert class="my-4" type="warning" title="rate-limit hit, take a break and stretch 🧘" variant="tonal">
                Telegram rate limit kicked-in, we're resuming after
                <strong>{{ sleepSeconds }} second{{ sleepSeconds == 1 ? '' : 's' }}</strong>
              </v-alert>
            </v-col>
          </v-row>
        </v-col>

        <v-col cols="12" md="4">
          <v-textarea v-model="groupsInput" label="Group links/invites" outlined placeholder="Enter group links and invites into this text area, 1 per line"
            rows="15" class="mt-2 text-xs" required elevation="8" />
        </v-col>
        <v-col cols="12" md="5">
          <v-card v-if="groups.length" class="mx-auto pb-4" max-width="600" color="blue-lighten-1" variant="outlined"
            elevation="8">
            <v-card-title>{{ groups.length }} group{{ groups.length == 1 ? '' : 's' }} identified</v-card-title>
            <v-list-item v-for="(item, i) in groups" :key="i" :value="item.invite" color="primary">
              <template v-slot:prepend>
                <v-icon :title="item.status" icon="mdi-clock-time-eight-outline" color="info"
                  v-if="item.status == 'pending'"></v-icon>
                <v-icon :title="item.status" icon="mdi-check" color="success" v-if="item.status == 'joined'"></v-icon>
                <v-icon :title="item.status" icon="mdi-account-clock" color="warning"
                  v-if="item.status == 'requested'"></v-icon>
                <v-icon :title="item.status" icon="mdi-link-off" color="error" v-if="item.status == 'expired'"></v-icon>
                <v-icon :title="item.status" icon="mdi-close" color="error"
                  v-if="!['requested', 'pending', 'joined'].includes(item.status)"></v-icon>
              </template>
              <v-list-item-content>
                <v-list-item-title>
                  <a :href="item.invite" target="_blank">{{ item.invite }}</a>
                </v-list-item-title>
                <v-list-item-subtitle style="font-size:16px">
                  <p v-if="item.name?.length" style="color:black">{{ item.name }}</p>
                  <p>status: <strong>{{ item.status }}</strong></p>
                  <p class="mt-1">
                    <v-chip v-if="item.type" size="x-small" variant="tonal"
                      :color="item.type == 'private' ? 'red' : 'green'" class="mr-2" rounded="sm">{{ item.type }}</v-chip>
                    <v-chip v-if="item.id" size="x-small" variant="tonal" color="black" rounded="sm">
                      id={{ item.id }}
                    </v-chip>
                  </p>
                </v-list-item-subtitle>
              </v-list-item-content>
            </v-list-item>
          </v-card>
        </v-col>

        <v-col cols="12" md="3">
          <v-card v-if="invalidGroups.length" class="mx-auto pb-4" max-width="500" color="red-lighten-1"
            variant="outlined" elevation="8">
            <v-card-title>{{ invalidGroups.length }} invalid group{{ invalidGroups.length == 1 ? '' : 's' }}
            </v-card-title>
            <v-list-item v-for="(item, i) in invalidGroups" :key="i" :value="item" color="primary">
              {{ item }}
            </v-list-item>
          </v-card>
        </v-col>
        <v-row justify="center my-2" v-if="groups.length">
          <v-btn class="" prepend-icon="mdi-content-copy" color="blue-grey" depressed large rounded @click="shareInvites">
            Copy valid links
          </v-btn>
          <v-snackbar v-model="copiedSnackbar">
            copied to clipboard
          </v-snackbar>
        </v-row>
      </v-row>
      <v-divider class="mt-6"></v-divider>
    </v-responsive>
  </v-container>
</template>

<script setup>
import { ref, watch, computed } from 'vue'

const params = new Proxy(new URLSearchParams(window.location.search), {
  get: (searchParams, prop) => searchParams.get(prop),
});

const apiId = ref('')
const apiHash = ref('')
const phoneNumber = ref('')
const confirmationCode = ref('')
const password = ref('')
const authenticationState = ref('Not started')
const buttonState = ref('start auth')
const authFlowStep = ref(0)
const muteJoinedGroups = ref(true)
const groupsInput = ref(params.links?.replaceAll(";", "\n") || ``)
const loading = ref(false)
const errorMessage = ref("")

const joinedChatIds = ref([])
const chatExceptions = ref(new Map())
const sleepSeconds = ref(0)
const processedInvites = ref(0)
const copiedSnackbar = ref(false)
const loadedGroups = ref(new Map()) // link -> props

import TdClient from '@dibgram/tdweb/dist/tdweb';

// hard reset on the state
new TdClient({}).send({ '@type': 'close' });
let client = new TdClient({});


const groups = computed(() => {
  return groupsInput.value.split('\n').map((line) => {
    const [invite, id] = line.split(',')
    let status = 'pending';
    if (joinedChatIds.value?.includes(id) || joinedChatIds.value?.includes(`-100${invite}`)) {
      status = 'joined'
    } else if (chatExceptions.value.has(id)) {
      status = chatExceptions.value.get(id)
    }
    return {
      invite,
      id,
      status,
      type: (invite.includes("t.me/+") || invite.includes('joinchat')) ? 'private' : 'public',
      ...(loadedGroups.value.has(invite) ? loadedGroups.value.get(invite) : {})
    }
  }).filter((group) => group.invite.length && group.invite.match(/(?:t|telegram)\.(?:me|dog)\/(joinchat\/|\+)?([\w-]+)/))
    .filter((group) => !group.invite.includes("t.me/c/"))
    .filter((group, i, arr) => arr.findIndex((g) => g.invite == group.invite) == i)
})

const invalidGroups = computed(() => {
  return groupsInput.value.split('\n').map((line) => {
    const [invite, id] = line.split(',')
    if (groups.value.filter((g) => g?.invite == invite).length) {
      return
    }
    return line
  }).filter((line) => line?.length)
})

watch(authenticationState, (newVal, oldVal) => {
  console.log('authenticationState changed from', oldVal, 'to', newVal)
  if (newVal == 'authorizationStateWaitTdlibParameters') {
    buttonState.value = 'start auth'
    authFlowStep.value = 0
  }
  if (newVal == 'authorizationStateWaitPhoneNumber') {
    buttonState.value = 'check phone number'
    authFlowStep.value = 1
  }
  if (newVal == 'authorizationStateWaitCode') {
    buttonState.value = 'verify code'
    authFlowStep.value = 2
  }
  if (newVal == 'authorizationStateWaitPassword') {
    buttonState.value = 'verify password'
    authFlowStep.value = 3
  }
  if (newVal == 'authorizationStateReady') {
    buttonState.value = 'ready'
    authFlowStep.value = 4
  }
})

watch(groups, (newVal, oldVal) => {
  if (newVal.length != oldVal.length) {
    processedInvites.value = 0
  }
})

client.onUpdate = update => {
  // console.log('receive update', update);
  if (update["@type"] == "updateAuthorizationState") {
    authenticationState.value = update.authorization_state["@type"];
  }
};

const upsert = (theMap, key, newProperties) => {
  // Check if the key exists in the map
  if (theMap.has(key)) {
    // If it exists, update its properties
    theMap.set(key, { ...theMap.get(key), ...newProperties });
  } else {
    // If it doesn't exist, create a new object with the desired properties
    theMap.set(key, { ...newProperties });
  }
}

const nextStep = () => {
  errorMessage.value = ""
  if (authenticationState.value == 'authorizationStateWaitTdlibParameters') {
    startAuth();
  }
  if (authenticationState.value == 'authorizationStateWaitPhoneNumber') {
    sendPhone();
  }
  if (authenticationState.value == 'authorizationStateWaitCode') {
    checkCode();
  }
  if (authenticationState.value == 'authorizationStateWaitPassword') {
    checkPassword();
  }
}

const startAuth = () => {
  loading.value = true;

  client.send({
    '@type': 'setTdlibParameters',
    use_test_dc: false,
    api_id: Number.parseInt(apiId.value),
    api_hash: apiHash.value,
    system_language_code: 'en',
    device_model: "Unknown",
    system_version: "UNIX",
    application_version: '1.0',
    use_secret_chats: false,
    use_message_database: true,
    use_file_database: false,
    database_directory: '/db',
    files_directory: '/f',
    use_quick_ack: true,
    chat_folder_count_max: 1000,
    // optimize_database_storage: true,
  }).then(() => {
    loading.value = false;
  }).catch((err) => {
    console.log('setTdlibParameters error', err)
    loading.value = false;
    errorMessage.value = err.message
  })
}

const sendPhone = () => {
  loading.value = true;
  client.send({
    '@type': 'setAuthenticationPhoneNumber',
    phone_number: phoneNumber.value,
  }).then(() => {
    loading.value = false;
  }).catch((err) => {
    console.log('setAuthenticationPhoneNumber error', err)
    loading.value = false;
    errorMessage.value = err.message
  })
}

const checkCode = () => {
  loading.value = true;
  client.send({
    '@type': 'checkAuthenticationCode',
    code: confirmationCode.value,
  }).then(() => {
    loading.value = false;
  }).catch((err) => {
    console.log('setAuthenticationPhoneNumber error', err)
    loading.value = false;
    errorMessage.value = err.message
  })
}

const checkPassword = () => {
  loading.value = true;
  client.send({
    '@type': 'checkAuthenticationPassword',
    password: password.value,
  }).then(() => {
    loading.value = false;
  }).catch((err) => {
    console.log('setAuthenticationPhoneNumber error', err)
    loading.value = false;
    errorMessage.value = err.message
  })
}

const clearStorage = () => {
  localStorage.clear();
  location.reload();
  indexedDB.deleteDatabase('tdlib');
  indexedDB.deleteDatabase('/tdlib/dbfs');
}

const shareInvites = () => {
  const url = new URL(window.location.href);
  // url.searchParams.set('links', groups.map((g) => `${g.invite}` + g.id?`,${g.invite}`:'').join('\n'))
  navigator.clipboard.writeText(groups.value.map((g) => `${g.invite}` + (g.id ? `,${g.id}` : '')).join('\n'))
  copiedSnackbar.value = true
}

const loadAllChats = () => {
  return new Promise((resolve, reject) => {
    // call loadChats until 404 is returned
    client.send({
      '@type': 'loadChats',
      limit: 1000,
    }).then((res) => {
      console.log('loading more chats...', res)
      resolve(loadAllChats())
    }).catch((err) => {
      // 404 when completed
      console.log('loadChats done', err)
      client.send({
        '@type': 'getChats',
        limit: 50000,
      }).then((res) => {
        joinedChatIds.value = [...joinedChatIds.value, ...res.chat_ids]
        resolve()
      }).catch((err) => {
        console.log('getChats error', err)
        reject()
      })
    })
  })
}

/**
 * Join groups
 * loads all groups so they can be checked for belonging by ID
 * then goes through each group and joins it, depending whether its private or public
 */
const joinGroups = () => {
  loading.value = true;

  loadAllChats().then(async () => {
    // rest the status for all loadedGroups entries
    loadedGroups.value.forEach((_, group) => {
      upsert(loadedGroups.value, group, { status: 'pending' })
    })
    chatExceptions.value.clear()
    for (processedInvites.value = 0; processedInvites.value < groups.value.length; processedInvites.value++) {
      let group = groups.value[processedInvites.value];
      if (group.status == 'joined') {
        console.log(`already joined ${group.invite}`)
        continue
      }
      if (group.type == 'private') {
        await joinPrivateGroup(group)
      } else {
        await joinPublicGroup(group)
      }
    }
    loading.value = false;
    loadAllChats()
  })
}

/**
 * joins a private group by invite link:
 *   - checks if it's already joined - skip if so
 *   - calls checkChatInviteLink to get channel title
 *   - calls joinChatByInviteLink and handles the response
 * likely leads to flood error and will wait accordingly
 */
const joinPrivateGroup = async (group) => {
  await client.send({
    '@type': 'checkChatInviteLink',
    invite_link: group.invite,
  }).then((res) => {
    console.log('checkChatInviteLink', res)
    upsert(loadedGroups.value, group.invite, { name: res.title })
  })

  try {
    await client.send({
      '@type': 'joinChatByInviteLink',
      invite_link: group.invite,
    })
  } catch (res) { // this method call fails by default... so we catch =
    console.log('joinChatByInviteLink', res)
    await handleRateLimit(res)
    console.log(res.message)
    if (res.message == "INVITE_REQUEST_SENT") {
      upsert(loadedGroups.value, group.invite, { status: 'requested' })
    } else if (res.message == "USER_ALREADY_PARTICIPANT") {
      upsert(loadedGroups.value, group.invite, { status: 'joined' })
    } else if (res.message == "INVITE_HASH_EXPIRED") {
      upsert(loadedGroups.value, group.invite, { status: 'expired' })
    } else {
      chatExceptions.value.set(group.id, res.message)
    }
    if (muteJoinedGroups.value) {
      muteJoinedGroup(group)
    }
  }
}

const joinPublicGroup = async (group) => {
  console.log('joinPublicGroup', group)
  // call searchPublicChat and then call joinChat
  if (!group.id?.length) {
    await client.send({
      '@type': 'searchPublicChat',
      username: group.invite.split('/').pop(),
    }).then((res) => {
      console.log('searchPublicChat', res)
      upsert(loadedGroups.value, group.invite, { name: res.title })
      group.id = res.id;
      upsert(loadedGroups.value, group.invite, { id: res.id })
    })
  }

  if (joinedChatIds.value.includes(group.id)) {
    console.log(`already joined ${group.invite}`)
    upsert(loadedGroups.value, group.invite, { status: "joined" })
    return
  }

  try {
    console.log(`chat_id: ${group.id}`)
    await client.send({
      '@type': 'joinChat',
      chat_id: group.id,
    }).then(res => {
      console.log('joinChat', res)
      upsert(loadedGroups.value, group.invite, { status: 'joined' })
      joinedChatIds.value.push(group.id)
      if (muteJoinedGroups.value) {
        muteJoinedGroup(group)
      }
    })
  } catch (res) { // this method call fails by default... so we catch
    console.log('joinChat', res)
    await handleRateLimit(res)
    console.log(res.message)
  }
}

const handleRateLimit = async (res) => {
  if (res.code == 429 || res?.message?.includes('Too Many Requests: retry after')) {
    console.log(`detected rate limit:`)
    console.log(res)
    // flood wait error, need to sleep for XYZ seconds
    sleepSeconds.value = Number.parseInt(res.message.match(/\d+/)?.[0]);
    const countDown = setInterval(() => {
      sleepSeconds.value--;
      if (sleepSeconds.value <= 0) {
        clearInterval(countDown)
      }
    }, 1000);
    console.log(`sleeping for ${sleepSeconds.value} seconds`)
    await new Promise((resolve) => setTimeout(resolve, 1000 * sleepSeconds.value));
    processedInvites.value--; //go back so it can be retried
  }
}

const muteJoinedGroup = async (group) => {
  console.log(`muting ${group.id}`)
  await client.send({
    '@type': 'setChatNotificationSettings',
    chat_id: group.id,
    notification_settings: {
      mute_for: 1000000000,
    },
  }).then((res) => {
    console.log('setChatNotificationSettings', res)
  }).catch((err) => {
    console.log('setChatNotificationSettings error', err)
  })
}

</script>

<style>
a {
  color: #2196f3;
}

a:hover {
  color: #1976d2;
}

code {
  background-color: #eee;
  padding: 0.2rem;
  border-radius: 0.2rem;
  /* color should be blue */
  color: #2196f3;
  font-size: 0.8rem;
}

textarea {
  font-size: 13px;
}
</style>
