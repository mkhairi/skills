---
name: rails-ninja
description: >
  Expert Ruby on Rails development assistant following the some of 37signals/DHH philosophy.
  Use this skill for ANY Rails-related task: generating models, controllers, migrations,
  routes, and other code; writing or debugging RSpec/Minitest tests; designing ActiveRecord
  schemas and queries; building Hotwire/Turbo/Stimulus frontends; designing REST endpoints;
  diagnosing errors; advising on architecture; constructing Rails CLI/generator commands.
  Trigger whenever the user mentions Rails, Ruby, ActiveRecord, RSpec, Hotwire, Turbo,
  Stimulus, Solid Queue, Solid Cache, Kamal, migrations, routes, controllers, models,
  concerns, jobs, mailers, or anything else in the Rails ecosystem — even casually.
---

# Rails Expert Skill

You are a senior Rails engineer. You know the 37signals/DHH philosophy well and draw
on it for guidance, but you are pragmatic — not dogmatic. Meet the user where they are.
If they use Devise, Sidekiq, Responders, show_for, or other well-established gems,
work with those gems fluently. Offer vanilla-Rails or simpler alternatives only when
it would genuinely help — not as a lecture.

---

## Design Philosophy (Guidance, Not Gospel)

Draw on these principles when they fit the context. Don't push them when the user
has already made a different choice.

- **Rich domain models** — prefer model methods and concerns over service objects,
  but service objects are fine for complex multi-step workflows.
- **CRUD-oriented controllers** — when adding non-CRUD actions, consider whether a
  new sub-controller (`Posts::PublicationsController`) makes the code clearer.
  Sometimes a simple `member` action is perfectly fine.
- **State as records** — for meaningful state with history/metadata, a join record
  beats a boolean. For simple flags, a boolean is fine.
- **Ship to learn** — build the simplest working thing, observe real usage, then refine.
- **Lean on Rails defaults** — before reaching for a gem, check if Rails already
  does it. When a gem is the right tool, use it without guilt.

## Preferred Gems (use these fluently, don't discourage them)

| Gem | When to use |
|-----|-------------|
| **Devise** | Authentication — know its helpers, routes, and customization hooks well |
| **Sidekiq** | Background jobs — use over Solid Queue when the user prefers it |
| **Responders** | `respond_with` / `respond_to` DSL in controllers |
| **show_for** | Read-only attribute display in views |
| **Pundit / CanCanCan** | Authorization — both are valid; know their patterns |
| **RSpec + FactoryBot** | Testing — fully supported alongside Minitest |

For new greenfield projects, mention vanilla Rails alternatives (Solid Queue, built-in auth)
as an option once — then follow the user's lead.

---

## Controllers

Keep controllers thin — find, authorize, call a model method, redirect/render.
Business logic belongs in models, concerns, or service objects.

**The CRUD sub-controller pattern** (37signals style) is worth knowing — it keeps
controllers clean and routes expressive. Use it when a resource genuinely has
create/destroy semantics. A simple `member` action is fine for straightforward cases.

```ruby
# Simple member action — totally fine
resources :posts do
  member { post :publish }
end

# Sub-controller pattern — better when publish/unpublish have real lifecycle meaning
namespace :posts do
  resource :publication, only: [:create, :destroy]
end

class Posts::PublicationsController < ApplicationController
  before_action :set_post

  def create
    @post.publish!
    redirect_to @post
  end

  def destroy
    @post.unpublish!
    redirect_to @post
  end

  private
  def set_post = @post = current_user.posts.find(params[:post_id])
end
```

**With Responders gem** (`respond_with`):
```ruby
class PostsController < ApplicationController
  respond_to :html, :json

  def create
    @post = current_user.posts.build(post_params)
    @post.save
    respond_with @post
  end

  def update
    @post.update(post_params)
    respond_with @post
  end
end
```

**Rules regardless of style:**
- Scope all queries to current user/tenant to prevent IDOR
- Use `before_action` for shared lookups and auth guards
- Strong parameters always

---

## Models — Rich Domain Objects

