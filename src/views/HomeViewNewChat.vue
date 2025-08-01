/* Force chat rows to be end-to-end width */ .chat-history-scroll-fix
.row.g-0.mb-2 { width: 100vw !important; max-width: 100vw !important;
margin-left: calc(-1 * ((100vw - 100%) / 2)) !important; margin-right: calc(-1 *
((100vw - 100%) / 2)) !important; box-sizing: border-box; } /* Make message text
span full width of chat bubble */ .chat-history-scroll-fix .d-block.mt-1 {
display: block; width: 100%; word-break: break-word; }
<template>
  <div
    class="container-fluid min-vh-100 d-flex flex-column bg-light px-0 router_view"
  >
    <div class="chat-app-wrapper mx-auto">
      <!-- Header -->
      <nav
        class="navbar navbar-expand-lg navbar-light bg-white shadow-sm py-2 px-3 sticky-top chat-app-header"
      >
        <div
          class="container-fluid px-0 d-flex flex-wrap align-items-center justify-content-between"
        >
          <div class="d-flex align-items-center gap-2 flex-wrap">
            <div
              class="input-group input-group-sm me-2"
              style="min-width: 120px; max-width: 180px"
            >
              <label
                class="input-group-text bg-light text-secondary"
                for="topicSelect"
                >Topic</label
              >
              <select
                v-model="selectedTopic"
                class="form-select"
                id="topicSelect"
                aria-label="Select topic"
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
            <div class="form-check form-switch ms-2">
              <input
                id="autoSubmitSpeech"
                type="checkbox"
                v-model="autoSubmitSpeech"
                class="form-check-input"
              />
              <label class="form-check-label" for="autoSubmitSpeech"
                >Auto</label
              >
            </div>
            <el-button
              type="success"
              icon="el-icon-mic"
              @click="startCopilot"
              v-show="state === 'end'"
              :loading="copilot_starting"
              :disabled="copilot_starting"
              class="btn btn-success btn-sm ms-2"
              aria-label="Start Copilot"
            ></el-button>
            <el-button
              type="info"
              icon="el-icon-video-camera"
              @click="startTabAudioCapture"
              v-show="state === 'end'"
              :disabled="copilot_starting"
              class="btn btn-info btn-sm ms-2"
              aria-label="Audio Capture"
            ></el-button>
            <el-button
              type="warning"
              icon="el-icon-stopwatch"
              :loading="copilot_stopping"
              @click="userStopCopilot"
              v-show="state === 'ing'"
              class="btn btn-warning btn-sm ms-2"
              aria-label="Stop Copilot"
            ></el-button>
          </div>
        </div>
      </nav>
      <!-- Chat History -->
      <div
        class="flex-grow-1 d-flex flex-column align-items-center w-100 bg-light pt-2 chat-app-body"
      >
        <div
          class="w-100 flex-grow-1 overflow-auto px-0 chat-history-scroll-fix"
          ref="chatHistory"
          tabindex="0"
          aria-live="polite"
          aria-atomic="false"
          style="scroll-behavior: smooth"
        >
          <div v-for="(item, idx) in qaHistory" :key="idx" class="row g-0 mb-2">
            <div
              :class="
                item.role === 'user' ? 'col-12 text-end' : 'col-12 text-start'
              "
            >
              <span v-if="item.role === 'user'" class="fw-bold text-light"
                >You:</span
              >
              <span v-if="item.role === 'ai'" class="fw-bold text-info"
                >Copilot:</span
              >
              <span class="d-block mt-1 plain-message">{{ item.text }}</span>
            </div>
          </div>
          <div v-if="interimResult && state === 'ing'" class="row g-0 mb-2">
            <div class="col-12 text-end">
              <span class="fw-bold text-light">You (speaking):</span>
              <span class="d-block mt-1 plain-message" style="opacity: 0.7">{{
                interimResult
              }}</span>
            </div>
          </div>
          <div class="loading-icon-wrapper" v-show="show_ai_thinking_effect">
            <LoadingIcon />
          </div>
        </div>
      </div>
      <!-- Input Bar -->
      <footer class="footer bg-white shadow-lg py-3 px-3 w-100 chat-app-footer">
        <form
          class="d-flex align-items-center gap-2 w-100"
          @submit.prevent="askCurrentText"
          style="flex-wrap: nowrap"
        >
          <input
            v-model="currentText"
            type="text"
            placeholder="Type or speak your question..."
            class="form-control form-control-lg border-primary flex-grow-1"
            @keyup.enter="askCurrentText"
            aria-label="Chat input"
            autocomplete="off"
            spellcheck="true"
            style="min-width: 0"
          />
          <el-button
            icon="el-icon-thumb"
            @click="askCurrentText"
            :disabled="!isGetGPTAnswerAvailable"
            type="primary"
            class="btn btn-primary btn-lg shadow-sm flex-shrink-0"
            aria-label="Send message"
          ></el-button>
          <el-button
            icon="el-icon-delete"
            :disabled="!currentText"
            @click="clearASRContent"
            type="danger"
            class="btn btn-danger btn-lg shadow-sm flex-shrink-0"
            aria-label="Clear input"
          ></el-button>
          <MyTimer ref="MyTimer" />
        </form>
      </footer>
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
  name: "HomeViewNewChat",
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
      lastRecognitionTime: 0,
      recognitionBuffer: "",
      processingRecognition: false,
      interimResult: "",
      silenceTimer: null,
      lastSpeechTime: 0,
      isSpeaking: false,
      audioLevel: 0,
      audioAnalyzer: null,
      audioContext: null,
      confidenceThreshold: 0.8,
      minSentenceLength: 5,
      ai_result: null,
      copilot_starting: false, //显示loading
      copilot_stopping: false,
      show_ai_thinking_effect: false,
      popStyle: {},
      selectedTopic: "general",
      autoSubmitSpeech: false,
      tabVideoStream: null,
      tabVideoVisible: false,
      qaHistory: [],
      suggestions: [],
      gptRunning: false,
    };
  },

  watch: {
    qaHistory() {
      // Automatically scroll to bottom when messages change
      this.$nextTick(() => {
        this.scrollToBottom(2, 80);
      });
    },
  },
  created() {
    // Initialize the debounced submit function
    this._debouncedSubmit = this._debounce(() => {
      if (
        this.currentText?.trim() &&
        (!this.lastQuestion || this.lastQuestion !== this.currentText.trim())
      ) {
        this.askCurrentText().finally(() => {
          this.processingRecognition = false;
          this.recognitionBuffer = "";
        });
      } else {
        this.processingRecognition = false;
      }
    }, 1000);
  },
  async mounted() {
    console.log("STARTED----");
    // Load history from localStorage
    const saved = localStorage.getItem("interview_qa_history");
    if (saved) {
      try {
        this.qaHistory = JSON.parse(saved);
        // Trim to last 10 conversation pairs (20 messages)
        if (this.qaHistory.length > 20) {
          this.qaHistory = this.qaHistory.slice(-20);
          localStorage.setItem(
            "interview_qa_history",
            JSON.stringify(this.qaHistory)
          );
        }
      } catch {}
    }
    // Scroll to bottom on mount
    this.$nextTick(() => {
      this.scrollToBottom(3, 80);
    });
  },
  beforeDestroy() {
    // Cleanup
    if (this.audioContext) {
      this.audioContext.close();
    }
    if (this.recognizer) {
      this.recognizer.close();
    }
  },
  methods: {
    // Smooth scroll to bottom helper
    // Smooth scroll to bottom helper (improved for mobile streaming)
    scrollToBottom(repeat = 3, delay = 100) {
      const chatHistory = this.$refs.chatHistory;
      if (!chatHistory) return;
      // Always force scroll to bottom
      for (let i = 0; i < repeat; i++) {
        setTimeout(() => {
          chatHistory.scrollTop = chatHistory.scrollHeight;
          // Try scrollIntoView on last message for mobile browsers
          const lastMsg = chatHistory.querySelector('.row.g-0.mb-2:last-child');
          if (lastMsg && typeof lastMsg.scrollIntoView === 'function') {
            lastMsg.scrollIntoView({ behavior: 'smooth', block: 'end' });
          }
        }, delay * i);
      }
    },

    // Debounce helper
    _debounce(func, wait) {
      let timeout;
      return (...args) => {
        clearTimeout(timeout);
        timeout = setTimeout(() => func.apply(this, args), wait);
      };
    },
    // ...existing code...

    handleInterimResult(text) {
      // Update interim display with smart truncation
      this.interimResult = text;

      // Enhanced question detection for interim results
      const potentialQuestion = text.trim().toLowerCase();
      const hasQuestionMarker = /[?？]\s*$/.test(potentialQuestion);
      const hasQuestionStart =
        /^(what|where|when|why|how|can you|could you|would you)\b/i.test(
          potentialQuestion
        );

      // Track potential questions for smarter auto-submit
      if (hasQuestionMarker || hasQuestionStart) {
        this._prepareQuickSubmit = true;
        this._lastQuestionIndicator = Date.now();
      }

      // Clear question preparation after delay if no final result
      if (this._prepareQuickSubmit) {
        clearTimeout(this._questionTimeout);
        this._questionTimeout = setTimeout(() => {
          this._prepareQuickSubmit = false;
        }, 3000); // Reset after 3 seconds if no final result
      }

      // Update audio activity timestamp
      this.lastSpeechTime = Date.now();
    },

    processFinalResult(text, isAutoSubmit) {
      const timeSinceLastRecognition = Date.now() - this.lastRecognitionTime;
      const currentText = text.trim();
      const currentBuffer = this.recognitionBuffer.trim();

      // Prevent duplicate sentences
      if (currentBuffer.toLowerCase().includes(currentText.toLowerCase())) {
        return; // Skip if the new text is already in the buffer
      }

      // Advanced sentence combination logic
      if (timeSinceLastRecognition < 1500) {
        // Check for partial overlaps
        const words = currentText.split(" ");
        const bufferWords = currentBuffer.split(" ");
        let overlapIndex = -1;

        // Look for overlapping phrases
        for (let i = 0; i < Math.min(words.length, bufferWords.length); i++) {
          const phrase = words
            .slice(0, i + 1)
            .join(" ")
            .toLowerCase();
          const bufferEnd = bufferWords
            .slice(-i - 1)
            .join(" ")
            .toLowerCase();
          if (bufferEnd === phrase) {
            overlapIndex = i;
          }
        }

        if (overlapIndex >= 0) {
          // Merge with overlap handling
          this.recognitionBuffer =
            currentBuffer + " " + words.slice(overlapIndex + 1).join(" ");
        } else {
          // Check for sentence boundaries
          const shouldAddPeriod = this.shouldAddPunctuation(
            currentBuffer,
            currentText
          );
          const separator = shouldAddPeriod ? ". " : " ";
          this.recognitionBuffer = currentBuffer + separator + currentText;
        }
      } else {
        // Handle question repetition
        if (
          this.isQuestionLike(currentText) &&
          this.isQuestionLike(currentBuffer)
        ) {
          // If both are questions, keep the newer one
          this.recognitionBuffer = currentText;
        } else {
          // Start new sentence
          this.recognitionBuffer = currentText;
        }
      }

      this.lastRecognitionTime = Date.now();
      this.currentText = this.recognitionBuffer;
      this.interimResult = ""; // Clear interim display

      // Enhanced auto-submit handling
      if (isAutoSubmit) {
        const isReadyToSubmit =
          this._prepareQuickSubmit ||
          this.isCompleteQuestion(this.recognitionBuffer) ||
          (this.isSpeaking === false &&
            Date.now() - this.lastSpeechTime > 1000);

        if (isReadyToSubmit && this.shouldAutoSubmit(this.recognitionBuffer)) {
          this._prepareQuickSubmit = false;
          this._debouncedSubmit();
        }
      }
    },

    shouldAddPunctuation(previous, current) {
      if (!previous || !current) return false;

      // Check if the new text starts with a capital letter
      const startsWithCapital = /^[A-Z]/.test(current);

      // Check if the previous text ends with punctuation
      const hasEndPunctuation = /[.!?。！？]\s*$/.test(previous);

      // Check for sentence-starting words
      const startsWithTransition =
        /^(however|therefore|furthermore|moreover|in addition|consequently)\b/i.test(
          current
        );

      return (startsWithCapital || startsWithTransition) && !hasEndPunctuation;
    },

    // Enhanced auto-submit detection with comprehensive patterns
    shouldAutoSubmit(text) {
      if (!this.autoSubmitSpeech || !this.state === "ing" || !text)
        return false;

      const normalizedText = text.trim().toLowerCase();

      // Definitive question markers with smart punctuation check
      if (/[?？]\s*$/.test(normalizedText) || /[.。]\s*$/.test(normalizedText))
        return true;

      // Comprehensive question phrase detection
      const hasQuestionPhrase =
        /(can you|could you|would you|what|where|when|why|how|tell me about|explain|describe|do you|have you|did you|are you|is there|could|should|will you)\b.*/.test(
          normalizedText
        );
      const hasQuestionContext =
        /\b(wondering|curious|like to know|explain|tell me|share|thoughts on|opinion|experience with|approach to)\b/.test(
          normalizedText
        );

      // Interview-specific patterns with broader context
      const isInterviewQuestion =
        /(your experience|your background|your role|your skills|your projects|your approach|your methodology|your process|your thoughts|your opinion|your view|your strategy|your expertise)\b/.test(
          normalizedText
        );

      // Technical interview patterns
      const isTechnicalQuestion =
        /(difference between|how does|how would you|best practices|design pattern|architecture|implementation|performance|scalability|optimization|framework|library|tool|technology)\b/.test(
          normalizedText
        );

      // Length-based smart submission for non-question statements that seem complete
      const isCompleteSentence =
        normalizedText.length > 50 && /[.!?。！？]\s*$/.test(normalizedText);

      return (
        hasQuestionPhrase ||
        (hasQuestionContext && normalizedText.length > 30) ||
        isInterviewQuestion ||
        isTechnicalQuestion ||
        isCompleteSentence
      );
    },

    // Helper method to submit text with proper state management
    _submitText() {
      if (
        this.currentText?.trim() &&
        (!this.lastQuestion || this.lastQuestion !== this.currentText.trim())
      ) {
        this.askCurrentText().finally(() => {
          this.processingRecognition = false;
          this.recognitionBuffer = "";
        });
      } else {
        this.processingRecognition = false;
      }
    },

    async askCurrentText() {
      if (this.gptRunning) return;
      this.gptRunning = true;
      // Always clear result and suggestions before starting
      this.ai_result = "";
      this.suggestions = [];
      const apiKey = localStorage.getItem("openai_key");

      let questionText = (this.currentText || "").trim();
      if (!questionText) {
        this.$message &&
          this.$message.error(
            "No question to answer. Please speak or type a question."
          );
        this.gptRunning = false;
        return;
      }
      if (this.selectedTopic != "general") {
        questionText = `[${this.selectedTopic}] ` + questionText;
      }
      let content = questionText;
      // Prevent duplicate answers for same question in auto-submit mode
      if (this.lastQuestion === content && this.ai_result) {
        this.gptRunning = false;
        return;
      }
      this.lastQuestion = content;
      this.show_ai_thinking_effect = true;
      const model = config_util.gpt_model();
      const gpt_system_prompt = config_util.gpt_system_prompt();
      content = gpt_system_prompt + "\n" + content;

      let answer = "";
      try {
        if (!apiKey) {
          throw new Error("You should setup an Open AI Key!");
        }

        const openai = new OpenAI({
          apiKey: apiKey,
          dangerouslyAllowBrowser: true,
        });
        // Get suggestions before GPT answer
        this.suggestions = await this.getSuggestions(questionText);
        const stream = await openai.chat.completions.create({
          model: model,
          messages: [{ role: "user", content: content }],
          stream: true,
        });
        this.show_ai_thinking_effect = false;

        // Add user question to chat history immediately
        this.qaHistory.push({ role: "user", text: this.lastQuestion });
        // Add empty AI answer to chat history for live update
        this.qaHistory.push({ role: "ai", text: "" });
        // Trim to last 10 conversation pairs (20 messages)
        if (this.qaHistory.length > 20) {
          this.qaHistory = this.qaHistory.slice(-20);
        }
        localStorage.setItem(
          "interview_qa_history",
          JSON.stringify(this.qaHistory)
        );

        for await (const chunk of stream) {
          const text = chunk.choices[0]?.delta?.content || "";
          this.ai_result += text;
          answer += text;
          // Update last AI message in qaHistory as answer grows
          if (
            this.qaHistory.length > 0 &&
            this.qaHistory[this.qaHistory.length - 1].role === "ai"
          ) {
            this.qaHistory[this.qaHistory.length - 1].text = answer;
            localStorage.setItem(
              "interview_qa_history",
              JSON.stringify(this.qaHistory)
            );
          }
          // Smooth scroll to bottom as answer grows (force for mobile)
          this.$nextTick(() => {
            this.scrollToBottom(3, 80);
          });
        }
        // Auto-clear input after answer
        this.currentText = " ";
        // Final smooth scroll after qaHistory update
        this.$nextTick(() => {
          this.scrollToBottom(4, 120);
        });
        // Clear suggestions after answer
        setTimeout(() => {
          this.suggestions = [];
        }, 3000);
      } catch (e) {
        this.show_ai_thinking_effect = false;
        this.ai_result = "" + e;
        this.$message && this.$message.error("GPT Error: " + e);
        this.gptRunning = false;
        return;
      }
      this.gptRunning = false;
    },
    async getSuggestions(question) {
      // Simple contextual prompt logic, can be replaced with GPT or more advanced logic
      const q = (question || "").toLowerCase();
      let suggestions = [];
      if (q.includes("introduce") || q.includes("yourself")) {
        suggestions.push(
          "Example: 'My name is John Doe, and I am a software engineer with 5 years of experience.'"
        );
        suggestions.push(
          "Tip: Mention your degree, university, or current role."
        );
        suggestions.push(
          "Phrase: 'I specialize in frontend development and have strong skills in Vue.js and React.'"
        );
        suggestions.push(
          "Tip: Highlight a recent project or achievement, e.g., 'Recently, I led a team to deliver a major web app.'"
        );
        suggestions.push(
          "Phrase: 'I'm excited about this opportunity because I enjoy solving real-world problems and collaborating with talented teams.'"
        );
      } else if (q.includes("strength") || q.includes("weakness")) {
        suggestions.push("Be honest but positive.");
        suggestions.push("Give examples to support your points.");
      } else if (q.includes("why") && q.includes("company")) {
        suggestions.push("Show enthusiasm for the company's mission.");
        suggestions.push("Relate your skills to the company's needs.");
      } else if (q.includes("project")) {
        suggestions.push("Describe the project's goal and your role.");
        suggestions.push("Highlight challenges and solutions.");
      } else if (q.includes("conflict") || q.includes("challenge")) {
        suggestions.push("Explain the situation clearly.");
        suggestions.push("Focus on resolution and learning.");
      } else if (q.includes("leadership")) {
        suggestions.push("Share a story showing leadership skills.");
        suggestions.push("Emphasize teamwork and results.");
      } else if (q.includes("machine learning") || q.includes("ai")) {
        suggestions.push("Define key concepts simply.");
        suggestions.push("Give practical examples.");
      } else if (q.includes("sql") || q.includes("database")) {
        suggestions.push("Mention normalization and indexing.");
        suggestions.push("Discuss query optimization.");
      } else if (
        q.includes("vue") ||
        q.includes("react") ||
        q.includes("frontend")
      ) {
        suggestions.push("Talk about component-based architecture.");
        suggestions.push("Mention state management and performance.");
      } else if (q.includes("node") || q.includes("backend")) {
        suggestions.push("Discuss REST APIs and async programming.");
        suggestions.push("Mention error handling and scalability.");
      } else {
        suggestions.push("Answer clearly and concisely.");
        suggestions.push("Use examples and structure your response.");
      }
      return suggestions;
    },
    clearHistory() {
      this.qaHistory = [];
      localStorage.removeItem("interview_qa_history");
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
      // Check for mediaDevices and getUserMedia support
      if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
        this.$message?.error(
          "Your browser does not support microphone access. Please use a modern browser and ensure the page is served over HTTPS."
        );
        this.copilot_starting = false;
        return;
      }
      // Request microphone permission before initializing SpeechSDK
      let stream;
      try {
        stream = await navigator.mediaDevices.getUserMedia({
          audio: {
            echoCancellation: true,
            noiseSuppression: true,
            autoGainControl: true,
          },
        });
        this.setupAudioAnalysis(stream);
      } catch (err) {
        this.$message?.error(
          "Microphone permission denied or unavailable. Please allow microphone access in your browser settings."
        );
        this.copilot_starting = false;
        return;
      }
      // Validate keys
      if (!openai_key) {
        this.$message?.error("You should setup Open AI API Token");
        this.copilot_starting = false;
        return;
      }
      if (!token) {
        this.$message?.error("You should setup Azure token");
        this.copilot_starting = false;
        return;
      }
      if (!region) {
        this.$message?.error("You should setup Azure region");
        this.copilot_starting = false;
        return;
      }
      // Setup SpeechSDK
      let recognizer;
      try {
        const speechConfig = SpeechSDK.SpeechConfig.fromSubscription(
          token,
          region
        );
        speechConfig.speechRecognitionLanguage = language;
        const audioConfig = SpeechSDK.AudioConfig.fromDefaultMicrophoneInput();
        recognizer = new SpeechSDK.SpeechRecognizer(speechConfig, audioConfig);
        this.recognizer = recognizer;
      } catch (e) {
        this.$message?.error("Start Failed: " + e);
        this.copilot_starting = false;
        return;
      }
      // Wire up event handlers with correct 'this' context using arrow functions
      recognizer.recognized = (sender, event) => {
        if (
          event.result.reason === SpeechSDK.ResultReason.RecognizedSpeech &&
          event.result.text
        ) {
          const text = event.result.text;
          const confidence = event.result.confidence || 0.8;
          if (text.trim().length < 3 && confidence < 0.7) return;
          if (typeof this.processSpeechResult === "function") {
            this.processSpeechResult(text, true, false, confidence);
          }
          // Auto-submit logic
          const currentBuffer = this.recognitionBuffer.trim();
          if (currentBuffer && this.autoSubmitSpeech && !this.gptRunning) {
            const timeSinceLastSubmit =
              Date.now() - (this._lastSubmitTime || 0);
            const minSubmitInterval = 2000;
            if (
              timeSinceLastSubmit > minSubmitInterval &&
              (this.isCompleteQuestion(currentBuffer) ||
                (/[?！?]\s*$/.test(currentBuffer) &&
                  currentBuffer.length > 10) ||
                (this._prepareQuickSubmit &&
                  this.shouldAutoSubmit(currentBuffer)))
            ) {
              this._lastSubmitTime = Date.now();
              this._debouncedSubmit();
              this._prepareQuickSubmit = false;
            }
          }
        } else if (event.result.reason === SpeechSDK.ResultReason.NoMatch) {
          const details = event.result.noMatchDetails;
          if (this.state === "ing") {
            if (
              details.reason === SpeechSDK.NoMatchReason.InitialSilenceTimeout
            ) {
              this.$message?.info("No speech detected. Please start speaking.");
            } else if (details.reason === SpeechSDK.NoMatchReason.Canceled) {
              return;
            } else {
              this.$message?.info(
                "Could not recognize speech. Please try speaking more clearly."
              );
            }
          }
        }
      };
      recognizer.recognizing = (s, e) => {
        if (e.result.text) {
          const interimText = e.result.text;
          clearTimeout(this._interimTimeout);
          this._interimTimeout = setTimeout(() => {
            if (interimText.length > 2) {
              const cleanText = interimText.replace(
                /^[\s]*(um|uh|eh|ah)[\s]+/i,
                ""
              );
              const estimatedConfidence = this.audioLevel > 0.5 ? 0.9 : 0.7;
              if (typeof this.processSpeechResult === "function") {
                this.processSpeechResult(
                  cleanText,
                  false,
                  true,
                  estimatedConfidence
                );
              }
              // Show interim speech in textbox as you speak
              this.currentText = cleanText;
            }
          }, 100);
        }
      };
      recognizer.canceled = (s, e) => {
        if (e.reason === SpeechSDK.CancellationReason.Error) {
          this.$message?.error(`Speech recognition error: ${e.errorDetails}`);
        }
      };
      recognizer.startContinuousRecognitionAsync(
        () => {
          this.copilot_starting = false;
          this.state = "ing";
          this.$refs.MyTimer?.start();
        },
        (err) => {
          this.copilot_starting = false;
          this.$message?.error("Start Failed:" + err);
        }
      );
    },
    userStopCopilot() {
      this.copilot_stopping = true;
      if (this.tabVideoStream) {
        this.tabVideoStream.getTracks().forEach((track) => track.stop());
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
          audio: { echoCancellation: false, noiseSuppression: false },
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
          this.$message.error(
            "No audio track found in selected tab/window. Please ensure you select a tab or window with audio playing."
          );
          stream.getTracks().forEach((track) => track.stop());
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
          stream.getTracks().forEach((track) => track.stop());
          this.tabVideoVisible = false;
          return;
        }
        const speechConfig = SpeechSDK.SpeechConfig.fromSubscription(
          token,
          region
        );
        speechConfig.speechRecognitionLanguage = language;
        let audioConfig;
        try {
          audioConfig = SpeechSDK.AudioConfig.fromStreamInput(audioStream);
        } catch (e) {
          this.$message.error(
            "Tab audio capture is not supported in this browser or by the Azure SDK. This feature requires browser and SDK support for MediaStream input.\nTry using a virtual audio cable or set your system audio as the microphone.\n\nTechnical details: " +
              (e && e.message ? e.message : e)
          );
          stream.getTracks().forEach((track) => track.stop());
          this.tabVideoVisible = false;
          return;
        }
        this.recognizer = new SpeechSDK.SpeechRecognizer(
          speechConfig,
          audioConfig
        );
        const recognizer = this.recognizer;
        const sdk = SpeechSDK;
        this.currentText = " ";
        this.copilot_starting = true;
        // Set up recognition event handlers for tab audio
        recognizer.recognized = (sender, event) => {
          if (
            sdk.ResultReason.RecognizedSpeech === event.result.reason &&
            event.result.text
          ) {
            this.processSpeechResult(event.result.text, true);
          } else if (sdk.ResultReason.NoMatch === event.result.reason) {
            console.log(
              "Speech could not be recognized:",
              event.result.noMatchDetails
            );
          }
        };

        recognizer.recognizing = (s, e) => {
          if (e.result.text) {
            console.log("Interim result:", e.result.text);
          }
        };

        recognizer.canceled = (s, e) => {
          if (e.reason === sdk.CancellationReason.Error) {
            console.error("Speech recognition error:", e.errorDetails);
            this.$message.error(`Speech recognition error: ${e.errorDetails}`);
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
            stream.getTracks().forEach((track) => track.stop());
            this.tabVideoVisible = false;
          }
        );
      } catch (err) {
        let msg =
          "Tab audio capture failed: " +
          (err && err.message ? err.message : err);
        if (err && err.name === "NotSupportedError") {
          msg +=
            "\n\nYour browser does not support tab audio capture with getDisplayMedia. Try updating your browser, or use a different browser. If you need to capture system audio, consider using a virtual audio cable or set your system audio as the microphone.";
        }
        this.$message.error(msg);
        this.tabVideoVisible = false;
      }
    },
    setupAudioAnalysis(stream) {
      try {
        // Create audio context and analyzer
        this.audioContext = new (window.AudioContext ||
          window.webkitAudioContext)();
        this.audioAnalyzer = this.audioContext.createAnalyser();
        this.audioAnalyzer.fftSize = 256;

        // Connect audio stream to analyzer
        const source = this.audioContext.createMediaStreamSource(stream);
        source.connect(this.audioAnalyzer);

        // Start monitoring audio levels
        this.monitorAudioLevel();
      } catch (error) {
        console.error("Audio analysis setup failed:", error);
      }
    },

    monitorAudioLevel() {
      if (!this.audioAnalyzer) return;

      const dataArray = new Uint8Array(this.audioAnalyzer.frequencyBinCount);
      const updateLevel = () => {
        this.audioAnalyzer.getByteFrequencyData(dataArray);

        // Calculate average volume level
        const average =
          dataArray.reduce((acc, value) => acc + value, 0) / dataArray.length;
        this.audioLevel = average / 255; // Normalize to 0-1

        // Update speaking state
        const isSpeakingNow = this.audioLevel > 0.1; // Threshold for speech
        if (isSpeakingNow !== this.isSpeaking) {
          this.isSpeaking = isSpeakingNow;
          if (isSpeakingNow) {
            this.handleSpeechStart();
          } else {
            this.handleSpeechEnd();
          }
        }

        // Continue monitoring if still recording
        if (this.state === "ing") {
          requestAnimationFrame(updateLevel);
        }
      };

      updateLevel();
    },

    handleSpeechStart() {
      // Clear any pending silence timers
      if (this.silenceTimer) {
        clearTimeout(this.silenceTimer);
      }
    },

    handleSpeechEnd() {
      // Set timer to check for sentence completion
      this.silenceTimer = setTimeout(() => {
        this.processPotentialSentenceEnd();
      }, 1000); // 1 second of silence
    },

    processPotentialSentenceEnd() {
      const currentBuffer = this.recognitionBuffer.trim();
      if (!currentBuffer || this.processingRecognition) return;

      // Check if we have a complete thought/sentence
      if (this.isCompleteSentence(currentBuffer)) {
        this.processSentenceEnd(currentBuffer);
      }
    },

    isCompleteSentence(text) {
      // Advanced sentence completion detection
      if (!text || text.length < this.minSentenceLength) return false;

      const normalizedText = text.trim();

      // Check for minimum word count
      const wordCount = normalizedText.split(/\s+/).length;
      if (wordCount < 3) return false;

      // Strong indicators of completion
      const hasEndPunctuation = /[.!?。？！]\s*$/.test(normalizedText);
      const isQuestion = this.isCompleteQuestion(normalizedText);

      // Check for complete grammatical structure
      const hasSubjectVerb =
        /\b(I|you|he|she|it|they|we)\b.*\b(is|are|was|were|have|has|had|do|does|did|can|could|will|would|should)\b/i.test(
          normalizedText
        );
      const hasCompleteStructure =
        /^[A-Z].*\b(and|or|but|because|however|therefore)\b.*[.!?]$/i.test(
          normalizedText
        );

      // Check for interview-specific patterns
      const isInterviewResponse =
        /^(yes|no|correct|exactly|absolutely|definitely|indeed)\b.*/.test(
          normalizedText.toLowerCase()
        );
      const isCompoundThought =
        normalizedText.includes(",") && normalizedText.length > 50;

      // Smart pause detection
      const hasNaturalPause =
        this.audioLevel < 0.1 && Date.now() - this.lastSpeechTime > 1000;

      return (
        hasEndPunctuation ||
        isQuestion ||
        (hasSubjectVerb && hasCompleteStructure) ||
        (isInterviewResponse && hasNaturalPause) ||
        (isCompoundThought && hasNaturalPause)
      );
    },

    processSentenceEnd(text) {
      if (!text || !this.autoSubmitSpeech) return;

      // Auto-submit based on enhanced criteria
      if (this.shouldAutoSubmit(text)) {
        this._debouncedSubmit();
      }
    },

    isQuestionLike(text) {
      if (!text) return false;
      const normalized = text.trim().toLowerCase();

      // Check for question marks
      if (/[?？]\s*$/.test(normalized)) return true;

      // Check for question starters
      const questionStarters =
        /^(what|where|when|why|how|can|could|would|should|do|does|did|is|are|was|were|have|has|had)\b/i;
      if (questionStarters.test(normalized)) return true;

      // Check for embedded questions
      const embeddedPatterns =
        /\b(tell me|explain|describe|clarify|elaborate on|share|discuss)\b/i;
      if (embeddedPatterns.test(normalized)) return true;

      return false;
    },

    // ...existing code...
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
@import "https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css";
@import "https://cdnjs.cloudflare.com/ajax/libs/animate.css/4.1.1/animate.min.css";

