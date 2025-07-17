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
        <div v-if="autoSubmitSpeech" class="alert alert-info py-1 px-2 mb-1" style="font-size:0.98em;">
          <i class="el-icon-info"></i> Auto-submit is ON: Answers will be shown instantly when a question is recognized.
        </div>
        <div class="speech-input-container">
          <textarea
            v-model="currentText"
            rows="2"
            placeholder="Speech Recognition Results will be shown here..."
            class="asr_input"
            @keyup.enter="askCurrentText"
          ></textarea>
          <div v-if="interimResult && state === 'ing'" class="interim-result">
            {{ interimResult }}
          </div>
        </div>
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
      <!-- <div class="box suggestion_box" v-if="suggestions.length">
        <div class="func_desc">
          <i class="el-icon-edit-outline"></i>
          <span>Teleprompter Suggestions</span>
        </div>
        <div style="max-height: 120px; overflow-y: auto;">
          <div v-for="(s, idx) in suggestions" :key="idx" style="color:#2d3748; background:#f4f7fa; border-radius:0.5rem; padding:0.25rem 0.5rem; margin-bottom:0.1rem; font-size:1em;">
            {{ s }}
          </div>
        </div>
      </div> -->
      <!-- <div class="box history_box" v-if="qaHistory.length">
        <div class="func_desc">
          <i class="el-icon-document"></i>
          <span>Interview Q&A History</span>
        </div>
        <div style="max-height: 220px; overflow-y: auto;">
          <div v-for="(item, idx) in qaHistory" :key="idx" class="mb-2">
            <div style="font-weight:600; color:#3b4252;">Q: {{ item.question }}</div>
            <div style="color:#2d3748; background:#f4f7fa; border-radius:0.5rem; padding:0.25rem 0.5rem; margin-bottom:0.1rem;">A: {{ item.answer }}</div>
          </div>
        </div>
        <button class="btn btn-sm btn-outline-danger mt-1" @click="clearHistory">Clear History</button>
      </div> -->
    
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
  name: "HomeViewNew",
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
  created() {
    // Initialize the debounced submit function
    this._debouncedSubmit = this._debounce(() => {
      if (this.currentText?.trim() && (!this.lastQuestion || this.lastQuestion !== this.currentText.trim())) {
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
      } catch {}
    }
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
    // Debounce helper
    _debounce(func, wait) {
      let timeout;
      return (...args) => {
        clearTimeout(timeout);
        timeout = setTimeout(() => func.apply(this, args), wait);
      };
    },

    // Enhanced speech processing with confidence checking
    processSpeechResult(text, isAutoSubmit = false, isInterim = false, confidence = 1) {
      if (!text?.trim()) return;

      const now = Date.now();
      this.lastSpeechTime = now;

      // Clear any pending silence detection
      if (this.silenceTimeout) {
        clearTimeout(this.silenceTimeout);
      }

      // Set up silence detection for auto-submit
      if (this.autoSubmitSpeech && !isInterim) {
        this.silenceTimeout = setTimeout(() => {
          const currentBuffer = this.recognitionBuffer.trim();
          if (currentBuffer && this.shouldAutoSubmit(currentBuffer)) {
            this._debouncedSubmit();
          }
        }, 1500); // 1.5 seconds of silence triggers check
      }

      // Skip low confidence results for final recognition with dynamic threshold
      if (!isInterim) {
        const dynamicThreshold = this.audioLevel > 0.7 ? 0.7 : this.confidenceThreshold;
        if (confidence < dynamicThreshold) {
          console.log('Skipping low confidence result:', text, 'confidence:', confidence);
          return;
        }
      }

      // Handle interim results with improved processing
      if (isInterim) {
        this.handleInterimResult(text);
        return;
      }

      // Process final result with enhanced logic
      this.processFinalResult(text, isAutoSubmit);

      if (isInterim) {
        this.interimResult = text;
        return;
      }

      const timeSinceLastRecognition = now - this.lastRecognitionTime;
      
      // Improved continuation logic
      if (timeSinceLastRecognition < 1500) {
        // Check if we should add punctuation
        let separator = " ";
        if (!/[.!?]\s*$/.test(this.recognitionBuffer) && /^[A-Z]/.test(text)) {
          separator = ". ";
        }
        this.recognitionBuffer += separator + text;
      } else {
        this.recognitionBuffer = text;
      }
      
      this.lastRecognitionTime = now;
      this.interimResult = ""; // Clear interim when we get final result

      // Update display with combined text
      this.currentText = this.recognitionBuffer;

      // Check for auto-submit conditions
      if (isAutoSubmit && !this.processingRecognition) {
        const trimmedText = this.recognitionBuffer.trim();
        if (this.shouldAutoSubmit(trimmedText)) {
          this.processingRecognition = true;
          this._debouncedSubmit();
        }
      }
    },

    handleInterimResult(text) {
      // Update interim display with smart truncation
      this.interimResult = text;
      
      // Enhanced question detection for interim results
      const potentialQuestion = text.trim().toLowerCase();
      const hasQuestionMarker = /[?？]\s*$/.test(potentialQuestion);
      const hasQuestionStart = /^(what|where|when|why|how|can you|could you|would you)\b/i.test(potentialQuestion);
      
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
      
      // Smart sentence combination logic
      if (timeSinceLastRecognition < 1500) {
        // Check for sentence boundaries and proper continuation
        const shouldAddPeriod = this.shouldAddPunctuation(this.recognitionBuffer, text);
        const separator = shouldAddPeriod ? ". " : " ";
        this.recognitionBuffer = this.recognitionBuffer.trim() + separator + text;
      } else {
        // Start new sentence
        this.recognitionBuffer = text;
      }
      
      this.lastRecognitionTime = Date.now();
      this.currentText = this.recognitionBuffer;
      this.interimResult = ""; // Clear interim display

      // Check for immediate auto-submit if prepared
      if (this._prepareQuickSubmit && isAutoSubmit) {
        this._prepareQuickSubmit = false;
        if (this.shouldAutoSubmit(this.recognitionBuffer)) {
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
      const startsWithTransition = /^(however|therefore|furthermore|moreover|in addition|consequently)\b/i.test(current);
      
      return (startsWithCapital || startsWithTransition) && !hasEndPunctuation;
    },

    // Enhanced auto-submit detection with comprehensive patterns
    shouldAutoSubmit(text) {
      if (!this.autoSubmitSpeech || !this.state === "ing" || !text) return false;
      
      const normalizedText = text.trim().toLowerCase();
      
      // Definitive question markers with smart punctuation check
      if (/[?？]\s*$/.test(normalizedText) || /[.。]\s*$/.test(normalizedText)) return true;
      
      // Comprehensive question phrase detection
      const hasQuestionPhrase = /(can you|could you|would you|what|where|when|why|how|tell me about|explain|describe|do you|have you|did you|are you|is there|could|should|will you)\b.*/.test(normalizedText);
      const hasQuestionContext = /\b(wondering|curious|like to know|explain|tell me|share|thoughts on|opinion|experience with|approach to)\b/.test(normalizedText);
      
      // Interview-specific patterns with broader context
      const isInterviewQuestion = /(your experience|your background|your role|your skills|your projects|your approach|your methodology|your process|your thoughts|your opinion|your view|your strategy|your expertise)\b/.test(normalizedText);
      
      // Technical interview patterns
      const isTechnicalQuestion = /(difference between|how does|how would you|best practices|design pattern|architecture|implementation|performance|scalability|optimization|framework|library|tool|technology)\b/.test(normalizedText);
      
      // Length-based smart submission for non-question statements that seem complete
      const isCompleteSentence = normalizedText.length > 50 && /[.!?。！？]\s*$/.test(normalizedText);
      
      return hasQuestionPhrase || 
             (hasQuestionContext && normalizedText.length > 30) || 
             isInterviewQuestion || 
             isTechnicalQuestion ||
             isCompleteSentence;
    },

    // Helper method to submit text with proper state management
    _submitText() {
      if (this.currentText?.trim() && (!this.lastQuestion || this.lastQuestion !== this.currentText.trim())) {
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
        this.$message && this.$message.error("No question to answer. Please speak or type a question.");
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

        for await (const chunk of stream) {
          const text = chunk.choices[0]?.delta?.content || "";
          this.ai_result += text;
          answer += text;
        }
        // Save Q&A to history
        this.qaHistory.push({ question: this.lastQuestion, answer });
        localStorage.setItem("interview_qa_history", JSON.stringify(this.qaHistory));
        // Auto-clear input after answer
        this.currentText = " ";
        // Clear suggestions after answer
        setTimeout(() => { this.suggestions = []; }, 3000);
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
        suggestions.push("Example: 'My name is John Doe, and I am a software engineer with 5 years of experience.'");
        suggestions.push("Tip: Mention your degree, university, or current role.");
        suggestions.push("Phrase: 'I specialize in frontend development and have strong skills in Vue.js and React.'");
        suggestions.push("Tip: Highlight a recent project or achievement, e.g., 'Recently, I led a team to deliver a major web app.'");
        suggestions.push("Phrase: 'I'm excited about this opportunity because I enjoy solving real-world problems and collaborating with talented teams.'");
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
      } else if (q.includes("vue") || q.includes("react") || q.includes("frontend")) {
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
        const stream = await navigator.mediaDevices.getUserMedia({ 
          audio: {
            echoCancellation: true,
            noiseSuppression: true,
            autoGainControl: true
          } 
        });
        
        // Setup audio analysis for this stream
        this.setupAudioAnalysis(stream);
        
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

      // Enhanced recognition event handlers with smart processing
      recognizer.recognized = (sender, event) => {
        if (sdk.ResultReason.RecognizedSpeech === event.result.reason && event.result.text) {
          const result = event.result;
          const text = result.text;
          const confidence = result.confidence || 0.8; // Default confidence if not provided
          
          // Enhanced noise filtering
          if (text.trim().length < 3 && confidence < 0.7) {
            console.log('Filtered out short, low confidence result:', text);
            return;
          }
          
          // Process final result with confidence
          this.processSpeechResult(text, true, false, confidence);
          
          // Smart auto-submit with context awareness
          const currentBuffer = this.recognitionBuffer.trim();
          if (currentBuffer && this.autoSubmitSpeech && !this.gptRunning) {
            const timeSinceLastSubmit = Date.now() - (this._lastSubmitTime || 0);
            const minSubmitInterval = 2000; // Minimum 2 seconds between submissions
            
            // Check for definite question end and timing
            if (timeSinceLastSubmit > minSubmitInterval && 
                (this.isCompleteQuestion(currentBuffer) || 
                 (/[?！?]\s*$/.test(currentBuffer) && currentBuffer.length > 10) ||
                 (this._prepareQuickSubmit && this.shouldAutoSubmit(currentBuffer)))) {
              
              this._lastSubmitTime = Date.now();
              this._debouncedSubmit();
              this._prepareQuickSubmit = false;
            }
          }
        } else if (sdk.ResultReason.NoMatch === event.result.reason) {
          const details = event.result.noMatchDetails;
          console.log("Recognition details:", details);
          
          // Smart error handling based on reason
          if (this.state === "ing") {
            if (details.reason === sdk.NoMatchReason.InitialSilenceTimeout) {
              this.$message.info("No speech detected. Please start speaking.");
            } else if (details.reason === sdk.NoMatchReason.Canceled) {
              // Don't show error for intentional cancellation
              return;
            } else {
              this.$message.info("Could not recognize speech. Please try speaking more clearly.");
            }
          }
        }
      };

      recognizer.recognizing = (s, e) => {
        // Enhanced interim result handling
        if (e.result.text) {
          const interimText = e.result.text;
          
          // Debounce rapid interim updates
          clearTimeout(this._interimTimeout);
          this._interimTimeout = setTimeout(() => {
            // Smart filtering of interim results
            if (interimText.length > 2) {
              // Check for common speech recognition artifacts
              const cleanText = interimText.replace(/^\s*(um|uh|eh|ah)\s+/i, '');
              
              // Process interim with confidence estimation
              const estimatedConfidence = this.audioLevel > 0.5 ? 0.9 : 0.7;
              this.processSpeechResult(cleanText, false, true, estimatedConfidence);
              
              // Update UI feedback
              if (this.autoSubmitSpeech) {
                const isLikelyQuestion = /^(what|how|why|when|where|could|can|should|would|will|do|is|are)\b/i.test(cleanText) ||
                                       /[?]\s*$/.test(cleanText);
                if (isLikelyQuestion) {
                  this.$refs.textarea?.classList.add('potential-question');
                } else {
                  this.$refs.textarea?.classList.remove('potential-question');
                }
              }
            }
          }, 100); // Debounce 100ms
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
        // Set up recognition event handlers for tab audio
        recognizer.recognized = (sender, event) => {
          if (sdk.ResultReason.RecognizedSpeech === event.result.reason && event.result.text) {
            this.processSpeechResult(event.result.text, true);
          } else if (sdk.ResultReason.NoMatch === event.result.reason) {
            console.log("Speech could not be recognized:", event.result.noMatchDetails);
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
    setupAudioAnalysis(stream) {
      try {
        // Create audio context and analyzer
        this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
        this.audioAnalyzer = this.audioContext.createAnalyser();
        this.audioAnalyzer.fftSize = 256;

        // Connect audio stream to analyzer
        const source = this.audioContext.createMediaStreamSource(stream);
        source.connect(this.audioAnalyzer);

        // Start monitoring audio levels
        this.monitorAudioLevel();
      } catch (error) {
        console.error('Audio analysis setup failed:', error);
      }
    },

    monitorAudioLevel() {
      if (!this.audioAnalyzer) return;

      const dataArray = new Uint8Array(this.audioAnalyzer.frequencyBinCount);
      const updateLevel = () => {
        this.audioAnalyzer.getByteFrequencyData(dataArray);
        
        // Calculate average volume level
        const average = dataArray.reduce((acc, value) => acc + value, 0) / dataArray.length;
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
        if (this.state === 'ing') {
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
      const hasSubjectVerb = /\b(I|you|he|she|it|they|we)\b.*\b(is|are|was|were|have|has|had|do|does|did|can|could|will|would|should)\b/i.test(normalizedText);
      const hasCompleteStructure = /^[A-Z].*\b(and|or|but|because|however|therefore)\b.*[.!?]$/i.test(normalizedText);
      
      // Check for interview-specific patterns
      const isInterviewResponse = /^(yes|no|correct|exactly|absolutely|definitely|indeed)\b.*/.test(normalizedText.toLowerCase());
      const isCompoundThought = normalizedText.includes(',') && normalizedText.length > 50;
      
      // Smart pause detection
      const hasNaturalPause = this.audioLevel < 0.1 && Date.now() - this.lastSpeechTime > 1000;
      
      return hasEndPunctuation || 
             isQuestion || 
             (hasSubjectVerb && hasCompleteStructure) ||
             (isInterviewResponse && hasNaturalPause) ||
             (isCompoundThought && hasNaturalPause);
    },

    processSentenceEnd(text) {
      if (!text || !this.autoSubmitSpeech) return;

      // Auto-submit based on enhanced criteria
      if (this.shouldAutoSubmit(text)) {
        this._debouncedSubmit();
      }
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

.speech-input-container {
  position: relative;
  width: 100%;
}

.interim-result {
  position: absolute;
  bottom: -20px;
  left: 0;
  right: 0;
  font-size: 0.9em;
  color: #6b7280;
  font-style: italic;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  padding: 0 0.5rem;
  animation: fadeIn 0.3s ease-in-out;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}
</style>
