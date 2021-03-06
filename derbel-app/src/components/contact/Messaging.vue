<script lang="ts" setup>
import { useAuthStore } from '@/stores/auth.store';
import axios from '@/utils/axios';
import { computed, nextTick, onMounted, reactive, ref, watch } from 'vue';
import { useWebSockets } from '@/stores/websockets.store';
import type { IMessage } from '@/types/message.type';
import moment, { type Moment } from 'moment';
import IconSendMessage from '../icons/IconSendMessage.vue';

const props = defineProps<{
  contact: any;
}>();

const authStore = useAuthStore();
const sockets = useWebSockets();

const rawMessages = ref<IMessage[]>([]);
const message = reactive({
  text: '',
});
const messagesEl = ref<HTMLElement | null>(null);
const messageTextEl = ref<HTMLElement | null>(null);

interface IMessageGroup {
  messageId: number;
  speakerId: number;
  messages: IMessage[];
}

const messages = computed(() => {
  return rawMessages.value.reduce((messages, message) => {
    const lastGroup = messages[messages.length - 1];
    if (lastGroup && message.speakerId === lastGroup.speakerId) {
      lastGroup.messages.push(message);
    } else {
      messages.push({
        messageId: message.id,
        speakerId: message.speakerId,
        messages: [message],
      });
    }
    return messages;
  }, [] as IMessageGroup[]);
});

onMounted(async () => {
  sockets.websocket?.emit('join-messaging', { contactId: props.contact.id });

  sockets.websocket?.on('receive-message', (msg) => {
    addMessage(msg);
  });

  fetchMessages();
});

watch(
  () => props.contact,
  () => fetchMessages()
);

async function fetchMessages() {
  rawMessages.value = await axios
    .get(`/contacts/${props.contact.id}/messages`)
    .then((res) => res.data);
  scrollMessagesToBottom();
}

function sendMessage() {
  if (!message.text) return;
  const data = { contactId: props.contact.id, message };
  sockets.websocket?.emit('send-message', data, (msg: any) => {
    addMessage(msg);
    message.text = '';
  });
}

function addMessage(msg: IMessage) {
  rawMessages.value.push(msg);
  scrollMessagesToBottom();
}

function scrollMessagesToBottom() {
  nextTick(() => messagesEl.value && scrollToBottom(messagesEl.value));
}

function scrollToBottom(el: HTMLElement) {
  el.scrollTop = el.scrollHeight - el.clientHeight;
}

function formatDate(date: string | Moment) {
  return moment().startOf('day').isSame(moment(date).startOf('day'))
    ? moment(date).format('H:mm A')
    : moment(date).format('MMM DD');
}
</script>

<template>
  <div class="message-groups" ref="messagesEl">
    <div
      v-for="messageGroup of messages"
      :key="messageGroup.messageId"
      class="message-group flex gap gap-1.5"
      :class="{
        'self-end': messageGroup.speakerId === authStore.user?.id,
        mine: messageGroup.speakerId === authStore.user?.id,
      }"
    >
      <div class="message-photo"></div>
      <div class="messages">
        <div v-for="(message, i) of messageGroup.messages" :key="message.id" class="message">
          <template v-if="i === 0">
            <div class="flex justify-between gap-5">
              <small class="message-speaker">{{
                messageGroup.speakerId === authStore.user?.id ? ' You' : message.speaker?.name
              }}</small>
              <small class="message-sent-at ml-auto">{{ formatDate(message.sentAt) }}</small>
            </div>
            <div class="message-text">{{ message.text }}</div>
          </template>
          <div v-else class="flex gap-2.5">
            <div class="message-text">{{ message.text }}</div>
            <small class="message-sent-at ml-auto">{{ formatDate(message.sentAt) }}</small>
          </div>
        </div>
      </div>
    </div>
  </div>
  <form @submit.prevent="sendMessage" class="new-message" @click="messageTextEl?.focus()">
    <input v-model="message.text" placeholder="Type a message here..." ref="messageTextEl" />
    <button type="submit">
      <IconSendMessage />
    </button>
  </form>
</template>

<style scoped>
.message-groups {
  display: flex;
  flex-direction: column;
  gap: 10px;
  flex: 1;
  padding: 10px 10px;
  padding-right: 15px;
  overflow: overlay;
}

.message-groups::-webkit-scrollbar {
  width: 5px;
}

.message-groups::-webkit-scrollbar-thumb {
  background: #46464622;
  border-radius: 5px;
}

.messages {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.message {
  display: flex;
  flex-direction: column;
  border-radius: 15px;
  background-color: #fff;

  gap: 10px;
  min-width: 220px;
  max-width: 680px;
  box-shadow: 0 0 4px rgba(0, 0, 0, 0.1);
  padding: 12px;
}

.message * {
  line-height: 100%;
}

.message-group.mine .message {
  border-top-right-radius: 5px;
  background: #d6e4e0;
  box-shadow: none;
}

.message-group:not(.mine) .message {
  border-top-left-radius: 5px;
}

.message-photo {
  width: 20px;
  height: 20px;
  background-color: #b2bdbd;
  border-radius: 40px;
}

.message-group.mine .message-photo {
  order: 1;
}

.message-speaker {
  font-size: 10px;
}

.message-sent-at {
  font-size: 10px;
  white-space: nowrap;
}

.message-text {
  font-size: 14px;
  line-height: 1.3;
}

.new-message {
  display: flex;
  align-items: center;
  gap: 10px;
  background-color: white;
  border-radius: 20px;
  padding: 10px;
  padding-left: 20px;
  box-shadow: 0 0 4px rgba(0, 0, 0, 0.1);
  transition: box-shadow 200ms;
}

.new-message:focus-within {
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.1);
}

.new-message input {
  background-color: transparent;
  border: none;
  font-size: 14px;
  padding: 0;
  outline: none;
}

.new-message input::placeholder {
  color: #464646;
  font-size: 14px;
}

.new-message button {
  border: none;
  background-color: #252525;
  color: white;
  padding: 10px;
  font-size: 20px;
  border-radius: 50px;
}
</style>