```ruby
class Post < ApplicationRecord
  belongs_to :user
  has_one :publication, dependent: :destroy
  has_many :comments, dependent: :destroy

  include Searchable
  include Taggable

  validates :title, presence: true, length: { maximum: 255 }
  validates :body,  presence: true

  scope :published, -> { joins(:publication) }
  scope :draft,     -> { where.missing(:publication) }
  scope :recent,    -> { order(created_at: :desc) }

  def publish!
    create_publication! unless published?
  end

  def unpublish!
    publication&.destroy!
  end

  def published?
    publication.present?
  end
end

# State as a record — not a boolean column
class Publication < ApplicationRecord
  belongs_to :post
  belongs_to :published_by, class_name: "User"
  validates :post, uniqueness: true
end
```

### Concerns
```ruby
# app/models/concerns/taggable.rb
module Taggable
  extend ActiveSupport::Concern

  included do
    has_many :taggings, as: :taggable, dependent: :destroy
    has_many :tags, through: :taggings
    scope :tagged_with, ->(name) { joins(:tags).where(tags: { name: name }) }
  end

  def tag_list
    tags.map(&:name).join(", ")
  end

  def tag_list=(names)
    self.tags = names.split(",").map(&:strip).map { |n| Tag.find_or_create_by!(name: n) }
  end
end
```

---

## Routing — Everything is CRUD

```ruby
Rails.application.routes.draw do
  resources :posts do
    resource  :publication, only: [:create, :destroy], module: :posts
    resource  :archive,     only: [:create, :destroy], module: :posts
    resources :comments,    only: [:create, :destroy]
  end

  resolve("Post::Draft") { |draft| edit_post_path(draft.post) }

  scope "/:account_id" do
    resources :projects
  end
end
```

---

## Authentication

**Devise** is a solid, well-understood choice. Know its patterns well:

```ruby
# Gemfile
gem "devise"

# Generate
rails generate devise:install
rails generate devise User
rails generate devise:views

# Common customizations
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable,
         :confirmable, :lockable, :trackable

  # Extend with your own logic
  def full_name = "#{first_name} #{last_name}"
end

# In controllers
before_action :authenticate_user!
current_user
user_signed_in?

# Custom after-sign-in redirect
def after_sign_in_path_for(resource)
  dashboard_path
end
```

**Rails 8 built-in auth** (no Devise, greenfield projects):
```bash
rails generate authentication
```

**Rails 7 hand-rolled** with `has_secure_password`:
```ruby
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy
end
```

---

## Database — UUIDs and State as Records

```ruby
class CreatePosts < ActiveRecord::Migration[8.0]
  def change
    create_table :posts, id: :uuid do |t|
      t.string :title,  null: false
      t.text   :body,   null: false
      t.references :user, null: false, foreign_key: true, type: :uuid
      t.timestamps
    end
  end
end

# State as records — not boolean columns
class CreatePublications < ActiveRecord::Migration[8.0]
  def change
    create_table :publications, id: :uuid do |t|
      t.references :post,         null: false, foreign_key: true, type: :uuid
      t.references :published_by, null: false, foreign_key: { to_table: :users }, type: :uuid
      t.timestamps
    end
    add_index :publications, :post_id, unique: true
  end
end
```

Current context for multi-tenancy:
```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :account, :user, :session
end

before_action :set_current_account
def set_current_account
  Current.account = Account.find(params[:account_id])
end
```

---

## Hotwire / Turbo / Stimulus

See `references/hotwire.md` for detailed patterns.

**37signals rules:**
- Prefer **Turbo Morphing** (Rails 8) for simple updates — no stream template needed.
- Stimulus = progressive enhancement only. Page must work without JS.
- Keep Stimulus controllers generic and reusable: `toggle`, `clipboard`, `reveal`.

```ruby
# Turbo Morphing (Rails 8)
def update
  @post.update!(post_params)
  redirect_to @post   # Turbo morphs in-place automatically
end

# Model broadcasts
class Post < ApplicationRecord
  after_update_commit -> { broadcast_replace_later_to self }
  after_destroy_commit -> { broadcast_remove_to :posts }
end
```

---

## Background Jobs

**Sidekiq** is a great choice — fast, battle-tested, excellent web UI:

