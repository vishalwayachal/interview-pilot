<template>
  <div class="homeview_container">
    <div class="center_container">
      <div class="box asr_box sticky-asr">
        <div class="func_desc">
          <i class="el-icon-microphone"></i>
          <span>Speech Recognition</span>
        </div>
        <div v-if="!currentText" class="no-content" style="display: none">
          No Content
        </div>
        <div class="mb-2 d-flex align-items-center gap-3 flex-wrap">
          <div class="flex-grow-1">
            <select
              id="topic-select"
              v-model="selectedTopic"
              class="form-select"
            >
              <option value="general">General</option>
              <option value="nodejs">Node.js</option>
              <option value="vuejs">Vue.js</option>
              <option value="php">PHP</option>
              <option value="python">Python</option>
              <option value="postgresql">PostgreSQL</option>
              <option value="mongo">MongoDB</option>
            </select>
          </div>
          <div class="form-check form-switch mb-0">
            <input
              class="form-check-input"
              type="checkbox"
              id="auto-submit-toggle"
              v-model="autoSubmitSpeech"
            />
            <label class="form-check-label" for="auto-submit-toggle">
              Auto-submit
            </label>
          </div>
        </div>
        <textarea
          v-model="currentText"
          rows="2"
          placeholder="Speech Recognition Results will be shown here..."
          class="asr_input"
          @keyup.enter="askCurrentText"
        ></textarea>
        <div
          class="asr_content preview_content"
          style="display: none"
          v-if="currentText"
        >
          {{ currentText }}
        </div>
        <div class="single_part_bottom_bar">
          <el-button
            icon="el-icon-thumb"
            @click="askCurrentText"
            :disabled="!isGetGPTAnswerAvailable"
            type="primary"
            plain
          >
            Ask
          </el-button>
          <el-button
            icon="el-icon-delete"
            :disabled="!currentText"
            @click="clearASRContent"
            type="danger"
            plain
          >
            Clear
          </el-button>
          <el-button
            type="success"
            icon="el-icon-mic"
            @click="startCopilot"
            v-show="state === 'end'"
            :loading="copilot_starting"
            :disabled="copilot_starting"
            class="main-action-btn"
            >Start</el-button
          >
          <el-button
            type="info"
            icon="el-icon-video-camera"
            @click="startTabAudioCapture"
            v-show="state === 'end'"
            :disabled="copilot_starting"
            class="main-action-btn"
            >Audio</el-button
          >
    
          <el-button
            type="warning"
            icon="el-icon-stopwatch"
            :loading="copilot_stopping"
            @click="userStopCopilot"
            v-show="state === 'ing'"
            class="main-action-btn"
            >Stop</el-button
          >
          <MyTimer ref="MyTimer" />
      </div>
      </div>
        <div class="box video_box" v-if="tabVideoVisible">
        <div class="func_desc">
          <i class="el-icon-video-camera"></i>
          <span>Tab Video Preview</span>
        </div>
        <video ref="tabVideo" style="max-width: 100%; border-radius: 0.5rem; box-shadow: 0 2px 8px rgba(60,80,120,0.07);" autoplay muted playsinline></video>
      </div>
      <div class="box gpt_box">
        <div class="func_desc">
          <i class="el-icon-s-custom"></i>
          <span>GPT Answer</span>
        </div>
        <span>{{ lastQuestion }}</span>
        <LoadingIcon v-show="show_ai_thinking_effect" />
        <div
          class="ai_result_content preview_content"
          :class="{ thinking: show_ai_thinking_effect }"
        >
          {{ ai_result }}
        </div>
      </div>
    
    </div>
  </div>
</template>

<script>
import Assert from "assert-js";
import LoadingIcon from "@/components/LoadingIcon.vue";
import MyTimer from "@/components/MyTimer.vue";
import * as SpeechSDK from "microsoft-cognitiveservices-speech-sdk";
import OpenAI from "openai";
import config_util from "../utils/config_util";

