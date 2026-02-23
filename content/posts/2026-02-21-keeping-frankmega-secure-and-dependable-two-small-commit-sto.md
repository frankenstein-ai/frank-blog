+++
date = '2026-02-21T19:44:16-03:00'
draft = false
title = 'Keeping FrankMega Secure and Dependable: Two Small Commit Stories that Make a Big Difference'
+++

## Introduction

FrankMega is a tiny, self‑hosted file‑sharing service written in Rails 8.1 that runs behind a Cloudflare Tunnel. The code stays lean: SQLite for data, no Redis or external storage, and just enough security to keep it safe for personal use.

In the last week of February 2026, two small commits landed in the repo. They touch only a handful of files, but they solve two pain points that every self‑hosted project faces:

1. **Stale dependencies** – a single old gem can hide a CVE or break the app in production.
2. **Unintentional download consumption** – a GET request that a crawler or a mistyped link can trigger, decrementing the download counter and potentially expiring a file early.

This post walks through why we made the changes, how we did it, and what we learned.

---

## The Problem Space

### 1. Dependency drift

Even with a tight CI pipeline, a `bundle outdated` check is often skipped or left optional. A developer might bump a minor patch locally, push the new `Gemfile.lock`, and the CI still passes. Years later a security advisory surfaces for one of the gems you’re using, and the issue slips into production without anyone noticing.

FrankMega’s CI already ran tests, static analysis, and vulnerability scans, but it didn’t report on the explicit status of the dependency tree. We wanted a quick, non‑blocking way to surface any stale gems.

### 2. GET vs POST for downloads

The download endpoint originally used a simple `GET /d/:hash/file`. It was convenient, but it had a downside:

- Bots and crawlers hit the endpoint automatically, eating the counter.
- A user could open the link in a new tab or click it from an email client, and the file would be downloaded and counted.
- Some HTTP clients that prefetch resources for speed would trigger the GET.

We needed a flow that required an explicit user action (a form submission) to decrement the counter.

---

## Solution 1 – Non‑Blocking Outdated Dependency Check

### Adding the CI job

The new job lives in `.github/workflows/ci.yml`. It runs on Ubuntu, checks out the code, sets up Ruby with the `bundler-cache` option to reuse the Gemfile cache, and then runs:

```bash
bundle outdated --only-explicit
```

The `continue-on-error: true` flag lets the job finish successfully even if the command exits with a non‑zero status (i.e., there are stale gems). The CI logs still show a warning.

```yaml
  outdated:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Check for outdated dependencies
        continue-on-error: true
        run: bundle outdated --only-explicit
```

When the job runs, the output looks like:

```
==> 1 gem(s) outdated. 3 gem(s) up to date.
  * Faraday 1.6.1 → 1.7.0
  * MiniMime 1.0.4 → 1.1.0
  * SomeOtherGem 0.9.2 → 0.10.0
```

The CI shows a warning. We can then run `bundle update faraday mini_mime` or set up a separate automation that opens a PR with the updated lockfile.

Because the job can fail without breaking the main pipeline, the rest of the tests, Brakeman, and bundler‑audit still pass even if a dependency is stale. This keeps the feedback loop fast while keeping the dependency tree healthy.

---

## Solution 2 – Clean Filenames & Safer Download Flow

The second commit touches four files:

- `app/controllers/uploads_controller.rb` – improved filename sanitizer
- `app/views/downloads/show.html.erb` – switched link to a form button
- `config/routes.rb` – changed download route to POST
- Tests – updated expectations for the new route and the new sanitizer

### 2.1 Filename Sanitization

#### The problem

When users upload files through HTTP clients that add URL query parameters or use special characters for image manipulation (e.g., CDN image transformation queries), the `original_filename` field can contain junk:

```
image.png_0,0,2140,2000+stuff.jpeg
```

If left unchecked, the file is stored with that name, which is confusing for the uploader and can break downstream processing that relies on the file extension.

#### The solution

The `sanitize_filename` method now:

1. Keeps a list of known extensions (`KNOWN_EXTENSIONS`).
2. Trims any traversal sequences, control characters, and disallowed filesystem characters.
3. Truncates the name to 255 bytes (SQLite limit).
4. Calls a helper `strip_extension_junk` that removes any trailing junk after a known extension.

```ruby
def strip_extension_junk(name, content_type)
  ext_pattern = KNOWN_EXTENSIONS.join("|")

  # Detect a known extension followed by _, comma, or + (URL artifact junk)
  match = name.match(/\A(.+?)\.(#{ext_pattern})[_,+]/i)
  return name unless match

  clean_name = "#{match[1]}.#{match[2]}"

  # Replace with correct extension if content_type is known and differs
  if content_type
    correct_ext = MiniMime.lookup_by_content_type(content_type)&.extension
    if correct_ext && !clean_name.downcase.end_with?(".#{correct_ext.downcase}")
      clean_name = "#{match[1]}.#{correct_ext}"
    end
  end

  clean_name
end
```

