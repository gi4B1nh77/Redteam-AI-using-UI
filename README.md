# Redteam-AI-using-UI
An end-to-end security evaluation framework that simulates real-world adversary behavior. By integrating Promptfoo for automated prompt injection and database management with Playwright for browser-based interaction, the system bypasses traditional API testing to evaluate the AI's safety and robustness directly through the final user interface.

Reference: https://www.promptfoo.dev/docs/installation/

### 1.	Initial Configuration
- Create new user:
~~~bash
sudo adduser pentester
sudo usermod -aG sudo pentester
su – pentester
sudo apt update
~~~
- Download and Install NVM:
~~~bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
~~~
- Install latest version of Node.js (LTS):
~~~bash
nvm install 20
nvm use 20
~~~
###2. Install Promptfoo
~~~bash
mkdir ~/promptfoo_project && cd ~/promptfoo_project
npm init -y
npm install promptfoo
~~~
- Init and check:
~~~bash
npx promptfoo init
~~~
###3. Install Playwright in Promptfoo (This will help you redteam AI by acting as USER, ex: Chatbox,...)
- Use provider:
~~~bash
npm install playwright @playwright/browser-chromium plywright-extra puppeteer-extra-plugin-stealth
~~~
- Use Target (Recommended): Create a .js file in project then guide the Promptfoo aim to this file, the purpose is intergate Playwright with Prompfoo, this avoid the account login phase in many chatbot
~~~bash
npm install -D @playwright/test
npx playwright install chromium
~~~
- Create a .js file, in my project it will be named healthcare-ui.js in the folder providers, I named providers because it will act as the provider but use differnet method to approach the chatbox:
<img width="905" height="131" alt="image" src="https://github.com/user-attachments/assets/98505994-da92-4795-bbda-4afda2d1e752" />

~~~bash

/**
 * Promptfoo UI provider for Genie Healthcare Agent (Playwright)
 * Robust approach:
 * - Login once (storageState.json)
 * - For each prompt:
 *   - send prompt
 *   - wait for user message to appear
 *   - wait up to 120s until assistant output "settles"
 *   - return the FINAL last assistant bubble (not placeholder)
 *
 * Install in promptfoo_project:
 *   npm i -D @playwright/test
 *   npx playwright install chromium
 *
 * ENV:
 *   MS_EMAIL, MS_PASSWORD
 */

const fs = require("fs");
const path = require("path");
const { chromium } = require("@playwright/test");

// ===================== CONFIG =====================
const BASE_URL = "You url to the chatbox";
const CHAT_URL = `${BASE_URL}/chat`;

const MS_EMAIL = process.env.MS_EMAIL || "Enter your email to login chatbox";
const MS_PASSWORD = process.env.MS_PASSWORD || "Password";

const TIMEOUT_MS = 180000;
const ANSWER_WAIT_MS = 180000; // you asked 120s
const STABLE_MS = 5000;        // stable window
const POST_LOGIN_WAIT_MS = 15000;

const STORAGE_STATE_PATH = path.join(process.cwd(), "storageState.json");

// ===================== SELECTORS (based on your screenshot) =====================
const SEL = {
  // Microsoft SSO
  email: "input#i0116",
  password: "input#i0118",
  nextOrYes: "#idSIButton9",
  no: "#idBtn_Back",
  proofUpRedirect: "#idSubmit_ProofUp_Redirect",
  skipSetupLink: 'button.ms-Link:has-text("Skip setup")',

  // Chat
  chatInput: 'textarea[id^="textarea-"]',

  // User message text in DOM (your screenshot shows .message-text inside user bubble)
  userMsgText: "div.msg.user div.message-text",

  // Assistant bubble (you already used this)
  assistantBubbles: "div.msg.assistant div.bubble",
};

// Placeholder / "still thinking" texts to ignore
const THINKING_PATTERNS = [
  "đang phân tích",
  "đang suy nghĩ",
  "đang xử lý",
  "đang kiểm tra",
  "tổng hợp",
  "đang truy vấn",
  "analyzing",
  "analysis",
  "thinking",
  "processing",
  "querying",
  "querying data",
  "synthesizing",
  "working on it",
  "please wait",
];

// ===================== HELPERS =====================
function isAuthRedirect(url) {
  return (
    url.includes("login.microsoftonline.com") ||
    url.includes("/MicrosoftIdentity/Account/SignIn") ||
    url.includes("signin-oidc")
  );
}

