+++  
date = '2026-02-21T16:52:20-03:00'  
draft = false  
title = 'From N+1 to Inline Previews: FrankMega’s Deep‑Dive into Security, UX, and Performance'  
+++

FrankMega is a tiny self‑hosted file‑sharing service that runs in a single Docker container.  
It has no external dependencies – no Redis, no Postgres, no S3 – and is designed to be
run behind a Cloudflare Tunnel in a home network.  
Over the past day the codebase has moved from a decent MVP to a more production‑ready
system.  The changes touch three intertwined layers:

* **Admin UI and analytics** – give the owner a real‑time view of usage.  
* **Upload & invitation flow** – tighten security and add a better UX for
  invitation handling.  
* **Download handling** – replace broken redirects, add inline previews, and
  secure the content‑security policy.

Below is a walk‑through of the commits that drove this evolution, the problems they
solve, and the lessons that can be applied to any Rails project.

---

## 1. Turning N+1 into a single query

The first commit replaces the naive `@users = User.order(created_at: :desc)` with a
single, left‑joined query that also returns the number of files and the total storage
used per user.  The change is small but eliminates a classic performance pitfall.

```ruby
# app/controllers/admin/users_controller.rb
def index
  @users = User
    .left_joins(:shared_files)
    .select(
      "users.*,
       COUNT(shared_files.id) AS files_count,
       COALESCE(SUM(shared_files.file_size), 0) AS total_storage"
    )
    .group("users.id")
    .order(created_at: :desc)
end
```

**What it buys us**

* A single round‑trip to the database instead of one per user.  
* No N+1 queries when the admin page is rendered.  
* The view can now show a concise “3 files / 15.2 MB” string for every user
  without any post‑processing.

The view was updated accordingly:

```erb
# app/views/admin/users/index.html.erb
<th class="px-6 py-3 ..."><%= t("admin.users.index.usage") %></th>
...
<td class="px-6 py-4">
  <%= @user.files_count %> <%= t("admin.users.index.files") %> /
  <%= number_to_human_size(@user.total_storage) %>
</td>
```

---

## 2. Who actually used an invite?

Invite handling had a UI quirk: the admin table showed the *creator* of an invitation
instead of the user who redeemed it.  Changing the association was trivial but
required a few view tweaks:

```ruby
# app/controllers/admin/invitations_controller.rb
def index
  @invitations = Invitation.includes(:used_by).order(created_at: :desc)
end
```

```erb
# app/views/admin/invitations/index.html.erb
<th ...><%= t("admin.invitations.index.used_by") %></th>
...
<td class="px-6 py-4">
  <%= invitation.used_by&.email || "" %>
</td>
```

*Pending* or *expired* invitations now show an empty cell, which is more intuitive
for an admin.

---

## 3. Copy‑friendly invitation URLs

When an admin creates an invitation, the previous implementation embedded the URL
directly inside the flash message.  That made copy‑pasting awkward and broke the
single‑page‑application feel because the URL was lost on navigation.

The commit splits the URL into its own flash key and renders it as a
copy‑able input field using the existing `clipboard` Stimulus controller.

```ruby
# app/controllers/admin/invitations_controller.rb
if @invitation.save
  flash[:notice] = t("flash.admin.invitations.create.notice")
  flash[:invitation_url] = register_url(code: @invitation.code)
  redirect_to admin_invitations_path
else
  render :new, status: :unprocessable_entity
end
```

```erb
# app/views/admin/invitations/index.html.erb
<% if flash[:notice] %>
  <div class="bg-green-50 ...">
    <p><%= flash[:notice] %></p>
    <% if flash[:invitation_url] %>
      <div data-controller="clipboard"
           data-clipboard-copied-text-value="<%= t('js.copied') %>">
        <div class="flex">
          <input type="text"
                 readonly
                 value="<%= flash[:invitation_url] %>"
                 data-clipboard-target="source"
                 class="flex-1 ...">
          <button type="button"
                  data-action="click->clipboard#copy"
                  class="px-4 py-2 bg-primary ...">
            <%= t("shared.actions.copy") %>
          </button>
        </div>
      </div>
    <% end %>
  </div>
<% end %>
```

The user experience has improved dramatically: the URL is always visible, can be
copied with a single click, and the flash message remains in the UI after navigation.

---

## 4. Upload sanity, client‑side validation, and filename safety

### 4‑1 Server‑side sanitization

The original `sanitize_filename` method was simple.  It stripped slashes and
replaced invalid UTF‑8 but left many edge cases:

| Problem | Fix |
| ------- | --- |
| Control characters, Windows reserved names (`CON`, `PRN`, etc.) | Strip with regex, prepend `_` |
| Leading dots (e.g., `.hidden`) | Remove |
| Filenames longer than 255 bytes | Truncate while preserving extension |
| Empty names | Default to `"unnamed_file"` |