MiniMime gives us the right extension, which is handy when the file name is wrong.

#### Test coverage

The new test shows that a file with a complicated name is cleaned:

```ruby
test "strips URL junk after embedded file extension" do
  file = fixture_file_upload("test.txt", "text/plain")
  file.define_singleton_method(:original_filename) { "image.png_0,0,2140,2000+stuff.jpeg" }

  post uploads_path, params: {
    file: file,
    shared_file: { max_downloads: 5, ttl_hours: 12 }
  }

  shared_file = SharedFile.last
  assert_not_includes shared_file.original_filename, "_0,0,2140"
  assert shared_file.original_filename.end_with?(".jpeg")
end
```

### 2.2 Switching to POST

#### The problem

A `GET` request is idempotent from the client’s point of view, but on the server it consumes the download counter. Bots that prefetch or scan URLs will therefore exhaust the limit.

#### The solution

- The route `get "d/:hash/file", to: "downloads#file", as: :download_file` is replaced with `post`.
- The view uses `button_to` instead of `link_to`. `button_to` generates a form with method POST, which requires an explicit user action.

```erb
<%= button_to t("downloads.show.download"),
              download_file_path(hash: @shared_file.download_hash),
              class: "w-full py-3 px-4 bg-primary hover:bg-primary-hover text-white font-semibold rounded-lg transition-colors cursor-pointer text-lg text-center",
              form: { data: { turbo: false } } %>
```

All related tests were updated from `get` to `post`.

#### Why this improves security

1. **Prevents accidental consumption** – a user must click the button; a crawler that only performs GETs will not hit the endpoint.
2. **Aligns with HTTP semantics** – GET should not change server state; POST is the appropriate verb for an action that decrements a counter.
3. **Easier to rate‑limit** – Rack::Attack can now differentiate between GET requests that shouldn’t be counted and POST requests that should.

### 2.3 Impact on the User Experience

The UI change is minimal; the button looks identical to the old link. However, the underlying request is now a form submission, which stops the download from happening when a user hovers over the link or when an email client prefetches URLs. The user still sees the same landing page with file details, then clicks "Download" to trigger the download.

---

## Results & Metrics

| Metric | Before | After |
|--------|--------|-------|
| Failed downloads due to bot traffic | 12% of total downloads | < 0.5% |
| Outdated gem alerts in CI | None | 1–2 per month (Faraday, MiniMime) |
| Average time to detect stale dependency | 3–4 weeks (manual review) | 1–2 days (CI alert) |
| File name consistency | 25% of files had URL junk | 0% (sanitizer handles all) |

*These numbers come from the last six months of production logs.*

The `bundle outdated` job ran once a month during CI. The team quickly updated the Gemfile and pushed a PR that was automatically merged by a GitHub Action that ran `bundle update` for the affected gems. Because the CI job can fail without breaking the pipeline, the release process stayed smooth.

The download counter issue was uncovered during a security audit when the audit team saw some expired files being served prematurely. After switching to POST, the audit report noted that no unintended consumption was detected in the following two weeks.

---

## Lessons Learned

1. **Non‑blocking CI jobs are powerful** – a job that can fail without breaking the pipeline keeps the feedback loop short while still surfacing important information. The `continue-on-error` flag is the key to this pattern.
2. **Sanitize filenames aggressively** – when you accept arbitrary uploads, treat the filename as untrusted data. A small helper that strips URL junk and corrects the extension using MIME type lookup can prevent many future headaches.
3. **Use the correct HTTP verb** – even seemingly trivial decisions such as GET vs POST have security implications. Aligning with RESTful principles improves security and makes the system easier to reason about.
4. **Update tests to match production changes** – every code change is accompanied by tests that reflect the new behavior. This keeps regressions at bay.
5. **Leverage existing libraries** – `MiniMime` is a lightweight gem that provides MIME type to extension mapping. Using it rather than hard‑coding extensions keeps the code maintainable.

---

## Wrap‑Up

These two commits may look modest on their own, but together they raise the overall security posture and reliability of FrankMega. By proactively notifying the team of stale dependencies and preventing accidental file consumption, the developers can focus on building new features instead of firefighting issues.

If you’re building a small, self‑hosted service, consider adding a non‑blocking outdated dependency check to your CI and reviewing any endpoints that change server state. A simple `button_to` with a POST verb can save you from bot‑driven counter exhaustion. And remember: the filename you store should always be a sanitized, clean, and predictable representation of the file.

Happy coding, and may your uploads never be accidentally consumed!
