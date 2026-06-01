<template>
  <div
    class="mx-auto flex h-full w-full flex-col gap-6 p-4 md:max-w-lg lg:max-w-xl justify-center"
  >
    <!-- Header -->
    <header class="flex flex-col gap-4 items-center">
      <img class="block w-full max-w-[180px] mb-10" src="/logo.png" />

      <p class="text-sm font-medium w-full">
        TCVS (Twitch Chat Voice Synthesis) is a simple Text-To-Speech
        implementation for Twitch.
      </p>
      <p class="text-sm font-medium w-full">
        Enter the name of any Twitch channel and press Start to
        have messages from chat read aloud in your browser.
      </p>
    </header>

    <!-- Channel -->
    <section class="flex flex-col gap-6">
      <div>
        <label
          for="channel"
          class="mb-2.5 block text-sm font-semibold text-heading"
        >
          Twitch Channel
        </label>

        <input
          id="channel"
          v-model="channel"
          type="text"
          name="channel"
          autocomplete="on"
          required
          :disabled="connected"
          class="block w-full rounded border border-default-medium bg-neutral-secondary-medium px-3.5 py-2.5 text-base text-heading shadow-xs placeholder:text-body focus:border-brand focus:ring-brand"
        />
      </div>

      <!-- Voice -->
      <section class="flex flex-col gap-6">
        <div>
          <label
            for="voices"
            class="mb-2.5 block text-sm font-semibold text-heading"
            >Select a Voice</label
          >

          <select
            id="voices"
            v-model="currentVoice"
            name="voices"
            class="block w-full px-3.5 py-2.5 bg-neutral-secondary-medium border border-default-medium text-heading text-sm rounded-base focus:ring-brand focus:border-brand shadow-xs placeholder:text-body rounded"
          >
            <option v-for="v in voices" :key="`${v.name}-${v.lang}`" :value="v">
              {{ v.name }} {{ v.lang }}
            </option>
          </select>
        </div>
      </section>
    </section>

    <!-- Settings Modal -->

    <button
      @click="modalOpen = true"
      type="button"
      class="bg-gray-200 hover:bg-gray-300 text-black font-bold py-2 px-4 rounded font-gotham"
    >
      Settings
    </button>

    <!-- Backdrop -->
    <div
      v-if="modalOpen"
      class="fixed inset-0 z-50 flex items-center justify-center bg-black/50"
      @click.self="close"
    >
      <!-- Modal -->
      <div class="w-full max-w-xl rounded-xl bg-white p-6 shadow-xl">
        <div class="mb-4 flex items-center justify-between">
          <h2 class="text-xl font-semibold">Settings</h2>

          <button @click="close" class="text-gray-500 hover:text-black">
            ✕
          </button>
        </div>

        <div class="mb-6">
          <div class="mt-4 flex flex-col gap-4">
            <!-- Toggles -->
            <div class="flex flex-col gap-4">
              <label class="flex items-center gap-2 text-sm">
                <input
                  id="randomVoice"
                  v-model="unique"
                  type="checkbox"
                  class="form-checkbox rounded"
                />

                <span>Assign random voice to each user</span>
              </label>

              <label class="flex items-center gap-2 text-sm">
                <input
                  id="readUsernames"
                  v-model="readUsernames"
                  type="checkbox"
                  class="form-checkbox rounded"
                />

                <span>Read usernames before messages</span>
              </label>
            </div>

            <div class="flex flex-col gap-2">
              <label for="ignoreList">Ignored Users / Bots</label>

              <TagsInput
                :tags="ignored"
                placeholder="Enter usernames..."
                @on-tags-changed="changeIgnored"
              />
            </div>

            <div class="flex flex-col gap-2">
              <label for="max">Max characters / message</label>

              <input
                id="max"
                v-model="max"
                class="form-input w-full rounded"
                type="number"
                min="1"
                max="400"
                step="1"
                name="max"
              />
            </div>

            <!-- Sliders -->
            <div class="flex flex-col gap-4 p-4 rounded border border-gray-300">
              <label for="ignoreList">Voice Settings</label>
              <div class="flex flex-1 flex-col gap-1.5">
                <label for="volume">Volume</label>

                <div class="flex items-center gap-2">
                  <input
                    id="volume"
                    v-model="volume"
                    class="w-full"
                    type="range"
                    min="0"
                    max="1"
                    step="0.1"
                  />

                  <small>{{ volume }}</small>
                </div>
              </div>

              <div class="flex flex-1 flex-col gap-1.5">
                <label for="pitch">Pitch</label>

                <div class="flex items-center gap-2">
                  <input
                    id="pitch"
                    v-model="pitch"
                    class="w-full"
                    type="range"
                    min="0"
                    max="2"
                    step="0.1"
                  />

                  <small>{{ pitch }}</small>
                </div>
              </div>

              <div class="flex flex-1 flex-col gap-1.5">
                <label for="rate">Rate</label>

                <div class="flex items-center gap-2">
                  <input
                    id="rate"
                    v-model="rate"
                    class="w-full"
                    type="range"
                    min="0.1"
                    max="2"
                    step="0.1"
                  />

                  <small>{{ rate }}</small>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div class="flex justify-end gap-2">
          <button
            @click="close"
            class="rounded border border-gray-300 px-4 py-2 hover:bg-gray-100 w-full"
          >
            Close
          </button>
        </div>
      </div>
    </div>

    <!-- Connection -->
    <button
      type="button"
      :class="[
        connected
          ? 'bg-red-500 hover:bg-red-700'
          : 'bg-blue-500 hover:bg-blue-700',
        ' text-white font-bold py-2 px-4 rounded font-gotham flex items-center justify-center',
      ]"
      :disabled="!channel.length || connecting || !currentVoice"
      @click="connected ? disconnect() : connect()"
    >
      <template v-if="connecting">
        <Loader2 class="mr-2 h-6 w-6 animate-spin" />
      </template>

      <template v-else>
        <span>
          {{ connected ? "Stop" : "Start" }}
        </span>
      </template>
    </button>

    <!-- Footer -->
    <footer class="flex w-full justify-center gap-2 text-sm">
      <small>Version 0.1.3</small>
      <small>&ndash;</small>

      <small>
        Created by
        <a
          href="https://github.com/notchris"
          target="_blank"
          class="underline underline-offset-2 hover:opacity-50"
        >
          notchris
        </a>
      </small>
    </footer>
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";

