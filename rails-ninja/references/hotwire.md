
---
name: rails-hotwire
description: Hotwire (Turbo Drive, Turbo Frames, Turbo Streams, Stimulus)
version: 1.0.0
rails_version: ">= 7.0"
tags:
  - hotwire
  - turbo
  - stimulus
  - frontend
---

# Rails Hotwire

Hotwire is an alternative approach to building modern web applications without using much JavaScript. It consists of Turbo (Drive, Frames, Streams) and Stimulus.

## Quick Reference

| Component | Purpose |
|-----------|---------|
| **Turbo Drive** | Fast page navigation without full reload |
| **Turbo Frames** | Decompose pages into independent contexts |
| **Turbo Streams** | Deliver page changes as HTML over WebSocket or in response to form submissions |
| **Stimulus** | JavaScript sprinkles for enhanced interactivity |

## Turbo Drive

Turbo Drive automatically makes all link clicks and form submissions use AJAX, replacing the page body without full reload.

```erb
<%# Turbo Drive is enabled by default in Rails 7+ %>
<%# Links and forms automatically use Turbo Drive %>

<%= link_to "Posts", posts_path %>
<%= link_to "External Site", "https://example.com", data: { turbo: false } %>

<%# Opt-out for specific elements %>
<div data-turbo="false">
  <%= link_to "Normal Link", some_path %>
</div>

<%# Programmatic navigation %>
<a href="/posts" data-turbo-action="advance">Posts</a>
<a href="/posts" data-turbo-action="replace">Posts (Replace History)</a>
```

## Turbo Frames

Turbo Frames allow you to scope navigation and form submissions to a specific part of the page.

### Basic Frame

```erb
<%# app/views/posts/index.html.erb %>
<%= turbo_frame_tag "posts_list" do %>
  <%= render @posts %>
  <%= link_to "New Post", new_post_path %>
<% end %>

<%# app/views/posts/new.html.erb %>
<%= turbo_frame_tag "posts_list" do %>
  <h2>New Post</h2>
  <%= form_with model: @post do |f| %>
    <%= f.text_field :title %>
    <%= f.submit %>
  <% end %>
<% end %>
```

### Lazy Loading Frames

```erb
<%# Load content on demand %>
<%= turbo_frame_tag "post_#{post.id}", src: post_path(post), loading: :lazy do %>
  <div class="loading">Loading post...</div>
<% end %>
```

### Targeting Frames

```erb
<%# Target a specific frame from outside %>
<%= link_to "Edit", edit_post_path(@post), data: { turbo_frame: "modal" } %>
<%= link_to "Break out of frame", posts_path, data: { turbo_frame: "_top" } %>

<%# Turbo Frame that breaks out by default %>
<%= turbo_frame_tag "navigation", target: "_top" do %>
  <%= link_to "Home", root_path %>
<% end %>
```

## Turbo Streams

Turbo Streams let you update multiple parts of the page in response to actions.

### Stream Actions

```erb
<%# app/views/posts/create.turbo_stream.erb %>

<%# Append to a container %>
<%= turbo_stream.append "posts" do %>
  <%= render @post %>
<% end %>

<%# Prepend to a container %>
<%= turbo_stream.prepend "posts" do %>
  <%= render @post %>
<% end %>

<%# Replace an element %>
<%= turbo_stream.replace @post do %>
  <%= render @post %>
<% end %>

<%# Update content of an element %>
<%= turbo_stream.update "post_#{@post.id}" do %>
  <%= render @post %>
<% end %>

<%# Remove an element %>
<%= turbo_stream.remove "post_#{@post.id}" %>

<%# Insert before/after %>
<%= turbo_stream.before "post_#{@post.id}" do %>
  <div class="notice">Updated!</div>
<% end %>

<%= turbo_stream.after "post_#{@post.id}" do %>
  <div class="ad">Advertisement</div>
<% end %>
```

### Controller Setup