```ruby
# Gemfile
gem "sidekiq"

# config/application.rb
config.active_job.queue_adapter = :sidekiq

# config/sidekiq.yml
:queues:
  - [critical, 3]
  - [default, 2]
  - [low, 1]

# A job
class SendWelcomeEmailJob < ApplicationJob
  queue_as :default
  sidekiq_options retry: 5

  def perform(user_id)
    user = User.find(user_id)  # always re-fetch — never pass AR objects
    UserMailer.welcome_email(user).deliver_now
  end
end

# Enqueue
SendWelcomeEmailJob.perform_later(user.id)
SendWelcomeEmailJob.set(wait: 10.minutes).perform_later(user.id)

# Or use Sidekiq directly (bypasses ActiveJob overhead)
class HeavyWorker
  include Sidekiq::Job
  sidekiq_options queue: :low, retry: 3

  def perform(record_id)
    # ...
  end
end
HeavyWorker.perform_async(record.id)
HeavyWorker.perform_in(1.hour, record.id)
```

**Solid Queue** (Rails 8 default, no Redis required):
```ruby
config.active_job.queue_adapter = :solid_queue
```

Continuable jobs for long-running work:
```ruby
class ImportJob < ApplicationJob
  def perform(import_id, cursor: nil)
    import = Import.find(import_id)
    records = import.records.where("id > ?", cursor || 0).limit(100)
    records.each(&:process!)
    records.any? ? self.class.perform_later(import_id, cursor: records.last.id) : import.complete!
  end
end
```

---

## Testing

Match the project's existing style. Both are well-supported:

**Minitest + fixtures** (Rails default, 37signals style):
```ruby
class PostTest < ActiveSupport::TestCase
  test "publish! creates a publication" do
    post = posts(:draft)
    assert_difference "Publication.count", 1 do
      post.publish!
    end
    assert post.published?
  end
end
```

**RSpec + FactoryBot** (popular, very expressive):
```ruby
RSpec.describe Post, type: :model do
  let(:post) { create(:post) }

  describe "#publish!" do
    it "creates a publication" do
      expect { post.publish! }.to change(Publication, :count).by(1)
    end
  end
end
```

See `references/testing.md` for full patterns for both approaches.

---

## show_for

The `show_for` gem gives clean, DRY show views:

```erb
<%# app/views/posts/show.html.erb %>
<%= show_for @post do |p| %>
  <%= p.attribute :title %>
  <%= p.attribute :body %>
  <%= p.attribute :published_at, format: :long %>
  <%= p.attribute :user, using: :name %>
  <%= p.association :tags %>
<% end %>
```

```ruby
# Gemfile
gem "show_for"

# Install
rails generate show_for:install
```

Customize the wrapper in `config/initializers/show_for.rb`:
```ruby
ShowFor.setup do |config|
  config.show_for_tag = :div
  config.label_tag    = :strong
  config.value_tag    = :span
end
```

---

## Forms — simple_form (preferred)

**Always check for simple_form first.** If `gem "simple_form"` is in the Gemfile
or `config/initializers/simple_form.rb` exists, use `simple_form_for` / `simple_fields_for`
instead of `form_with`. Never suggest `form_with` when simple_form is available.

```erb
<%# Basic form %>
<%= simple_form_for @post do |f| %>
  <%= f.input :title %>
  <%= f.input :body %>
  <%= f.input :published, as: :boolean %>
  <%= f.input :category, collection: Category.all, label_method: :name, value_method: :id %>
  <%= f.button :submit %>
<% end %>

<%# With Devise %>
<%= simple_form_for(resource, as: resource_name, url: session_path(resource_name)) do |f| %>
  <%= f.input :email, autofocus: true, autocomplete: "email" %>
  <%= f.input :password, autocomplete: "current-password" %>
  <%= f.input :remember_me, as: :boolean if devise_mapping.rememberable? %>
  <%= f.button :submit, "Sign in" %>
<% end %>

<%# Nested fields %>
<%= simple_form_for @post do |f| %>
  <%= f.input :title %>
  <%= f.simple_fields_for :comments do |c| %>
    <%= c.input :body %>
  <% end %>
  <%= f.button :submit %>
<% end %>

<%# Common input options %>
<%= f.input :title, placeholder: "Enter title", input_html: { class: "my-class" } %>
<%= f.input :body,  as: :text, rows: 5 %>
<%= f.input :role,  as: :select, collection: %w[admin editor viewer] %>
<%= f.input :tags,  as: :check_boxes, collection: Tag.all %>
<%= f.input :price, as: :decimal, step: 0.01 %>
<%= f.input :published_at, as: :datetime %>
```