import TwitchJs, { Chat } from "twitch-js";
import type { ChatOptions } from "twitch-js/lib/Chat";

import { CircleQuestionMark, Loader2, X } from "lucide-vue-next";

import TagsInput from "./components/TagsInput.vue";

let chat: Chat | undefined;

type EmoteTag = {
  emotes: {
    start: number;
    end: number;
  }[];
};

const URL_REGEX =
  /\b((?:[a-z][\w-]+:(?:\/{1,3}|[a-z0-9%])|www\d{0,3}[.]|[a-z0-9.\-]+[.][a-z]{2,4}\/)(?:[^\s()<>]+|\(([^\s()<>]+|(\([^\s()<>]+\)))*\))+(?:\(([^\s()<>]+|(\([^\s()<>]+\)))*\)|[^\s`!()\[\]{};:'".,<>?«»“”‘’]))/g;

function speakMessage(
  text: string,
  voice: SpeechSynthesisVoice,
  volume: number,
  pitch: number,
  rate: number,
) {
  return new Promise<void>((resolve, reject) => {
    const msg = new SpeechSynthesisUtterance(text);
    msg.voice = voice;
    msg.volume = volume;
    msg.pitch = pitch;
    msg.rate = rate;
    msg.onend = () => {
      resolve();
    };
    msg.onerror = (e) => {
      reject(e.error);
    };

    speechSynthesis.cancel();
    speechSynthesis.speak(msg);
  });
}

interface HasRange {
  start: number;
  end: number;
}

export default defineComponent({
  name: "App",
  data() {
    return {
      modalOpen: false,
      abortController: undefined as AbortController | undefined,
      channel: "",
      voices: [] as SpeechSynthesisVoice[],
      currentVoice: undefined as SpeechSynthesisVoice | undefined,
      connected: false,
      connecting: false,
      ignored: [""] as string[],
      volume: 1.0,
      pitch: 1.0,
      rate: 1.0,
      max: 200,
      readUsernames: false,
      unique: false,
      userList: {} as Record<string, { v: SpeechSynthesisVoice }>,
    };
  },
  mounted() {
    // Check existing channel
    if (localStorage.channel) {
      this.channel = localStorage.channel;
    }

    // Check existing ignored
    if (localStorage.ignored) {
      try {
        this.ignored = JSON.parse(localStorage.ignored);
      } catch (e) {
        console.error("Failed to parse ignored list:", e);
        this.ignored = ["streamelements", "nightbot"];
      }
    }

    this.populateVoiceList();
    speechSynthesis.addEventListener("voiceschanged", this.populateVoiceList);
  },

  watch: {
    ignored(newIgnored) {
      localStorage.ignored = JSON.stringify(newIgnored);
    },
    channel(newChannel) {
      localStorage.channel = newChannel;
    },
  },

  beforeUnmount() {
    speechSynthesis.removeEventListener(
      "voiceschanged",
      this.populateVoiceList,
    );
  },

  methods: {
    close() {
      this.modalOpen = false;
    },

    async disconnect() {
      this.connected = false;
      this.connecting = false;

      this.abortController?.abort();
      this.abortController = undefined;

      if (speechSynthesis.speaking) {
        speechSynthesis.cancel();
      }

      if (chat) {
        await chat.disconnect();
      }

      this.userList = {};

      chat = undefined;
    },
    async connect() {
      try {
        this.connecting = true;

        this.abortController = new AbortController();

        await this.makeTwitchChatReadableStream(this.channel, {
          username: undefined,
          token: undefined,
          log: { enabled: false },
        }).pipeTo(this.makeSpeakMessageWritable(), {
          signal: this.abortController.signal,
        });
      } catch (e) {
        if (e instanceof DOMException && e.name === "AbortError") {
          return;
        }

        console.warn("Connection closed:", e);
      } finally {
        this.connecting = false;
      }
    },
    changeIgnored(tags: string[]) {
      this.ignored = tags;
    },

    populateVoiceList() {
      this.voices = speechSynthesis.getVoices();

      if (!this.currentVoice && this.voices.length) {
        this.currentVoice = this.voices[0];
      }
    },

    /**
     * Removes multiple non-overlapping ranges from a string.
     *
     * Ranges are inclusive in both start and end:
     *
     *   removeMultipleRanges("abcdefghijklmnop", [{start: 1, end: 2}, {start: 4, end: 8}]);
     *   // -> "adjklmnop"
     *
     * (Inclusive because Twitch API emoji ranges are inclusive)
     */
    removeMultipleRanges(str: string, ranges: HasRange[]) {
      ranges.sort((a, b) => b.start - a.start);
      for (let i = 0; i < ranges.length; i++) {
        let {start, end} = ranges[i];
        if (!(start >= 0 && end < str.length && start < end)) {
          continue;
        }
        str = str.slice(0, start) + str.slice(end+1);
      }

      return str;
    },

    makeTwitchChatReadableStream(
      channel: string,
      options: Partial<ChatOptions>,
    ) {
      chat = new TwitchJs.Chat(options);

      return new ReadableStream<{ message: string; user: string }>({
        start: async (controller) => {
          chat?.on("PRIVMSG", (message) => {
            const user = message.username.toLowerCase();
            let msg = message.message || "";

            const tags = message.tags as EmoteTag;

            msg = this.removeMultipleRanges(msg, tags.emotes || []);
            msg = msg.replace(URL_REGEX, "").trim();

            if (!msg.length) return;

            if (this.ignored.some((u) => u.toLowerCase() === user)) {
              return;
            }

            if (!this.userList[user]) {
              this.userList[user] = {
                v: this.getRandomVoice(),
              };
            }

            if (this.readUsernames) {
              msg = `${user} ${msg}`;
            }

            controller.enqueue({
              message: msg,
              user,
            });
          });

          await chat?.connect();

          this.connected = true;
          this.connecting = false;

          await chat?.join(channel);
        },

        cancel: async () => {
          await chat?.disconnect();
        },
      });
    },
    makeSpeakMessageWritable() {
      return new WritableStream<{ message: string; user: string }>({
        write: async (data) => {
          await speakMessage(
            data.message.slice(0, this.max),

            this.unique
              ? this.userList[data.user].v
              : (this.currentVoice as SpeechSynthesisVoice),
            this.volume,
            this.pitch,
            this.rate,
          );
        },
      });
    },
    getRandomVoice() {
      const englishVoices = this.voices.filter((v) =>
        v.lang.toLowerCase().startsWith("en"),
      );

      const pool = englishVoices.length ? englishVoices : this.voices;

      return pool[Math.floor(Math.random() * pool.length)];
    },
  },
  components: {
    Loader2,
    X,
    CircleQuestionMark,
    TagsInput,
  },
});
</script>
