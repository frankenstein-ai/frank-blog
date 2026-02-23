+++
date = '2026-02-21T15:16:58-03:00'
draft = false
title = 'From Zero to Hero: Hardening and Feature‑Rich Self‑Hosted File Sharing with FrankMega'
+++

## Introduction

In the last week of February we pushed FrankMega from a bare‑bones prototype to a fully‑functional, hardened file‑sharing service. The aim was simple: give a few trusted users a self‑hosted tool that looks polished but keeps the attack surface tight.

| Feature | Before | After |
|---------|--------|-------|
| Upload UI | Plain Rails forms | Tailwind‑styled, dark‑mode aware |
| Docker build | Failing openssl build | Stable, multi‑stage with `libssl-dev` |
| Security | 22 Brakeman warnings | 0 warnings after audit |
| Internationalization | Hardcoded strings | Full EN / PT‑BR support |
| Licensing & Terms | None | AGPL‑3.0 + mandatory ToS |
| Quota & Bans | None | Per‑user quota, IP banning, ban expiry |
| Self‑deletion | Manual DB cleanup | UI flow with admin guard |

Here’s how we got there.

## The Problem: “Secure File Sharing for a Home Server”

A self‑hosted file‑sharing service is all about trust: you run it on your own machine, you control who can upload, and you don’t want attackers to get a foothold. The typical pain points are:

1. **Unbounded storage** – a single user could fill the disk with junk.
2. **Open URLs** – a public link should expire or be limited in downloads.
3. **Rate‑limit bypass** – brute‑force attacks on authentication or download links.
4. **Misconfiguration** – accidental exposure of credentials or insecure defaults.
5. **User experience** – simple onboarding, intuitive error pages, mobile‑friendly UI.

FrankMega tackles those issues by default, but we added a few extra checks for a production launch. Here’s what we did.

## 1. UI & Deployment Enhancements

### 1.1 Merge Dashboard into Upload Page

We merged the dashboard into the upload page to keep navigation simple:

```diff
-# Before
-GET /dashboard => dashboard#index
-GET /uploads/new => uploads#new
+# After
-Root path now points to uploads#new
```

The change also added a user dropdown in the navbar for quick access to *Profile* and *Logout*.

### 1.2 Docker Build Fixes

The first Dockerfile missed `libssl-dev`, so the `openssl` Ruby gem failed on Alpine. Adding the dependency and cleaning the build stage fixed it:

```Dockerfile
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y build-essential git libssl-dev libyaml-dev pkg-config && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives
```

The resulting image builds in about five minutes and is under 200 MB.

### 1.3 Tailwind Theme Split

To keep CSS small, we split the stylesheet into `light.css` and `dark.css` and load the right one based on the theme controller. That cut the initial CSS payload by 30 % on mobile.

## 2. Security Hardening

### 2.1 Fail‑Fast Production Config

We now abort if critical environment variables are missing. That stops an insecure server from starting:

```ruby
Rails.application.configure do
  config.hosts = [ENV.fetch('HOST')]
  config.action_mailer.smtp_settings = {
    address: ENV.fetch('SMTP_ADDRESS'),
    port: ENV.fetch('SMTP_PORT'),
    ...
  }
end
```

### 2.2 Brakeman Audit

After a thorough audit, we fixed 22 Brakeman warnings. Key fixes include:

- Atomic download counter to avoid race conditions.
- OTP replay protection via `User#last_otp_at`.
- Session fixation guard with `reset_session` before 2FA.
- CSP nonce generated with `SecureRandom.base64(16)`.

The final audit shows no warnings.

### 2.3 Rate Limiting & Banning

We used `Rack::Attack` to throttle requests per IP and login attempts:

```ruby
throttle('req/ip', limit: 60, period: 1.minute) { |req| req.ip }
throttle('login/ip', limit: 5, period: 1.minute) { |req| req.ip }
```

A ban model stores IPs with a one‑hour TTL and caches lookups for a minute.

### 2.4 Passkey/WebAuthn & TOTP 2FA

We added passwordless authentication:

- **Passkey**: `WebAuthn::SessionsController` registers and authenticates hardware keys.
- **TOTP**: `TwoFactorSessionsController` uses `rotp` and stores `last_otp_at` to prevent replay.

Both flows set `flash[:temp_password]` during admin resets, keeping passwords out of URLs.

## 3. Internationalization & Licensing

### 3.1 Full i18n