body {
  background: #f6f8fc;
}

.router_view {
  min-height: 100vh;
  background: #f6f8fc;
  box-sizing: border-box;
}

/* Modern chat row and bubble styling */
.chat-row {
  margin: 0 0 8px 0 !important;
  width: 100%;
  display: flex;
  align-items: flex-end;
}
.chat-bubble-wrapper {
  display: flex;
  align-items: flex-end;
  gap: 0.5rem;
  max-width: 100%;
}
.chat-avatar {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.3rem;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.07);
  border: 1.5px solid #e0e7ef;
  background: #fff;
  margin: 0 6px;
}
.user-avatar {
  background: #2563eb;
  color: #fff;
}
.ai-avatar {
  background: #fff;
  color: #2563eb;
}
.chat-bubble {
  border-radius: 1.2rem !important;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  max-width: 80vw;
  min-width: 60px;
  margin: 0 !important;
  position: relative;
  padding: 0;
  transition: background 0.2s;
}
.user-bubble {
  background: #2563eb !important;
  color: #fff !important;
  margin-left: auto;
}
.ai-bubble {
  background: #fff !important;
  color: #222 !important;
  border: 1px solid #e5e7eb;
  margin-right: auto;
}
.chat-timestamp {
  font-size: 0.85em;
  color: #a0a0a0;
  margin-left: 8px;
}
.chat-copy-btn {
  position: absolute;
  top: 8px;
  right: 12px;
  background: transparent;
  border: none;
  color: #bbb;
  opacity: 0;
  transition: opacity 0.2s;
  font-size: 1.1em;
  cursor: pointer;
  z-index: 2;
}
.chat-bubble:hover .chat-copy-btn {
  opacity: 1;
}
.chat-suggestions {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
}
.chat-suggestion-btn {
  border-radius: 1.2rem !important;
  font-size: 0.95rem;
  padding: 0.25rem 0.8rem;
  background: #f3f6fa !important;
  border: 1.5px solid #e0e7ef !important;
  color: #2563eb !important;
  transition: background 0.2s, color 0.2s;
}
.chat-suggestion-btn:hover {
  background: #2563eb !important;
  color: #fff !important;
}
.chat-code {
  background: #f6f8fc;
  border-radius: 0.7rem;
  padding: 0.7em 1em;
  font-size: 0.98em;
  font-family: "Fira Mono", "Consolas", "Menlo", monospace;
  color: #222;
  margin: 0.5em 0 0 0;
  overflow-x: auto;
}
.chat-input-area {
  min-height: 44px;
  max-height: 120px;
  resize: vertical;
  line-height: 1.5;
}
@media (max-width: 700px) {
  .chat-bubble {
    max-width: 98vw;
  }
  .chat-row {
    margin-bottom: 6px !important;
  }
}