```ruby
class PostsController < ApplicationController
  def create
    @post = Post.new(post_params)

    respond_to do |format|
      if @post.save
        format.turbo_stream
        format.html { redirect_to @post }
      else
        format.turbo_stream { render :form_errors, status: :unprocessable_entity }
        format.html { render :new, status: :unprocessable_entity }
      end
    end
  end

  def update
    respond_to do |format|
      if @post.update(post_params)
        format.turbo_stream
        format.html { redirect_to @post }
      else
        format.turbo_stream { render :form_errors, status: :unprocessable_entity }
        format.html { render :edit, status: :unprocessable_entity }
      end
    end
  end

  def destroy
    @post.destroy

    respond_to do |format|
      format.turbo_stream { render turbo_stream: turbo_stream.remove(@post) }
      format.html { redirect_to posts_path }
    end
  end
end
```

### Broadcast Updates (Real-time)

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  broadcasts_to ->(post) { "posts" }, inserts_by: :prepend

  # Or more control
  after_create_commit -> { broadcast_prepend_to "posts" }
  after_update_commit -> { broadcast_replace_to "posts" }
  after_destroy_commit -> { broadcast_remove_to "posts" }

  # Broadcast to specific users/channels
  after_create_commit -> { broadcast_prepend_to [author, "posts"], target: "posts_list" }
end
```

```erb
<%# Subscribe to broadcasts %>
<%= turbo_stream_from "posts" %>

<div id="posts">
  <%= render @posts %>
</div>
```

## Stimulus

Stimulus is a modest JavaScript framework for enhancing static or server-rendered HTML.

### Basic Controller

```javascript
// app/javascript/controllers/clipboard_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["source"]
  static values = {
    message: String,
    count: { type: Number, default: 0 }
  }

  copy() {
    navigator.clipboard.writeText(this.sourceTarget.value)
    this.countValue++
    alert(`Copied! (${this.countValue} times)`)
  }
}
```

```erb
<div data-controller="clipboard">
  <input data-clipboard-target="source" type="text" value="Copy me!">
  <button data-action="click->clipboard#copy">Copy to Clipboard</button>
</div>
```

### Common Patterns

```javascript
// Toggle visibility
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["content"]

  toggle() {
    this.contentTarget.classList.toggle("hidden")
  }
}
```

```erb
<div data-controller="toggle">
  <button data-action="click->toggle#toggle">Toggle</button>
  <div data-toggle-target="content" class="hidden">
    Hidden content
  </div>
</div>
```

```javascript
// Form autosave
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["form", "status"]

  async save() {
    this.statusTarget.textContent = "Saving..."

    const formData = new FormData(this.formTarget)
    const response = await fetch(this.formTarget.action, {
      method: this.formTarget.method,
      body: formData,
      headers: {
        "X-CSRF-Token": document.querySelector("[name='csrf-token']").content
      }
    })

    if (response.ok) {
      this.statusTarget.textContent = "Saved!"
    } else {
      this.statusTarget.textContent = "Error saving"
    }
  }
}
```

```erb
<div data-controller="autosave">
  <%= form_with model: @post, data: { autosave_target: "form", action: "change->autosave#save" } do |f| %>
    <%= f.text_area :content %>
  <% end %>
  <span data-autosave-target="status"></span>
</div>
```

## Best Practices

1. **Use Turbo Frames** for independent page sections
2. **Use Turbo Streams** for real-time updates and multi-part updates
3. **Keep Stimulus controllers small** and focused on one behavior
4. **Use Stimulus values** instead of reading from the DOM
5. **Graceful degradation** - ensure functionality works without JavaScript when possible
6. **Use broadcasts** for real-time features (chat, notifications, etc.)
7. **Test Turbo interactions** with system tests
8. **Use `data-turbo-confirm`** for destructive actions
9. **Optimize** by lazy-loading frames and using caching

## Common Patterns

### Modal with Turbo Frame

```erb
<%# Layout %>
<div id="modal" class="modal">
  <%= turbo_frame_tag "modal_content" %>
</div>

<%# Link that opens modal %>
<%= link_to "New Post", new_post_path, data: { turbo_frame: "modal_content" } %>

<%# app/views/posts/new.html.erb %>
<%= turbo_frame_tag "modal_content" do %>
  <h2>New Post</h2>
  <%= render "form", post: @post %>
<% end %>
```

### Inline Editing

```erb
<%# Show mode %>
<%= turbo_frame_tag "post_#{@post.id}" do %>
  <h2><%= @post.title %></h2>
  <%= link_to "Edit", edit_post_path(@post) %>