export default {
  name: "HomeView",
  props: {},
  computed: {
    isDevMode() {
      return process.env.NODE_ENV === "development";
    },
    isGetGPTAnswerAvailable() {
      // return this.state === "ing" && !!this.currentText
      return !!this.currentText;
    },
  },
  components: { LoadingIcon, MyTimer },
  data() {
    return {
      currentText: " ",
      lastQuestion: "",
      state: "end", //end\ing
      ai_result: null,
      copilot_starting: false, //显示loading
      copilot_stopping: false,
      show_ai_thinking_effect: false,
      popStyle: {},
      selectedTopic: "general",
      autoSubmitSpeech: false,
      tabVideoStream: null,
      tabVideoVisible: false,
    };
  },
  async mounted() {
    console.log("STARTED----");
  },
  beforeDestroy() {},
  methods: {
    async askCurrentText() {
      const apiKey = localStorage.getItem("openai_key");

      if (this.selectedTopic != "general") {
        this.currentText = `[${this.selectedTopic}] ` + this.currentText;
      }
      let content = this.currentText;
      this.lastQuestion = content;
      this.ai_result = "";
      this.currentText = " ";
      this.show_ai_thinking_effect = true;
      const model = config_util.gpt_model();
      const gpt_system_prompt = config_util.gpt_system_prompt();
      content = gpt_system_prompt + "\n" + content;

      try {
        if (!apiKey) {
          throw new Error("You should setup an Open AI Key!");
        }

        const openai = new OpenAI({
          apiKey: apiKey,
          dangerouslyAllowBrowser: true,
        });
        const stream = await openai.chat.completions.create({
          model: model,
          messages: [{ role: "user", content: content }],
          stream: true,
        });
        this.show_ai_thinking_effect = false;

        for await (const chunk of stream) {
          const text = chunk.choices[0]?.delta?.content || "";
          this.ai_result += text;
        }
      } catch (e) {
        this.show_ai_thinking_effect = false;
        this.ai_result = "" + e;
      }
    },
    clearASRContent() {
      this.currentText = " ";
    },
    async startCopilot() {
      this.currentText = " ";
      this.copilot_starting = true;
      const token = localStorage.getItem("azure_token");
      const region = config_util.azure_region();
      const language = config_util.azure_language();
      const openai_key = localStorage.getItem("openai_key");
      console.log({ region, language });

      // Check for mediaDevices and getUserMedia support
      if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
        if (this.$message) {
          this.$message.error(
            "Your browser does not support microphone access. Please use a modern browser and ensure the page is served over HTTPS."
          );
        } else {
          alert(
            "Your browser does not support microphone access. Please use a modern browser and ensure the page is served over HTTPS."
          );
        }
        this.copilot_starting = false;
        return;
      }

      // Request microphone permission before initializing SpeechSDK
      try {
        await navigator.mediaDevices.getUserMedia({ audio: true });
      } catch (err) {
        if (this.$message) {
          this.$message.error(
            "Microphone permission denied or unavailable. Please allow microphone access in your browser settings."
          );
        } else {
          alert(
            "Microphone permission denied or unavailable. Please allow microphone access in your browser settings."
          );
        }
        this.copilot_starting = false;
        return;
      }

      try {
        if (!openai_key) {
          throw new Error("You should setup Open AI API Token");
        }
        if (!token) {
          throw new Error("You should setup Azure token");
        }
        if (!region) {
          throw new Error("You should setup Azure region");
        }

        const speechConfig = SpeechSDK.SpeechConfig.fromSubscription(
          token,
          region
        );
        speechConfig.speechRecognitionLanguage = language;
        const audioConfig = SpeechSDK.AudioConfig.fromDefaultMicrophoneInput();
        this.recognizer = new SpeechSDK.SpeechRecognizer(
          speechConfig,
          audioConfig
        );
      } catch (e) {
        console.error("Error starting copilot:", e);
        this.$message.error("Start Failed: " + e);
        this.copilot_starting = false;
        return;
      }

      const recognizer = this.recognizer;
      const sdk = SpeechSDK;

      recognizer.recognized = (sender, event) => {
        if (
          sdk.ResultReason.RecognizedSpeech === event.result.reason &&
          event.result.text.length > 0
        ) {
          const text = event.result.text;
          this.currentText = this.currentText + "\n" + text;
          // If autoSubmitSpeech is ON, voice pilot is started, currentText exists, and question is completed (ends with '?'), execute askCurrentText
          if (
            this.autoSubmitSpeech &&
            this.state === "ing" &&
            this.currentText &&
            /[\?？]\s*$/.test(this.currentText.trim())
          ) {
            this.askCurrentText();
          }
        } else if (sdk.ResultReason.NoMatch === event.result.reason) {
          console.log("Speech could not be recognized");
        }
      };

      recognizer.startContinuousRecognitionAsync(
        () => {
          this.copilot_starting = false;
          this.state = "ing";
          this.$refs.MyTimer.start();
          window.console.log("recognition started");
        },
        (err) => {
          this.copilot_starting = false;
          this.$message.error("Start Failed:" + err);
          window.console.error("recogniton start failed", err);
        }
      );
    },
    userStopCopilot() {
      this.copilot_stopping = true;
      if (this.tabVideoStream) {
        this.tabVideoStream.getTracks().forEach(track => track.stop());
        this.tabVideoStream = null;
        this.tabVideoVisible = false;
      }
      this.recognizer.stopContinuousRecognitionAsync(
        () => {
          console.log("stoped");
          this.copilot_stopping = false;
          this.state = "end";
          this.$refs.MyTimer.stop();
        },
        (err) => {
          console.log("err:", err);
        }
      );
    },
    async startTabAudioCapture() {
      // Prompt user to select a tab/window to capture audio and video from
      try {
        const stream = await navigator.mediaDevices.getDisplayMedia({
          video: true,
          audio: { echoCancellation: false, noiseSuppression: false }
        });
        // Show video in the page
        this.tabVideoStream = stream;
        this.tabVideoVisible = true;
        this.$nextTick(() => {
          const videoEl = this.$refs.tabVideo;
          if (videoEl) {
            videoEl.srcObject = stream;
            videoEl.play();
          }
        });
        // Check for audio tracks
        const audioTracks = stream.getAudioTracks();
        if (!audioTracks.length) {
          this.$message.error("No audio track found in selected tab/window. Please ensure you select a tab or window with audio playing.");
          stream.getTracks().forEach(track => track.stop());
          this.tabVideoVisible = false;
          return;
        }
        // Only use the audio stream for speech recognition
        const audioStream = new MediaStream(audioTracks);
        const token = localStorage.getItem("azure_token");
        const region = config_util.azure_region();
        const language = config_util.azure_language();
        if (!token || !region) {
          this.$message.error("Azure token/region not set.");
          stream.getTracks().forEach(track => track.stop());
          this.tabVideoVisible = false;
          return;
        }
        const speechConfig = SpeechSDK.SpeechConfig.fromSubscription(token, region);
        speechConfig.speechRecognitionLanguage = language;
        let audioConfig;
        try {
          audioConfig = SpeechSDK.AudioConfig.fromStreamInput(audioStream);
        } catch (e) {
          this.$message.error("Tab audio capture is not supported in this browser or by the Azure SDK. This feature requires browser and SDK support for MediaStream input.\nTry using a virtual audio cable or set your system audio as the microphone.\n\nTechnical details: " + (e && e.message ? e.message : e));
          stream.getTracks().forEach(track => track.stop());
          this.tabVideoVisible = false;
          return;
        }
        this.recognizer = new SpeechSDK.SpeechRecognizer(speechConfig, audioConfig);
        const recognizer = this.recognizer;
        const sdk = SpeechSDK;
        this.currentText = " ";
        this.copilot_starting = true;
        recognizer.recognized = (sender, event) => {
          if (
            sdk.ResultReason.RecognizedSpeech === event.result.reason &&
            event.result.text.length > 0
          ) {
            const text = event.result.text;
            this.currentText = this.currentText + "\n" + text;
            if (
              this.autoSubmitSpeech &&
              this.state === "ing" &&
              this.currentText &&
              /[\?？]\s*$/.test(this.currentText.trim())
            ) {
              this.askCurrentText();
            }
          } else if (sdk.ResultReason.NoMatch === event.result.reason) {
            console.log("Speech could not be recognized");
          }
        };
        recognizer.startContinuousRecognitionAsync(
          () => {
            this.copilot_starting = false;
            this.state = "ing";
            this.$refs.MyTimer.start();
            window.console.log("tab audio recognition started");
          },
          (err) => {
            this.copilot_starting = false;
            this.$message.error("Tab Audio Start Failed:" + err);
            window.console.error("tab audio recognition start failed", err);
            stream.getTracks().forEach(track => track.stop());
            this.tabVideoVisible = false;
          }
        );
      } catch (err) {
        let msg = "Tab audio capture failed: " + (err && err.message ? err.message : err);
        if (err && err.name === 'NotSupportedError') {
          msg += "\n\nYour browser does not support tab audio capture with getDisplayMedia. Try updating your browser, or use a different browser. If you need to capture system audio, consider using a virtual audio cable or set your system audio as the microphone.";
        }
        this.$message.error(msg);
        this.tabVideoVisible = false;
      }
    },
  },
};

