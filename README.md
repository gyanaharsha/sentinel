# Sentinel

Check whether a password appears in real breach data — without the password ever leaving your browser.

**Live:** https://gyanaharsha.github.io/sentinel/

## Why I built this

I kept running into the same statistic while preparing for security work: stolen credentials are the single most common way breaches start (22% of confirmed breaches in the 2025 Verizon DBIR), largely because people reuse passwords — Cybernews put reuse at 94% across 19 billion leaked passwords they analyzed.

The fix everyone recommends is "check your passwords against breach data." But that advice has a built-in contradiction: sending your password to a checker site is exactly the kind of exposure you're trying to avoid. When I learned how Have I Been Pwned's range API resolves this with k-anonymity, it was one of those ideas that's obvious only after someone shows it to you. I wanted to build it myself, and make the protocol *visible* on the page instead of hidden in a network call.

My background is cryptography and cybersecurity (MS CS, Syracuse), and this doubles as a working demo of concepts I otherwise only had on paper: hashing properties, range queries, entropy, side channels.

## How the check works

1. The password is hashed with SHA-1 **in the browser** (Web Crypto API). Nothing is transmitted yet.
2. Only the **first 5 characters** of the 40-character hash go to `api.pwnedpasswords.com/range/{prefix}`. Roughly 800 leaked hashes share any given prefix, so the server can't tell which one — or whether any — is mine. That crowd of ~800 is the "k" in k-anonymity.
3. The API returns every suffix in that range; the match happens **client-side**.

The page renders this visually: the 5 departing characters highlighted, the 35 staying characters marked "never leave this page." Open DevTools → Network during a check and you can verify the claim — the request contains five hex characters and nothing else.

I also send `Add-Padding: true`, which makes all API responses roughly the same size, so response length can't leak information to anyone observing traffic volume.

## The strength panel

"Not breached" isn't the same as "strong," so a second panel analyzes the password itself:

- **Entropy** — `length × log2(charset)`, in bits. Length dominates; each character multiplies the search space.
- **Crack-time estimate** — against an offline GPU rig at ~10B guesses/sec.
- **Pattern flags** — the stuff crackers try first because humans are predictable: capital-first-letter-plus-trailing-digit (the pattern most people use), years, keyboard walks, sequences, repeats. A password matching these is much weaker than its raw entropy suggests.

## Notes on choices

- **Why SHA-1 if it's broken?** It's broken for collision resistance, which isn't a property this design relies on. It's a bucketing identifier for the range lookup, and HIBP publishes the dataset as SHA-1. Right tool for this specific job.
- **Single HTML file, no framework.** Deliberate. I can account for every line, it deploys anywhere static files are served, and there's no dependency chain to trust.
- **Nothing is stored.** No analytics, no logging, no backend of mine at all.

## Run it locally

```bash
cd site && python3 -m http.server 8000
# open http://localhost:8000  (crypto.subtle needs https or localhost)
```

## Also deployable on AWS

The `terraform/` folder deploys the same site the way production sites ship: private S3 origin behind CloudFront with Origin Access Control, HTTPS via the default CloudFront cert. `terraform init && terraform apply`. GitHub Pages is the canonical home; the AWS path exists because I wanted to build the origin-access pattern myself.

## What's in v2 (Jul 2026)

- **Live breach-data chart.** Every k-anonymity check already downloads ~800 real (suffix, count) records. v2 charts that range — sorted counts on a log scale. Password popularity is heavy-tailed / Zipfian (Wang et al., *Zipf's Law in Passwords*, IEEE TIFS 2017), and you can see it in every single range. If your password is breached, its bar is highlighted. No datasets shipped, nothing fabricated — it's the actual API response, visualized.
- **NIST section.** SP 800-63B Rev. 4 (finalized 2025) now *mandates* blocklist screening and prohibits composition rules — length-first. The strength flags were rewritten to match (15-char single-factor minimum instead of complexity worship).
- **Generator.** `crypto.getRandomValues` with rejection sampling to avoid modulo bias; lookalike characters (l/I/1/O/0) dropped deliberately. Honest entropy readout.
- **Redesign.** v1 was a dark theme with a glowing accent — which, fair, is what every security side-project looks like. v2 is light, system fonts, one blue.

## Roadmap

- [ ] Full public breach-corpus analysis with pandas (length/entropy distributions, top patterns)
- [ ] Hindi/Telugu localization

## Credits

Breach data: [Have I Been Pwned](https://haveibeenpwned.com/Passwords) Pwned Passwords (free range API, thanks to Troy Hunt). k-anonymity range-query design originally by Cloudflare + HIBP.
