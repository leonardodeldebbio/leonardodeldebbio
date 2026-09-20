<!DOCTYPE html>
<html lang="it">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Leonardo — terminal</title>
<style>
  :root {
    --bg: #cfd3d8;
    --window-radius: 10px;
    --chrome-bg: #e0e8f0;
    --btn-green: #3bb662;
    --btn-yellow: #e5c30f;
    --btn-red: #e75448;
    --term-bg: #1b1d1f;
    --term-fg: #f0f0f0;
    --term-gray: #8a8f98;
    --term-green: #3ecf6b;
    --term-cyan: #6ad3e8;
    --prompt-color: #6ad3e8;
    box-sizing: border-box;
    padding-top: env(safe-area-inset-top, 0px);
    padding-bottom: env(safe-area-inset-bottom, 0px);
  }
  @media (prefers-color-scheme: dark) {
    :root:not([data-theme="light"]) {
      --bg: #14161a;
    }
  }
  :root[data-theme="dark"] {
    --bg: #14161a;
  }
  html { scroll-padding-top: env(safe-area-inset-top, 0px); height: 100%; }
  * { box-sizing: border-box; }
  body {
    height: 100%;
    margin: 0;
    background: var(--bg);
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  }

  .terminal-window {
    width: 100%;
    max-width: 680px;
    background: var(--term-bg);
    border-radius: var(--window-radius);
    box-shadow: 0 20px 60px rgba(0,0,0,0.45), 0 2px 8px rgba(0,0,0,0.3);
    overflow: hidden;
  }

  header.chrome {
    background: var(--chrome-bg);
    height: 34px;
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 0 12px;
  }

  header.chrome .button {
    width: 12px;
    height: 12px;
    border-radius: 50%;
  }
  .button.green { background: var(--btn-green); }
  .button.yellow { background: var(--btn-yellow); }
  .button.red { background: var(--btn-red); }

  header.chrome .title {
    flex: 1;
    text-align: center;
    font-size: 12px;
    color: #55606e;
    font-weight: 600;
    margin-right: 40px;
    user-select: none;
  }

  section.terminal {
    color: var(--term-fg);
    font-family: Menlo, Monaco, "Consolas", "Courier New", monospace;
    font-size: 14px;
    line-height: 1.6;
    padding: 18px 16px;
    height: 420px;
    overflow-y: auto;
    white-space: pre-wrap;
    word-break: break-word;
  }

  section.terminal::-webkit-scrollbar { width: 8px; }
  section.terminal::-webkit-scrollbar-thumb { background: #3a3d42; border-radius: 4px; }

  .line { margin: 0 0 2px 0; }
  .prompt-line .prompt-sign { color: var(--prompt-color); }
  .gray { color: var(--term-gray); }
  .green { color: var(--term-green); }
  .quote { color: var(--term-fg); font-style: italic; }
  .heading { color: var(--term-cyan); font-weight: bold; }

  .cursor {
    display: inline-block;
    width: 8px;
    height: 1em;
    background: var(--term-fg);
    vertical-align: text-bottom;
    margin-left: 2px;
    animation: blink 0.85s steps(1) infinite;
  }

  @keyframes blink {
    0%, 49% { opacity: 1; }
    50%, 100% { opacity: 0; }
  }
</style>
</head>
<body>

  <div class="terminal-window">
    <header class="chrome">
      <div class="button green"></div>
      <div class="button yellow"></div>
      <div class="button red"></div>
      <div class="title">leonardo — bash — 80x24</div>
    </header>
    <section class="terminal" id="terminal"></section>
  </div>

<script>
(function () {
  const term = document.getElementById('terminal');

  // Each step: a command to "type", followed by output lines that appear once typing finishes.
  const script = [
    {
      cmd: "whoami",
      output: [
        { text: "It's me, Leonardo", cls: "heading" }
      ]
    },
    {
      cmd: "cat about.txt",
      output: [
        { text: "Welcome to my profile, I'm Leonardo, an occasional developer" },
        { text: "and student based in Italy. I usually work on open source" },
        { text: "projects in my freetime." }
      ]
    },
    {
      cmd: "cat currently.txt",
      output: [
        { text: "What I'm working on", cls: "heading" },
        { text: "" },
        { text: "I'm currently working on many projects I really hope you'll" },
        { text: "see one day." }
      ]
    },
    {
      cmd: "cat contacts.txt",
      output: [
        { text: "Contacts", cls: "heading" },
        { text: "" },
        { text: "You can contact me through my email if you want to collab" },
        { text: "on a project." }
      ]
    },
    {
      cmd: "fortune",
      output: [
        { text: "\u201cThe people who are crazy enough to think they can change", cls: "quote" },
        { text: "the world are the ones who do\u201d", cls: "quote" },
        { text: "  \u2014 Steve Jobs", cls: "gray" }
      ]
    }
  ];

  let stepIndex = 0;
  let charIndex = 0;
  const typeSpeed = 45;
  const postDelay = 900;

  function appendLine(html, cls) {
    const div = document.createElement('div');
    div.className = 'line' + (cls ? ' ' + cls : '');
    div.innerHTML = html;
    term.appendChild(div);
    scrollToBottom();
  }

  function scrollToBottom() {
    term.scrollTop = term.scrollHeight;
  }

  // The live prompt line being typed into
  let currentPromptLine = null;
  let currentPromptSpan = null;

  function startPromptLine() {
    currentPromptLine = document.createElement('div');
    currentPromptLine.className = 'line prompt-line';
    currentPromptLine.innerHTML = '<span class="prompt-sign">$</span> <span class="typed"></span><span class="cursor"></span>';
    term.appendChild(currentPromptLine);
    currentPromptSpan = currentPromptLine.querySelector('.typed');
    scrollToBottom();
  }

  function typeStep() {
    if (stepIndex >= script.length) {
      // idle blinking prompt at the end
      startPromptLine();
      return;
    }

    const step = script[stepIndex];

    if (charIndex === 0) {
      startPromptLine();
    }

    if (charIndex < step.cmd.length) {
      currentPromptSpan.textContent += step.cmd.charAt(charIndex);
      charIndex++;
      setTimeout(typeStep, typeSpeed);
    } else {
      // finished typing this command: remove the blinking cursor from this line
      const cur = currentPromptLine.querySelector('.cursor');
      if (cur) cur.remove();

      setTimeout(function () {
        step.output.forEach(function (o) {
          appendLine(o.text === "" ? "&nbsp;" : escapeHtml(o.text), o.cls);
        });
        stepIndex++;
        charIndex = 0;
        setTimeout(typeStep, postDelay);
      }, 300);
    }
  }

  function escapeHtml(str) {
    return str
      .replace(/&/g, "&amp;")
      .replace(/</g, "&lt;")
      .replace(/>/g, "&gt;");
  }

  typeStep();
})();
</script>

</body>
</html>
