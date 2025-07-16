<template>

  <div class="container py-4 settings-bg">
    <div class="alert alert-info mb-4 shadow-sm">
      <div class="d-flex flex-column flex-md-row align-items-md-center justify-content-between">
        <span><i class="el-icon-info mr-2"></i> The following settings are only retained locally in your browser.</span>
      </div>
    </div>

    <div class="row g-4">
      <div class="col-12 col-lg-6">
        <div class="card h-100 border-0 shadow-sm settings-card">
          <div class="card-header bg-primary text-white d-flex align-items-center">
            <i class="el-icon-key mr-2" style="font-size:1.5em;"></i>
            <h2 class="mb-0 fs-5">OpenAI Settings</h2>
          </div>
          <div class="card-body">
            <div class="mb-3">
              <label class="form-label fw-bold">API Key
              </label>
              <el-input placeholder="sk-xxxx" v-model="openai_key" @change="onKeyChange('openai_key')" />
            </div>
            <div class="mb-3">
              <label class="form-label fw-bold">GPT Model</label>
              <div>
                <el-radio-group v-model="gpt_model" @change="onKeyChange('gpt_model')">
                  <el-radio label="gpt-3.5-turbo">gpt-3.5-turbo</el-radio>
                  <el-radio label="gpt-4">gpt-4</el-radio>
                </el-radio-group>
              </div>
            </div>
            <div class="mb-3">
              <label class="form-label fw-bold">GPT Prompt
                <span class="desc_text ms-2">(Customize the system prompt for GPT answers)</span>
              </label>
              <el-input type="textarea" placeholder="You can setup custom prompt here" :rows="4"
                        v-model="gpt_system_prompt" @change="onKeyChange('gpt_system_prompt')"/>
            </div>
           
          </div>
        </div>
      </div>

      <div class="col-12 col-lg-6">
        <div class="card h-100 border-0 shadow-sm settings-card">
          <div class="card-header bg-success text-white d-flex align-items-center">
            <i class="el-icon-microphone mr-2" style="font-size:1.5em;"></i>
            <h2 class="mb-0 fs-5">Azure Speech Recognition</h2>
          </div>
          <div class="card-body">
            <div class="mb-3">
              <label class="form-label fw-bold">Azure Token
              </label>
              <el-input placeholder="Input Your Azure Speech Resource Token (KEY 1)" v-model="azure_token"
                        @change="onKeyChange('azure_token')" />
            </div>
            <div class="mb-3">
              <label class="form-label fw-bold">Location/Region
                <span class="desc_text ms-2">(e.g. eastasia, westus, etc.)</span>
              </label>
              <el-input placeholder="e.g. eastasia" v-model="azure_region" @change="onKeyChange('azure_region')" />
            </div>
            <div class="mb-3">
              <label class="form-label fw-bold">Recognition Language
              </label>
              <el-input placeholder="e.g. en-US" v-model="azure_language" @change="onKeyChange('azure_language')" />
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="row mt-4">
      <div class="col-12 d-flex justify-content-end">
        <el-button class="btn btn-warning px-4" @click="toDef"><i class="el-icon-refresh mr-1"></i> Reset All Settings</el-button>
      </div>
    </div>
  </div>
</template>

<script>
import config_util from "../utils/config_util"

export default {
  name: 'HelloWorld',
  props: {},
  data() {
    return {
      openai_key: "",
      gpt_model: "gpt-3.5-turbo",
      gpt_system_prompt: "",
      azure_token: "",
      azure_region: "",
      azure_language: "",
      open_ai_api_url: "https://platform.openai.com/api-keys",
      azure_application_url: "https://github.com/interview-copilot/Interview-Copilot/blob/main/docs/azure_speech_service_tutorial.md",
      full_language_codec_url: "https://learn.microsoft.com/en-us/azure/ai-services/speech-service/language-support?tabs=stt#speech-to-text"
    }
  },
  mounted() {
    this.openai_key = localStorage.getItem("openai_key")
    this.gpt_system_prompt = config_util.gpt_system_prompt()
    this.gpt_model = config_util.gpt_model()
    this.azure_token = localStorage.getItem("azure_token")
    this.azure_region = config_util.azure_region()
    this.azure_language = config_util.azure_language()
  },
  methods: {
    onKeyChange(key_name) {
      console.log("setItem", key_name, this[key_name])
      localStorage.setItem(key_name, this[key_name])
    },
    toDef() {
      localStorage.clear();
    }
  }


}


</script>

<style scoped>
.settings-bg {
  background: linear-gradient(120deg, #f8fafc 0%, #e0e7ef 100%);
  min-height: 100vh;
}
.settings-card {
  border-radius: 1rem;
  background: #fff;
  box-shadow: 0 2px 16px 0 rgba(60, 80, 120, 0.07);
  border: 1px solid #e3e6ee;
}
.card-header {
  border-top-left-radius: 1rem !important;
  border-top-right-radius: 1rem !important;
  font-size: 1.1rem;
  font-weight: 600;
  letter-spacing: 0.01em;
}
.form-label {
  font-weight: 600;
  margin-bottom: 0.2rem;
}
.desc_text {
  color: #6c757d;
  font-size: 0.95em;
  margin-bottom: 0;
}
.el-input.form-control {
  margin-top: 0.1rem;
  margin-bottom: 0.1rem;
}
.el-radio-group {
  margin-top: 0.1rem;
}
.el-radio {
  margin-right: 1.2em;
}
.btn-link {
  color: #0d6efd;
  text-decoration: underline;
  font-size: 0.98em;
}
.btn-link:hover {
  color: #0a58ca;
}
.btn-warning {
  font-weight: 600;
  border-radius: 0.5rem;
}
.el-icon-info, .el-icon-key, .el-icon-microphone, .el-icon-link, .el-icon-refresh, .el-icon-stopwatch {
  vertical-align: middle;
}


</style>