function deleteStorageStateIfExists() {
  try {
    if (fs.existsSync(STORAGE_STATE_PATH)) fs.unlinkSync(STORAGE_STATE_PATH);
  } catch {}
}

function isPlaceholder(text) {
  const t = String(text || "").trim().toLowerCase();
  if (!t) return true;
  return THINKING_PATTERNS.some((p) => t.includes(p));
}

async function clickIfVisible(page, selector, timeout = 3000) {
  const loc = page.locator(selector).first();
  try {
    await loc.waitFor({ state: "visible", timeout });
    await loc.click({ timeout });
    return true;
  } catch {
    return false;
  }
}

async function fillAndEnterIfVisible(page, selector, text, timeout = 15000) {
  const loc = page.locator(selector).first();
  try {
    await loc.waitFor({ state: "visible", timeout });
    await loc.fill(text);
    await loc.press("Enter");
    return true;
  } catch {
    return false;
  }
}

async function waitForChatReady(page, timeout = TIMEOUT_MS) {
  await page.waitForSelector(SEL.chatInput, { timeout });
}

// ===================== LOGIN ONCE =====================
async function loginOnceAndSaveState(browser) {
  if (fs.existsSync(STORAGE_STATE_PATH)) return STORAGE_STATE_PATH;

  const context = await browser.newContext();
  const page = await context.newPage();
  page.setDefaultTimeout(TIMEOUT_MS);

  await page.goto(BASE_URL, { waitUntil: "domcontentloaded", timeout: TIMEOUT_MS });

  await fillAndEnterIfVisible(page, SEL.email, MS_EMAIL, 25000);
  await clickIfVisible(page, SEL.nextOrYes, 8000);
  await fillAndEnterIfVisible(page, SEL.password, MS_PASSWORD, 25000);

  await clickIfVisible(page, SEL.proofUpRedirect, 12000);
  await clickIfVisible(page, SEL.skipSetupLink, 12000);

  await clickIfVisible(page, SEL.nextOrYes, 12000);
  await clickIfVisible(page, SEL.no, 4000);

  await page.goto(CHAT_URL, { waitUntil: "domcontentloaded", timeout: TIMEOUT_MS });

  if (isAuthRedirect(page.url())) {
    await context.close();
    throw new Error(`Auth redirect during loginOnce: ${page.url()}`);
  }

  await waitForChatReady(page, TIMEOUT_MS);

  // ✅ wait 10s after login (your requirement)
  await page.waitForTimeout(POST_LOGIN_WAIT_MS);

  await context.storageState({ path: STORAGE_STATE_PATH });
  await context.close();
  return STORAGE_STATE_PATH;
}

// ===================== CORE: WAIT FINAL ANSWER (settle) =====================
async function waitFinalAssistantAnswer(page, beforeAssistantCount) {
  const start = Date.now();
  let lastSignature = "";
  let lastChangeAt = Date.now();

  while (Date.now() - start < ANSWER_WAIT_MS) {
    const count = await page.locator(SEL.assistantBubbles).count();

    // Need at least 1 new assistant bubble compared to before sending prompt
    if (count <= beforeAssistantCount) {
      await page.waitForTimeout(3000);
      continue;
    }

    const lastText = await page.locator(SEL.assistantBubbles).last().innerText().catch(() => "");
    const normalized = String(lastText || "").trim();

    // signature changes if count changes or last text changes
    const sig = `${count}::${normalized}`;

    if (sig !== lastSignature) {
      lastSignature = sig;
      lastChangeAt = Date.now();
      await page.waitForTimeout(300);
      continue;
    }

    // stable long enough
    if (Date.now() - lastChangeAt >= STABLE_MS) {
      // still placeholder? then keep waiting
      if (isPlaceholder(normalized)) {
        await page.waitForTimeout(5000);
        continue;
      }
      return normalized;
    }

    await page.waitForTimeout(2500);
  }

  throw new Error("Timeout: assistant output did not settle within 120s");
}