<% end %>

<%# Edit mode %>
<%# app/views/posts/edit.html.erb %>
<%= turbo_frame_tag "post_#{@post.id}" do %>
  <%= form_with model: @post do |f| %>
    <%= f.text_field :title %>
    <%= f.submit "Save" %>
  <% end %>
<% end %>
```

### Pagination with Turbo Frames

```erb
<%= turbo_frame_tag "posts" do %>
  <%= render @posts %>
  <%= link_to "Load More", posts_path(page: @next_page) %>
<% end %>
```

## Troubleshooting

- **Frame not updating**: Check that frame IDs match
- **Form not working**: Ensure you're using `form_with` not `form_for`
- **Turbo Drive issues**: Use `data-turbo="false"` to opt-out
- **Broadcasts not working**: Check ActionCable is configured
- **Stimulus controller not connecting**: Check console for errors, verify data-controller name

## References

- [Turbo Handbook](https://turbo.hotwired.dev/handbook/introduction)
- [Stimulus Handbook](https://stimulus.hotwired.dev/handbook/introduction)
- [Hotwire Homepage](https://hotwired.dev/)
- [Turbo Rails Gem](https://github.com/hotwired/turbo-rails)


# Hotwire Reference (37signals Style)

---

## Core Rules

- Always `render :action, status: :unprocessable_entity` on validation failure (Turbo needs non-2xx to re-render)
- Use `method: :morph` for replacing complex card/record containers
- Controller responds to `turbo_stream` format; renders `.turbo_stream.erb` template
- Broadcast from models, not controllers
- Keep Stimulus controllers under 50 lines, single responsibility

---

## Turbo Frames

```erb
<%# Scoped page region — only this updates %>
<%= turbo_frame_tag "post_#{@post.id}" do %>
  <h2><%= @post.title %></h2>
  <%= link_to "Edit", edit_post_path(@post) %>
<% end %>

<%# Edit form must match frame id %>
<%= turbo_frame_tag "post_#{@post.id}" do %>
  <%= render "form", post: @post %>
<% end %>

<%# Lazy-loaded frame %>
<%= turbo_frame_tag "comments", src: post_comments_path(@post), loading: :lazy %>

<%# Break out of frame %>
<%= link_to "View full page", post_path(@post), data: { turbo_frame: "_top" } %>
```

### Controller — nothing special needed
```ruby
def update
  if @post.update(post_params)
    redirect_to @post
  else
    render :edit, status: :unprocessable_entity  # Turbo re-renders the frame
  end
end
```

---

## Turbo Streams

### Template pattern (preferred for complex updates)
```erb
<%# app/views/cards/create.turbo_stream.erb %>
<%= turbo_stream.replace dom_id(@card, :card_container),
    partial: "cards/container",
    method: :morph,
    locals: { card: @card.reload } %>

<%# Multiple targets %>
<%= turbo_stream.prepend "cards_list", @card %>
<%= turbo_stream.update  "card_count", Card.count %>
<%= turbo_stream.replace :flash, partial: "shared/flash" %>
```

### Controller
```ruby
def create
  @card = current_user.cards.create!(card_params)
  respond_to do |format|
    format.turbo_stream  # renders create.turbo_stream.erb
    format.html { redirect_to cards_path }
  end
end
```

### Actions: `append`, `prepend`, `replace`, `update`, `remove`, `before`, `after`

---

## ActionCable Broadcasts (37signals pattern)

### Model broadcasts
```ruby
class Card < ApplicationRecord
  # Auto-broadcast on create/update/destroy
  broadcasts_to ->(card) { [card.board, :cards] }, inserts_by: :prepend
end

class Comment < ApplicationRecord
  broadcasts_to :card  # broadcasts to card's stream
end
```

### Manual broadcasts
```ruby
Turbo::StreamsChannel.broadcast_replace_to(
  [board, :cards],
  target: dom_id(card, :card_container),
  partial: "cards/container",
  locals: { card: card }
)
```

### Subscribe in view
```erb
<%= turbo_stream_from @board, :cards %>
<div id="cards_list">
  <%= render @cards %>