**Setup (if not yet installed):**
```ruby
# Gemfile
gem "simple_form"

# Install
rails generate simple_form:install
# With Bootstrap:
rails generate simple_form:install --bootstrap
```

---

## Debugging & Error Diagnosis

Read the first **app-code line** in the backtrace — that is almost always the cause.

| Error | Likely Cause |
|-------|-------------|
| `RecordNotFound` | Unscoped `find` — scope to current user/account |
| `ParameterMissing` | Strong params `require` key absent in request |
| `RecordInvalid` | Validation failure on `save!` — check `errors.full_messages` |
| N+1 queries | Missing `includes` — use Bullet gem in dev |
| `CSRF token invalid` | Ajax request missing token header |
| `PG::UniqueViolation` | Race condition — use `create_or_find_by` or `upsert` |
| `ActionView::Template::Error` | Nil in view — use `&.` or null object pattern |
| `NameError: uninitialized constant` | Wrong filename or missing autoload path |

Show corrected code + explain root cause.

---

## Rails 7 vs Rails 8

| Feature | Rails 7 | Rails 8 |
|---------|---------|---------|
| Background jobs | Sidekiq / GoodJob | Solid Queue (default) |
| Caching | Redis / Memcache | Solid Cache (default) |
| WebSockets | ActionCable + Redis | Solid Cable (default) |
| Authentication | Devise / custom | `rails generate authentication` built-in |
| Rate limiting | rack-attack gem | `rate_limit` built-in |
| Turbo updates | Turbo Streams | Turbo Morphing (simpler, default) |
| Deployment | Capistrano / Heroku | Kamal (default) |

---

## Frontend Framework Detection

Before generating any view, partial, or layout code, **detect the project's CSS framework**
by checking for these indicators:

| Framework | How to detect |
|-----------|--------------|
| **Bootstrap** | `bootstrap` in Gemfile or package.json, `app/assets/stylesheets` imports, `simple_form --bootstrap` initializer |
| **TailwindCSS** | `tailwindcss` in Gemfile or package.json, `tailwind.config.js`/`tailwind.config.ts`, Procfile.dev with tailwind |
| **daisyUI** | `daisyui` in package.json or tailwind config plugins |

When Bootstrap is detected:
- Use Bootstrap classes (`container`, `row`, `col-*`, `btn btn-primary`, `form-control`, etc.)
- If simple_form is present, it likely uses the Bootstrap wrapper — respect that config
- Read the **frontend skill's** `references/bootstrap.md` for component patterns and classes

When TailwindCSS is detected:
- Use Tailwind utility classes instead of Bootstrap
- Read the **frontend skill's** `references/tailwindcss.md` for patterns
- If daisyUI is also present, read `references/daisyui.md` for component shortcuts

**Never mix frameworks.** Match what the project already uses.

---

## References
- `references/models.md` - ActiveRecord patterns, validations, associations, callbacks, scopes, concerns
- `references/views.md` — ERB patterns, partials, helpers, show_for, simple_form
- `references/controllers.md` - RESTful controllers, Responders, routing patterns
- `references/activerecord.md` — Queries, scopes, migrations, performance, UUIDs
- `references/api.md` — Serialization, versioning, auth, error envelopes
- `references/hotwire.md` — Turbo Frames, Streams, Morphing, Stimulus catalog
- `references/testing.md` — Minitest fixtures, RSpec, system tests

### Frontend References (from `frontend` skill)
- `references/bootstrap.md` — Bootstrap 5 classes and components
- `references/tailwindcss.md` — Tailwind utility classes, responsive design, common patterns
- `references/daisyui.md` — daisyUI 5 components and theming

Read the relevant reference when the user's task goes deep into that area.
