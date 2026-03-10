<script>
import { mapGetters } from 'vuex';
import { useUISettings } from 'dashboard/composables/useUISettings';
import { useAccount } from 'dashboard/composables/useAccount';
import ConversationBox from '../../../components/widgets/conversation/ConversationBox.vue';
import { BUS_EVENTS } from 'shared/constants/busEvents';
import CmdBarConversationSnooze from 'dashboard/routes/dashboard/commands/CmdBarConversationSnooze.vue';
import { emitter } from 'shared/helpers/mitt';

export default {
  components: {
    ConversationBox,
    CmdBarConversationSnooze,
  },
  beforeRouteLeave(to, from, next) {
    if (this.conversationId) {
      this.$store.dispatch('clearSelectedState');
    }
    next();
  },
  props: {
    inboxId: {
      type: [String, Number],
      default: 0,
    },
    conversationId: {
      type: [String, Number],
      default: 0,
    },
  },
  setup() {
    const { uiSettings } = useUISettings();
    const { accountId } = useAccount();
    return { uiSettings, accountId };
  },
  computed: {
    ...mapGetters({
      chatList: 'getAllConversations',
      currentChat: 'getSelectedChat',
    }),
  },
  watch: {
    conversationId() {
      this.fetchConversationIfUnavailable();
    },
  },
  created() {
    if (!this.conversationId) {
      this.$store.dispatch('clearSelectedState');
    }
  },
  mounted() {
    this.$store.dispatch('agents/get');
    this.$store.dispatch('portals/index');
    this.initialize();
    this.fetchConversationIfUnavailable();
    this.$watch('$store.state.route', () => this.initialize());
    this.$watch('chatList.length', () => {
      this.setActiveChat();
    });
  },
  methods: {
    onConversationLoad() {
      this.fetchConversationIfUnavailable();
    },
    initialize() {
      this.$store.dispatch('setActiveInbox', this.inboxId);
      this.setActiveChat();
    },
    fetchConversationIfUnavailable() {
      if (!this.conversationId) return;
      const chat = this.findConversation();
      if (!chat) {
        this.$store.dispatch('getConversation', this.conversationId);
      }
    },
    findConversation() {
      const conversationId = parseInt(this.conversationId, 10);
      const [chat] = this.chatList.filter(c => c.id === conversationId);
      return chat;
    },
    setActiveChat() {
      if (this.conversationId) {
        const selectedConversation = this.findConversation();
        if (
          !selectedConversation ||
          selectedConversation.id === this.currentChat.id
        ) {
          return;
        }
        const { messageId } = this.$route.query;
        this.$store
          .dispatch('setActiveChat', {
            data: selectedConversation,
            after: messageId,
          })
          .then(() => {
            emitter.emit(BUS_EVENTS.SCROLL_TO_MESSAGE, { messageId });
          });
      } else {
        this.$store.dispatch('clearSelectedState');
      }
    },
  },
};
</script>

<template>
  <section class="flex w-full h-full min-w-0">
    <ConversationBox
      v-if="conversationId"
      :inbox-id="inboxId"
      :is-on-expanded-layout="true"
      :is-contact-panel-open="false"
    />
    <CmdBarConversationSnooze />
  </section>
</template>