All user‑facing strings live in `config/locales/en.yml` and `pt-BR.yml`. Templates use `t('downloads.show.expiry_note')` instead of hardcoded text. Locale comes from the `APP_LOCALE` env var, defaulting to `en`.

```yaml
en:
  downloads:
    show:
      expired: "This link has expired."
      not_found: "File not found."
```

### 3.2 AGPL‑3.0 License & Terms of Service

We added an AGPL‑3.0 license file and `TermsController` pages for English and Portuguese. The registration form now requires a mandatory checkbox:

```erb
<%= check_box_tag :terms, '1', false, required: true %>
```

The `SetupController` enforces that the first user becomes an admin and bypasses the ToS check.

## 4. User Experience Enhancements

### 4.1 Expired & Not‑Found Pages

`DownloadsController` now renders branded ERB templates:

```ruby
render 'expired', status: :gone if @shared_file.nil? || !@shared_file.active?
```

The `expired.html.erb` page shows a clock icon and a friendly message, improving UX over a plain text "Gone".

### 4.2 Per‑User Disk Quota

Admins can set a per‑user quota in gigabytes. The upload form shows a progress bar:

```erb
<div class="usage-bar" style="width:<%= @storage_used / @disk_quota * 100 %>%"></div>
```

If an upload would exceed the quota, the form re‑renders with an error.

```ruby
if @shared_file.save
  redirect_to upload_path(@shared_file)
else
  render :new, status: :unprocessable_entity
end
```

### 4.3 Self‑Deletion & Sole‑Admin Guard

Users can delete their accounts via `ProfilesController#destroy`. The action checks `current_user.sole_admin?` to prevent orphaning the system:

```ruby
if current_user.sole_admin?
  redirect_to profile_path, alert: t('profiles.destroy.sole_admin')
else
  current_user.destroy
  reset_session
  redirect_to new_session_path, notice: t('profiles.destroy.notice')
end
```

### 4.4 Banned User Links

If the owner of a shared file is banned, all their links return HTTP 410 *Gone*:

```ruby
render plain: "Gone", status: :gone if @shared_file.user.banned?
```

This keeps public URLs clean.

## 5. Operational Features

### 5.1 Automatic Cleanup

A background job runs every 15 minutes to purge expired files, bans, and old notifications. It deletes files from disk via `ActiveStorage::Blob.service`, ensuring no orphaned data.

### 5.2 Cloudflare Integration

`config/initializers/cloudflare.rb` trusts Cloudflare proxy IPs so `request.ip` reflects the real client. The Docker compose file now includes a `cloudflared` sidecar, simplifying local testing and production deployment.

### 5.3 Docker Compose

The compose file was updated to include:

```yaml
services:
  web:
    build: .
    environment:
      - HOST
      - WEBAUTHN_ORIGIN
      - WEBAUTHN_RP_ID
      - FOR
  uploads:
    volume: uploads
  db_data:
    volume: db_data
```

Volumes `uploads` and `db_data` persist files and the SQLite database, respectively.

## 6. Testing & CI

After hardening, the test suite grew to 109 tests and 257 assertions. The updated `lefthook.yml` now runs:

1. **Rubocop** – style and Rails‑specific linting.
2. **Brakeman** – static security analysis.
3. **Bundler‑Audit** – vulnerable gem detection.
4. **Rails tests** – full integration coverage.

The CI pipeline in `.github/workflows/ci.yml` triggers on pushes and PRs, ensuring that any new feature automatically undergoes the same scrutiny.

## 7. Lessons Learned

| Lesson | Takeaway |
|--------|----------|
| Fail‑fast is safer than fail‑soft | Explicit env var checks stop an insecure server from starting. |
| Atomic DB ops are essential | A race‑condition on download counters could let users exceed their quota. |
| User‑centric UX reduces support | Friendly expired/not‑found pages and quota bars improve trust. |
| Security shouldn’t be an afterthought | Integrating Brakeman and RuboCop into the merge workflow catches issues early. |
| Self‑hosted means self‑auditing | Regular security audits and CI keep the app hardened over time. |

## Conclusion

In a day’s worth of commits, FrankMega became a robust, secure, and user‑friendly file‑sharing service for home servers behind Cloudflare. The journey showed that coupling security hardening, internationalization, and thoughtful UX can be done even on a small codebase. If you’re building a self‑hosted tool, start with a clear threat model, automate checks, iterate UI based on real flows, and document configuration upfront. With those principles, you’ll create software that’s functional, resilient, and trustworthy. Happy hacking!