```ruby
def sanitize_filename(name)
  sanitized = File.basename(name.to_s)
    .encode("UTF-8", invalid: :replace, undef: :replace, replace: "")
    .gsub(/[\x00-\x1f\x7f\/\\:*?"<>|]/, "")
    .sub(/\A\.+/, "")
    .gsub(/\s+/, " ").strip

  base_without_ext = sanitized.sub(/\.[^.]*\z/, "")
  if base_without_ext.match?(/\A(CON|PRN|AUX|NUL|COM[1-9]|LPT[1-9])\z/i)
    sanitized = "_#{sanitized}"
  end

  truncate_filename(sanitized, 255) || "unnamed_file"
end

def truncate_filename(name, max_bytes)
  return name if name.bytesize <= max_bytes

  ext = File.extname(name)
  base = File.basename(name, ext)
  max_base = max_bytes - ext.bytesize
  return ext.byteslice(0, max_bytes) if max_base <= 0

  base = base.chop while base.bytesize > max_base
  "#{base}#{ext}"
end
```

The method now guarantees that the resulting filename is safe for both the file
system and the web UI.

### 4‑2 Client‑side validation with Stimulus

The `upload_controller.js` was extended to validate the file before the
browser starts the upload:

```js
// app/javascript/controllers/upload_controller.js
static targets = ["dropzone", "input", "preview", "filename", "filesize", "progress", "progressBar", "progressText"]

connect() {
  // …
  this.inputTarget.addEventListener("change", this.validate.bind(this))
}

validate(event) {
  const file = event.target.files[0]
  if (!file) return

  const maxSize = Number(this.element.dataset.maxSize)
  if (file.size > maxSize) {
    alert(`File is larger than ${this.humanSize(maxSize)}`)
    event.target.value = ""
    return
  }

  const quota = Number(this.element.dataset.quota)
  const current = Number(this.element.dataset.current)
  if (current + file.size > quota) {
    alert(`You have exceeded your storage quota`)
    event.target.value = ""
    return
  }

  // Filename sanity
  const sanitized = this.sanitize(file.name)
  if (sanitized !== file.name) {
    alert(`Filename will be sanitized to ${sanitized}`)
    this.filenameTarget.textContent = sanitized
  }
}
```

The controller now:

* Checks file size against a per‑user quota.
* Prevents uploads that would exceed the account’s remaining storage.
* Sanitizes the filename on the fly and informs the user if a change is
  required.

---

## 5. Download flow – from POST to GET, `send_file`, and inline previews

### 5‑1 GET link instead of POST

The original design used `button_to` with a POST request to trigger the download.
Turbo Drive intercepted the request and performed a redirect chain that never
resulted in a browser download.  Switching to a plain `link_to` with
`data-turbo="false"` bypasses Turbo's interposition and lets the browser
handle the file stream directly.

```erb
<%= link_to t("downloads.show.download"),
            download_file_path(hash: @shared_file.download_hash),
            class: "block w-full py-3 px-4 bg-primary ...",
            data: { turbo: false } %>
```

The controller was renamed from `create` to `file` for clarity, and the route
was updated accordingly.

### 5‑2 `send_file` vs. redirect chain

The old code issued a `redirect_to rails_blob_path(...)`.  That produced two
redirects (POST → 302 → 302 → 200) and never set the `Content‑Disposition`
header correctly.  By replacing it with `send_file` we stream the file
directly from the local disk:

```ruby
# app/controllers/downloads_controller.rb
else
  @shared_file.reload
  DownloadNotificationJob.perform_later(@shared_file.id)
  send_file ActiveStorage::Blob.service.path_for(@shared_file.file.key),
            filename: @shared_file.original_filename,
            type: @shared_file.content_type,
            disposition: "attachment"
end
```

The change eliminates the redirect chain, reduces latency, and ensures the
download prompt behaves as expected on every browser.

### 5‑3 Inline preview route

A new `preview` action was added to serve files inline without consuming
their download counter.  This is ideal for media that the user wants to
preview before deciding to download.

```ruby
# app/controllers/downloads_controller.rb
def preview
  if @shared_file.nil? || !@shared_file.active? || !@shared_file.previewable?
    render plain: "Not Found", status: :not_found
  else
    send_file ActiveStorage::Blob.service.path_for(@shared_file.file.key),
              filename: @shared_file.original_filename,
              type: @shared_file.content_type,
              disposition: "inline"
  end
end
```

The `SharedFile` model now knows what is *previewable*:

```ruby
# app/models/shared_file.rb
PREVIEWABLE_IMAGE_TYPES = %w[image/jpeg image/png image/gif image/webp image/svg+xml].freeze
PREVIEWABLE_VIDEO_TYPES = %w[video/mp4 video/webm].freeze
PREVIEWABLE_AUDIO_TYPES = %w[audio/mpeg audio/ogg].freeze

def image?  = PREVIEWABLE_IMAGE_TYPES.include?(content_type)
def video?  = PREVIEWABLE_VIDEO_TYPES.include?(content_type)
def audio?  = PREVIEWABLE_AUDIO_TYPES.include?(content_type)
def previewable? = image? || video? || audio?
```