const demo_text = `
Hello, thank you for coming for the interview. Please introduce yourself.

I'm Jhon, currently an undergraduate student majoring in Data Science at HK University. I am in the top 10% of my class, specializing in deep learning and proficient in web development. Additionally, I have contributed to several well-known open-source projects as mentioned in my resume.

Alright, let me ask you a machine learning question.

Sure, go ahead.

Can you explain the Hidden Markov Model?
`;

async function sleep(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
/* Sticky ASR section for mobile */
.sticky-asr {
  position: static;
}

@media (max-width: 600px) {
  .sticky-asr {
    position: sticky;
    top: 0;
    z-index: 100;
    background: #fff;
    box-shadow: 0 2px 8px 0 rgba(60, 80, 120, 0.07);
  }
}
@import "https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css";

.homeview_container {
  min-height: 100vh;
  background: linear-gradient(120deg, #f8fafc 0%, #e0e7ef 100%);
  padding: 0;
}

.center_container {
  display: flex;
  flex-wrap: wrap;
  gap: 2rem;
  justify-content: center;
  align-items: flex-start;
  width: 100%;
  margin: 0 auto;
  padding: 2rem 0;
}

.box {
  background: #fff;
  border-radius: 1rem;
  box-shadow: 0 2px 16px 0 rgba(60, 80, 120, 0.07);
  padding: 1rem 1.2rem 0.75rem 1.2rem;
  min-width: 320px;
  max-width: 420px;
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  border: 1px solid #dee2e6;
}

.func_desc {
  text-align: center;
  font-size: 1.1rem;
  font-weight: 600;
  color: #3b4252;
  margin-bottom: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
}

.no-content {
  color: #adb5bd;
  text-align: center;
  margin-bottom: 0.5rem;
  font-style: italic;
  font-size: 1rem;
}

.asr_input {
  width: 100%;
  min-height: 36px;
  max-height: 100px;
  resize: vertical;
  font-size: 1rem;
  font-family: "JetBrains Mono", "Courier New", Courier, monospace;
  border-radius: 0.5rem;
  padding: 0.35rem 0.6rem;
  margin-bottom: 0.25rem;
  background: #f8f9fa;
  border: 1px solid #ced4da;
  transition: border 0.2s;
}
.asr_input:focus {
  border: 1.5px solid #7fa7e6;
  outline: none;
  background: #f1f5fa;
}

.preview_content {
  background: #f4f7fa;
  border-radius: 0.5rem;
  padding: 0.35rem 0.6rem;
  margin-bottom: 0.25rem;
  min-height: 20px;
  font-size: 1rem;
  color: #2d3748;
  white-space: pre-wrap;
  word-break: break-word;
  box-shadow: 0 1px 4px 0 rgba(60, 80, 120, 0.03);
}

.ai_result_content {
  overflow-y: auto;
  flex-grow: 1;
  min-height: 40px;
  margin-bottom: 0.5rem;
  font-family: "JetBrains Mono", "Courier New", Courier, monospace;
  font-size: 1rem;
  color: #2d3748;
  transition: background 0.2s;
}
.ai_result_content.thinking {
  background: repeating-linear-gradient(
    135deg,
    #e3eaf6,
    #e3eaf6 10px,
    #f4f7fa 10px,
    #f4f7fa 20px
  );
  opacity: 0.8;
}

.single_part_bottom_bar {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  gap: 0.5rem;
  margin-top: 0;
  width: 100%;
  margin-bottom: 0;
}
.single_part_bottom_bar > .el-button,
.single_part_bottom_bar > button {
  flex: 1 1 0;
  font-size: 1rem;
  border-radius: 0.5rem;
  padding: 0.5rem 0;
  min-width: 0;
  margin: 0;
  text-align: center;
}

@media (max-width: 900px) {
  .center_container {
    flex-direction: column;
    gap: 0.7rem;
    align-items: stretch;
    padding: 0.5rem 0;
  }
  .box {
    min-width: 0;
    max-width: 100vw;
    width: 100%;
    margin: 0 0 0.7rem 0;
    padding: 0.7rem 0.5rem 0.3rem 0.5rem;
    border-radius: 0.75rem;
    font-size: 1em;
    gap: 0.3rem;
  }
  .asr_box,
  .gpt_box {
    margin: 0;
  }
}

@media (max-width: 600px) {
  .homeview_container {
    padding: 0 1vw;
    min-width: 0;
    max-width: 100vw;
  }
  .center_container {
    gap: 0.3rem;
    min-width: 0;
    width: 100%;
    padding: 0.3rem 0;
  }
  .box {
    padding: 0.3rem 1vw 0.3rem 1vw;
    font-size: 1.05em;
    min-width: 0;
    max-width: 100vw;
    width: 100%;
    gap: 0.2rem;
  }
  .func_desc {
    font-size: 1.08em;
    margin-bottom: 0.3rem;
  }
  .asr_input,
  .preview_content,
  .ai_result_content {
    font-size: 1.08em;
    padding: 0.3rem 0.15rem;
  }
  .main-action-btn {
    min-width: 60px;
    font-size: 1.08em;
    padding: 0.3rem 0;
    border-radius: 0.5rem;
  }
  /* Show all action buttons in one row on mobile */
  .single_part_bottom_bar {
    flex-direction: row !important;
    gap: 0.5rem;
    margin-top: 0;
    justify-content: stretch;
    align-items: center;
    width: 100%;
    margin-bottom: 0;
  }
  .single_part_bottom_bar > .el-button,
  .single_part_bottom_bar > button {
    width: 100%;
    min-width: 0;
    margin: 0;
    font-size: 1.08em;
    flex: 1 1 0;
    text-align: center;
  }
}
</style>