// Wait until user message containing our prompt appears (to confirm send succeeded)
async function waitUserEcho(page, promptText, timeoutMs = 60000) {
  const raw = String(promptText || "")
    .trim()
    .replace(/\s+/g, " ");

  const needle = raw.slice(0, 30).toLowerCase();

  await page.waitForFunction(
    ({ needle }) => {
      const norm = (s) =>
        String(s || "").trim().replace(/\s+/g, " ").toLowerCase();

      const selectors = [
        "div.msg.user div.message-text",
        "div.msg.user .bubble",
        "div.msg.user",
      ];

      for (const sel of selectors) {
        const nodes = Array.from(document.querySelectorAll(sel));
        for (let i = nodes.length - 1; i >= 0; i--) {
          const t = norm(nodes[i].innerText);
          if (t && t.includes(needle)) return true;
        }
      }
      return false;
    },
    { needle },
    { timeout: timeoutMs }
  );
}

// ===================== PROMPTFOO PROVIDER =====================
class HealthcareUiProvider {
  constructor(options = {}) {
    this.providerId = options.id || "healthcare-agent-ui";
    this.config = options.config || {};
    this.headless = this.config.headless !== false;
    this.timeoutMs = Number(this.config.timeoutMs || TIMEOUT_MS);
  }

  id() {
    return this.providerId;
  }

  async callApi(prompt) {
    const browser = await chromium.launch({ headless: this.headless });

    try {
      // Ensure login state (self-heal if expired)
      try {
        await loginOnceAndSaveState(browser);
      } catch {
        deleteStorageStateIfExists();
        await loginOnceAndSaveState(browser);
      }

      let ctx = await browser.newContext({ storageState: STORAGE_STATE_PATH });
      let page = await ctx.newPage();
      page.setDefaultTimeout(this.timeoutMs);

      await page.goto(CHAT_URL, { waitUntil: "domcontentloaded", timeout: this.timeoutMs });

      // If redirected to auth, self-heal once
      if (isAuthRedirect(page.url())) {
        await ctx.close();
        deleteStorageStateIfExists();
        await loginOnceAndSaveState(browser);

        ctx = await browser.newContext({ storageState: STORAGE_STATE_PATH });
        page = await ctx.newPage();
        page.setDefaultTimeout(this.timeoutMs);
        await page.goto(CHAT_URL, { waitUntil: "domcontentloaded", timeout: this.timeoutMs });
      }

      // Robust: wait chat ready with retry + late-auth-detect
const CHAT_INPUT_TIMEOUT = 180000;

async function ensureChatReady(p) {
  if (isAuthRedirect(p.url())) return false;

  try {
    await p.waitForSelector(SEL.chatInput, { timeout: CHAT_INPUT_TIMEOUT });
    return !isAuthRedirect(p.url());
  } catch {
    return false;
  }
}

// 1) attempt #1
let ok = await ensureChatReady(page);

if (!ok) {
  console.error("[healthcare-ui] chat input not ready, url=", page.url());

  // reload once
  try {
    await page.reload({ waitUntil: "domcontentloaded", timeout: this.timeoutMs });
  } catch {}

  ok = await ensureChatReady(page);
}

// 2) if still not ok -> self-heal (delete state, login again) then final attempt
if (!ok) {
  await ctx.close();
  deleteStorageStateIfExists();
  await loginOnceAndSaveState(browser);

  ctx = await browser.newContext({ storageState: STORAGE_STATE_PATH });
  page = await ctx.newPage();
  page.setDefaultTimeout(this.timeoutMs);

  await page.goto(CHAT_URL, { waitUntil: "domcontentloaded", timeout: this.timeoutMs });

  // final attempt
  const finalOk = await ensureChatReady(page);
  if (!finalOk) {
    // dump url/title for root cause
    const title = await page.title().catch(() => "");
    throw new Error(`Chat UI not ready after retry+relogin. url=${page.url()} title=${title}`);
  }
}


      // Give UI a short settle even in reused session (reduces late welcome injection)
      await page.waitForTimeout(2000);

      const chatInput = page.locator(SEL.chatInput).first();
      await chatInput.waitFor({ state: "visible", timeout: this.timeoutMs });

      // Snapshot assistant count BEFORE sending prompt
      const beforeAssistant = await page.locator(SEL.assistantBubbles).count();

      const text = String(prompt ?? "");
      await chatInput.fill(text);
      await chatInput.press("Enter");

      // Confirm user message posted (prevents reading preloaded welcome)
      await waitUserEcho(page, text, 90000);

      // Wait until assistant output settles; return FINAL last assistant bubble
      const output = await waitFinalAssistantAnswer(page, beforeAssistant);

      await ctx.close();
      return { output };
    } finally {
      await browser.close();
    }
  }
}

module.exports = HealthcareUiProvider;
module.exports.default = HealthcareUiProvider;
~~~

  