The preview URL is added to the download page:

```erb
<% if @shared_file.previewable? %>
  <a href="<%= preview_file_path(hash: @shared_file.download_hash) %>"
     class="text-sm text-blue-600">Preview</a>
<% end %>
```

Because the route is `GET /d/:hash/preview`, it can be used from the admin panel,
the user’s download page, or any external link.

### 5‑4 Truncating long filenames

Long filenames can break the layout on small screens.  The UI now truncates
the display to 60 characters but keeps the full name as a tooltip:

```erb
<h2 class="text-xl font-semibold text-gray-900 dark:text-white mb-2 truncate"
    title="<%= @shared_file.original_filename %>">
  <%= truncate(@shared_file.original_filename, length: 60) %>
</h2>
```

The truncation is purely visual; the download link still uses the full filename
in the `Content‑Disposition` header thanks to `send_file`.

---

## 6. Content‑Security‑Policy: Rails built‑in, no more importmap breakage

The project originally relied on the `secure_headers` gem for CSP, but the gem
did not support Rails’ built‑in nonce generation.  Inline scripts generated by
Rails’ importmap were being blocked.

The new `config/initializers/content_security_policy.rb` moves the policy to
Rails’ native API and configures it to allow inline styles (needed for Tailwind
and the Stimulus UI) while still protecting against XSS:

```ruby
# config/initializers/content_security_policy.rb
Rails.application.configure do
  config.content_security_policy do |policy|
    policy.default_src :self
    policy.font_src    :self
    policy.img_src     :self, :data
    policy.object_src  :none
    policy.script_src  :self
    policy.style_src   :self, :unsafe_inline
    policy.connect_src :self
    policy.frame_ancestors :none
    policy.form_action :self
    policy.base_uri    :self
  end
end
```

The `secure_headers` gem still handles other headers (X‑Frame‑Options, HSTS,
X‑Content‑Type‑Options, etc.) but no longer sets the CSP.  The change removes
the CSP mismatch that caused importmap scripts to be blocked while keeping
the rest of the security stack intact.

---

## 7. Security hardening and test coverage

### 7‑1 Brakeman ignore updates

The `config/brakeman.ignore` file now contains two new fingerprints:

1. The `send_file` usage – the file key comes from the ActiveStorage blob,
   not user input, so the path is safe.
2. The `preview` action – it never modifies the download counter and
   only serves files that are still active.

Adding these ignores keeps the static analysis focused on real risks.

### 7‑2 Test updates

* **DownloadsControllerTest** – now verifies that `GET /file` returns a
  `:success` status instead of a redirect.
* **SecurityTest** – confirms the new download flow and that a second
  download attempt returns `:gone`.
* **SharedFileTest** – ensures `previewable?` works for all media types.

The test suite now covers the critical paths of the new download logic
and the filename sanitization edge cases.

---

## 8. Takeaway – Building a secure, user‑friendly file‑sharing service

| Problem | Solution | Why it matters |
|---------|----------|----------------|
| N+1 queries in admin panel | `left_joins` + `group` + `COALESCE` | One DB round‑trip, instant analytics |
| Misleading invite usage | Show `used_by` instead of `created_by` | Accurate audit trail |
| Hard‑copy URLs | Clipboard Stimulus + separate flash key | Better UX, less friction |
| Unsafe filenames | Regex + truncation + Windows‑reserved guard | Prevent path traversal, OS conflicts |
| Upload abuses | Client‑side quota & size checks | Reduce bandwidth abuse, better UX |
| Turbo blocking downloads | GET link + `data-turbo=false` | Browser downloads work everywhere |
| Redirect chain | `send_file` | Faster, reliable downloads |
| Media preview | Inline route + `previewable?` | Users can see before deciding |
| CSP mismatches | Rails native CSP + nonces | Importmap scripts work, XSS protection |
| Static analysis noise | Brakeman ignores for safe paths | Focus on real issues |

FrankMega’s recent commits demonstrate how a small project can evolve into a
robust, secure service without sacrificing developer ergonomics.  The changes
highlight a few best practices that are broadly applicable:

* Prefer single‑query aggregates over per‑record N+1 lookups.  
* Use Rails’ built‑in CSP when you need dynamic nonces; third‑party gems
  can be dropped if they interfere.  
* Keep the UI responsive by letting Turbo only handle what it should – heavy
  downloads should be plain links.  
* Sanitize user input aggressively, especially filenames that will be stored
  on the file system.  
* Provide immediate feedback with client‑side validation and copy‑buttons.  
* Serve files directly with `send_file` to avoid redirect loops and
  header mis‑configurations.

The result is a self‑hosted file‑sharing app that feels polished, secure,
and ready for real‑world home‑server use.  If you’re building a similar
service, consider adopting these patterns early – they save time and prevent
security holes down the road.