/* Shared wrapper for header, body, and footer alignment */
/* Ensure header and body are perfectly aligned */
.chat-app-wrapper {
  max-width: 100%;
  width: 100%;
  margin-left: 0;
  margin-right: 0;
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  background: #f6f8fc;
  position: relative;
}
.chat-app-header {
  border-radius: 1.2rem 1.2rem 0 0;
  margin-bottom: 0;
  width: 100%;
  max-width: 100%;
  margin-left: 0;
  margin-right: 0;
  left: 0;
  right: 0;
  padding-left: 0 !important;
  padding-right: 0 !important;
  box-sizing: border-box;
}
.chat-app-body {
  width: 100%;
  max-width: 100%;
  margin-left: 0;
  margin-right: 0;
  margin-top: 0;
  flex: 1 1 auto;
  padding-bottom: 0;
  padding-left: 0 !important;
  padding-right: 0 !important;
  box-sizing: border-box;
}
/* Ensure chat history is always aligned with header */
.chat-history-scroll-fix {
  margin-left: 16px;
  margin-right: 16px;
  padding-left: 0 !important;
  padding-right: 0 !important;
  padding-bottom: 25px !important;
  background: #f6f8fc;
  max-width: calc(100% - 32px) !important;
  box-sizing: border-box;
}
.chat-app-footer {
  border-radius: 0 0 1.2rem 1.2rem !important;
  width: 100%;
  max-width: 100%;
  margin: 0;
  left: 0;
  right: 0;
  position: sticky;
  bottom: 0;
  z-index: 101;
}
.footer {
  box-shadow: 0 -2px 12px rgba(0, 0, 0, 0.07);
}

