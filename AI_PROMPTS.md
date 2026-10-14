# Generative AI Disclosure

Using AI tools for this challenge is allowed. Be honest here — an accurate
"I used AI heavily" is worth more to us than an inaccurate "I did not".

**Did you use AI tools?** (yes / no): yes

**Which ones?** (e.g. ChatGPT, Claude, Copilot, Cursor): Claude

> If you answered **no**, write "N/A" in sections 1 and 2 and skip to
> section 3. That is a perfectly good answer.

---

## 1. Key prompts

List the main prompts you used. Paraphrasing is fine.

1. `Why do my tests get 502 Bad Gateway from Nginx if I don't use depends_on with condition: service_healthy in Docker Compose?`
2. `My GitHub Actions pipeline fails with permission denied on run-tests.sh. How do I fix the execute permission in Git so it stays fixed after cloning?`
3. `How do I configure nginx:alpine to run as the unprivileged nginx user, and how does that affect the port in compose?`

---

## 2. Where the AI got it wrong

Describe **one concrete case** where the AI's answer was wrong, outdated, or
sub-optimal. Cover: what it suggested, how you noticed it was wrong, and what
you did instead. 

When hardening the proxy to run as an unprivileged user, Claude suggested simply adding USER nginx to the Dockerfile and switching the listening port to 8080. However, when starting the container, Nginx crashed immediately with [emerg] open() "/run/nginx.pid" failed (13: Permission denied) because /var/run on Alpine requires root permissions by default. 
I inspected docker compose logs proxy and realized the model assumed the unprivileged user already had write access to system runtime directories. To resolve this, I added a build step in the Dockerfile that pre-creates /var/run/nginx.pid and grants ownership of /var/run/nginx.pid, /var/cache/nginx, and /etc/nginx to the nginx user before dropping privileges with USER nginx.

> ...

---

## 3. Anything in your submission you could not fully explain?
nothing

We would rather know. Write "nothing" if that is the case.

> ...
