<template>

  <div class="container py-4">
    <div class="alert alert-info mb-4">
      <div class="d-flex flex-column flex-md-row align-items-md-center justify-content-between">
        <span>The following settings are only retained locally in your browser.</span>
        <span>See <a :href="github_url" target="_blank">Github</a> for setup instructions.</span>
      </div>
    </div>

    <div class="row">
      <div class="col-12 col-lg-6 mb-4">
        <div class="card h-100">
          <div class="card-header bg-primary text-white">
            <h2 class="mb-0">Open AI</h2>
          </div>
          <div class="card-body">
            <div class="mb-3 text-muted small">To use GPT, you need an API Key from the <a :href="open_ai_api_url" target="_blank">Open AI</a></div>
            <div class="mb-3">
              <label class="form-label fw-bold">API Key</label>
              <el-input class="form-control" placeholder="sk-xxxx" v-model="openai_key" @change="onKeyChange('openai_key')" />
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
              <label class="form-label fw-bold">GPT Prompt</label>
              <el-input type="textarea" class="form-control" placeholder="You can setup custom prompt here" :rows="5"
                        v-model="gpt_system_prompt" @change="onKeyChange('gpt_system_prompt')"/>
            </div>
          </div>
        </div>
      </div>

      <div class="col-12 col-lg-6 mb-4">
        <div class="card h-100">
          <div class="card-header bg-success text-white">
            <h2 class="mb-0">Azure Speech Recognition</h2>
          </div>
          <div class="card-body">
            <div class="mb-3 text-muted small">
              We use Microsoft Azure's speech recognition service. You can apply for a free Azure token by referring to <a
                :href="azure_application_url" target="_blank">this tutorial</a>:
            </div>
            <div class="mb-3">
              <label class="form-label fw-bold">Azure token</label>
              <el-input class="form-control" placeholder="Input Your Azure Speech Resource Token (KEY 1)" v-model="azure_token"
                        @change="onKeyChange('azure_token')" />
            </div>
            <div class="mb-3">
              <label class="form-label fw-bold">Location/Region</label>
              <el-input class="form-control" placeholder="e.g. eastasia" v-model="azure_region" @change="onKeyChange('azure_region')" />
            </div>
            <div class="mb-3">
              <label class="form-label fw-bold">Recognition Language</label>
              <el-input class="form-control" placeholder="e.g. en-US" v-model="azure_language" @change="onKeyChange('azure_language')" />
            </div>
            <div class="desc_text">
              <span style="text-decoration: gray">zh-CN</span> for Chinese, See <a :href="full_language_codec_url"
                                                                                   target="_blank">here</a> for
              other language codes
            </div>
          </div>
        </div>
      </div>
    </div>
  

    <!-- Uncomment to enable reset button -->
    <!--
    <div class="text-end">
      <el-button class="btn btn-warning" @click="toDef">Set all settings to default</el-button>
    </div>
    -->
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
      github_url: "https://github.com/interview-copilot/Interview-Copilot",
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


.separator {
  margin-top: 10px;
}


.desc_text {
  color: gray;
  font-size: small;
  margin-bottom: 3px;
}

.card {
  box-shadow: 0 2px 8px rgba(0,0,0,0.05);
}

.form-label {
  font-weight: 600;
}

</style>