</div>
```

---

## Stimulus Controller Catalog

37signals has ~52 Stimulus controllers, 62% reusable. Key patterns:
- Single responsibility
- `static values` / `static classes` for configuration
- `this.dispatch("event")` for communication between controllers
- Private methods with `#`

### Copy to Clipboard
```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values  = { content: String }
  static classes = ["success"]

  async copy(event) {
    event.preventDefault()
    this.reset()
    try {
      await navigator.clipboard.writeText(this.contentValue)
      this.element.classList.add(this.successClass)
    } catch {}
  }

  reset() {
    this.element.classList.remove(this.successClass)
    this.element.offsetWidth  // force reflow for animation reset
  }
}
```

### Auto Resize Textarea
```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { minHeight: { type: Number, default: 0 } }

  connect()  { this.resize() }
  resize()   {
    this.element.style.height = "auto"
    const h = Math.max(this.minHeightValue, this.element.scrollHeight)
    this.element.style.height = `${h}px`
  }
}
// Usage: data-controller="autoresize" data-action="input->autoresize#resize"
```

### Toggle Class
```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static classes  = ["toggle"]
  static targets  = ["checkbox"]

  toggle()   { this.element.classList.toggle(this.toggleClass) }
  add()      { this.element.classList.add(this.toggleClass) }
  remove()   { this.element.classList.remove(this.toggleClass) }
}
```

### Auto Submit Form
```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  submit() { this.element.requestSubmit() }
}
// Usage: data-controller="auto-submit" data-action="change->auto-submit#submit"
```

### Element Removal
```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  remove() { this.element.remove() }
}
// Usage: <div data-controller="element-removal">
//          <button data-action="element-removal#remove">Dismiss</button>
//        </div>
```

### Hotkey
```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  click(event) {
    if (this.#isClickable && !this.#shouldIgnore(event)) {
      event.preventDefault()
      this.element.click()
    }
  }

  #shouldIgnore(event) {
    return event.defaultPrevented || event.target.closest("input, textarea")
  }

  get #isClickable() {
    return getComputedStyle(this.element).pointerEvents !== "none"
  }
}
// Usage: data-action="keydown.n@window->hotkey#click"
```

### Dialog (native `<dialog>`)
```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["dialog"]
  static values  = { modal: { type: Boolean, default: false } }

  open()   {
    this.modalValue ? this.dialogTarget.showModal() : this.dialogTarget.show()
    this.dialogTarget.setAttribute("aria-hidden", "false")
    this.dispatch("show")
  }

  close()  {
    this.dialogTarget.close()
    this.dialogTarget.setAttribute("aria-hidden", "true")
    this.dispatch("close")
  }

  toggle() { this.dialogTarget.open ? this.close() : this.open() }

  closeOnClickOutside({ target }) {
    if (!this.element.contains(target)) this.close()
  }
}
```

### Fetch on Visible (lazy loading)
```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { url: String, threshold: { type: Number, default: 0.1 } }

  connect() {
    this.observer = new IntersectionObserver(
      entries => entries.forEach(e => { if (e.isIntersecting) this.fetch() }),
      { threshold: this.thresholdValue }
    )
    this.observer.observe(this.element)
  }

  disconnect() { this.observer?.disconnect() }

  async fetch() {
    this.observer.disconnect()
    const response = await fetch(this.urlValue)
    this.element.innerHTML = await response.text()
  }
}
// Usage: data-controller="fetch-on-visible" data-fetch-on-visible-url-value="/lazy-content"
```

---

## View Helpers (Stimulus-integrated)

```ruby
# Auto-submit form helper
def auto_submit_form_with(**attributes, &)
  data = attributes.delete(:data) || {}
  data[:controller] = "auto-submit #{data[:controller]}".strip
  form_with **attributes, data: data, &
end

# Clipboard button
def button_to_copy_to_clipboard(url, &)
  tag.button class: "btn", data: {
    controller: "copy-to-clipboard",
    action: "copy-to-clipboard#copy",
    copy_to_clipboard_success_class: "btn--success",
    copy_to_clipboard_content_value: url
  }, &
end

# Icon helper
def icon_tag(name, **options)
  tag.span class: class_names("icon icon--#{name}", options.delete(:class)),
           "aria-hidden": true, **options
end
```
