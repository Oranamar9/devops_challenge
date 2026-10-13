# Generative AI Disclosure

Using AI tools for this challenge is allowed. Be honest here — an accurate
"I used AI heavily" is worth more to us than an inaccurate "I did not".

**Did you use AI tools?** (yes / no): Yes

**Which ones?** (e.g. ChatGPT, Claude, Copilot, Cursor): Claude

> If you answered **no**, write "N/A" in sections 1 and 2 and skip to
> section 3. That is a perfectly good answer.

---

## 1. Key prompts

List the main prompts you used. Paraphrasing is fine.

1. "Here's part of the log Docker gave me. It looks like it continued with some default setting, so I don't think this was the problem. Correct me if I'm wrong."
(followed by the full build log)

2. "nginx is up and the backend works on its own, but requests through the proxy return 502. Help me analyze these proxy logs and find where the connection between them fails."
(followed by the proxy logs showing "connect() failed (111: Connection refused) while connecting to upstream")

3. "The tests fail because they run before the backend has finished loading. Can I add a healthcheck for the backend in docker-compose.yml, so the tests won't start until it's fully up?"

4. "CI fails with 'Permission denied' (exit code 126) on ./scripts/run-tests.sh. If I run chmod +x on the script and push, will that fix it for CI and for anyone who clones the repo?"

---

## 2. Where the AI got it wrong

Describe **one concrete case** where the AI's answer was wrong, outdated, or
sub-optimal. Cover: what it suggested, how you noticed it was wrong, and what
you did instead. 3–5 sentences.

> The AI suggested nginx:1.28-alpine as the base image. When I checked nginx.org, I saw the current stable version is 1.30.5, and nginx only supports the latest stable branch, so 1.28 no longer gets security fixes. I switched to nginx:1.30.5-alpine.

---

## 3. Anything in your submission you could not fully explain?

We would rather know. Write "nothing" if that is the case.

> ...
