# Interview Assistant

> Utilizing GPT and Microsoft Azure's cutting-edge technology to assist interviewers

**Online Demo:** https://interview-copilot.github.io

This web application captures audio from the microphone, uses Microsoft Azure's speech recognition service to obtain a transcript, and then invokes GPT to generate an answer.

![](./docs/imgs/2_demo/demo.png)

## Why This Project?

Although there are excellent interview assistance programs on GitHub (like [cheetah](https://github.com/leetcode-mafia/cheetah)), they can only run on specific desktop platforms and are difficult to install or extend.

This project is built with pure web technology. It does not need installation and can run in the browser of desktop, tablet, or smartphones. It is **serverless**—API Token information is stored locally in your browser.

|                                                      | Windows | Mac  | Pad \ Phone |
| ---------------------------------------------------- | ------- | ---- | ----------- |
| [cheetah](https://github.com/leetcode-mafia/cheetah) |         | ✅    |             |
| [ecoute](https://github.com/SevaSk/ecoute)           | ✅       |      |             |
| This Project                                         | ✅       | ✅    | ✅           |

## Setting Instruction

To use GPT, you need an Open AI API Token: https://platform.openai.com

To use Azure's speech recognition service, you can apply for a free token by referring to [this tutorial](./docs/azure_speech_service_tutorial.md).

## Developing
This project is based on Vue2. Just `cd app`:

**install packages:** `npm install`

**develop:** `npm run serve`

**build:** `npm run build`