/* Remove all left/right margin and padding from chat rows */
.chat-history-scroll-fix .row,
.chat-history-scroll-fix .row.g-0,
.chat-history-scroll-fix .row.g-0.mb-2 {
  margin-left: 0 !important;
  margin-right: 0 !important;
  padding-left: 0 !important;
  padding-right: 0 !important;
  --bs-gutter-x: 0 !important;
  --bs-gutter-y: 0 !important;
  min-width: 0 !important;
  width: 100% !important;
  display: flex;
  align-items: flex-end;
  gap: 0.5rem;
  margin-top: 6px;
  margin-bottom: 6px;
  background: transparent;
  /* Add a subtle separator for clarity */
  /* border-bottom: 1px solid #f0f2f7; */
}

/* Remove all left/right margin and padding from columns */
.chat-history-scroll-fix [class^="col-"] {
  padding-left: 0 !important;
  padding-right: 0 !important;
  margin-left: 0 !important;
  margin-right: 0 !important;
}

/* Plain message style (no bubble) */
.plain-message {
  background: #f7fafd;
  color: #222;
  border: 1.5px solid #e3e8f0;
  border-radius: 0.8rem;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.04);
  padding: 0.5rem 1rem 0.5rem 1rem !important;
  margin: 2px 0 2px 0 !important;
  font-size: 1.08rem;
  line-height: 1.6;
  word-break: break-word;
  display: block;
  width: 100%;
  text-align: inherit;
  transition: background 0.2s, border 0.2s;
}

/* User message box (right) */
.text-end .plain-message {
  background: #e6f0ff;
  color: #174ea6;
  border: 1.5px solid #b6d4fe;
  text-align: right;
}
/* AI message box (left) */
.text-start .plain-message {
  background: #f7fafd;
  color: #222;
  border: 1.5px solid #e3e8f0;
  text-align: left;
}

/* Ensure alignment matches parent col-10 text-end/text-start */
.text-end .plain-message {
  text-align: right;
}
.text-start .plain-message {
  text-align: left;
}
.chat-history-scroll-fix .col-2.text-start,
.chat-history-scroll-fix .col-2.offset-10.text-end {
  display: flex;
  align-items: flex-end;
  justify-content: center;
  padding: 0 !important;
  margin: 0 !important;
}
.chat-history-scroll-fix .rounded-circle {
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.07);
  border: 1.5px solid #e0e7ef;
}

/* Footer modern look */
.footer {
  border-radius: 0 0 1.2rem 1.2rem !important;
  background: #fff !important;
  box-shadow: 0 -2px 12px rgba(0, 0, 0, 0.07);
  padding-left: 0.5rem !important;
  padding-right: 0.5rem !important;
}
.footer form {
  gap: 0.5rem !important;
}
.footer input.form-control-lg {
  border-radius: 1.2rem !important;
  background: #f3f6fa !important;
  border: 1.5px solid #e0e7ef !important;
  font-size: 1.08rem;
  padding: 0.6rem 1rem !important;
}
.footer .btn {
  border-radius: 1.2rem !important;
  font-size: 1.1rem;
  min-width: 44px;
  min-height: 44px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.07);
}

/* Improved loading icon styling */
.loading-icon-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 56px;
  margin-top: 8px;
  margin-bottom: 4px;
  padding-left: 0;
  padding-right: 0;
}
.loading-icon-wrapper > * {
  width: 48px !important;
  height: 48px !important;
  animation: spin 1.1s linear infinite;
}
@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

@media (max-width: 700px) {
  .chat-app-wrapper {
    max-width: 100vw;
    border-radius: 0 !important;
  }
  .chat-app-header {
    border-radius: 0 !important;
    padding-left: 0.25rem !important;
    padding-right: 0.25rem !important;
  }
  .chat-app-footer {
    border-radius: 0 0 1.2rem 1.2rem !important;
    padding-left: 0.25rem !important;
    padding-right: 0.25rem !important;
  }
  .chat-history-scroll-fix {
    padding-bottom: 10px !important;
  }
  .loading-icon-wrapper {
    padding-left: 0;
    padding-right: 0;
  }
  .w-100.flex-grow-1.overflow-auto.chat-history-scroll-fix {
    padding-left: 0 !important;
    padding-right: 0 !important;
  }
  .el-button,
  .el-button.btn,
  .footer .el-button,
  .footer .el-button.btn {
    padding: 8px 14px !important;
  }
}
</style>
