# ActiveRecord Reference (37signals Style)

---

## State as Records (Core Pattern)

The single most important 37signals DB pattern: **don't use boolean columns for state**.
Create a separate record instead. You get timestamp, author, and easy scoping for free.

```ruby
# BAD: boolean column
add_column :cards, :closed, :boolean, default: false
# You only know current state. Not when. Not who.

# GOOD: separate record
create_table :closures do |t|
  t.references :card, null: false
  t.references :user          # who closed it
  t.timestamps                # when it happened
end

class Closure < ApplicationRecord
  belongs_to :card, touch: true  # touch: true invalidates card cache
  belongs_to :user, optional: true
end

class Card < ApplicationRecord
  has_one :closure, dependent: :destroy

  scope :closed, -> { joins(:closure) }
  scope :open,   -> { where.missing(:closure) }

  def closed?    = closure.present?
  def open?      = !closed?
  def closed_at  = closure&.created_at
  def closed_by  = closure&.user
end
```

---

## Migrations

```ruby
class CreateCards < ActiveRecord::Migration[8.0]
  def change
    create_table :cards, id: :uuid do |t|
      t.uuid    :account_id, null: false
      t.uuid    :board_id,   null: false
      t.uuid    :creator_id, null: false
      t.string  :title,      null: false, limit: 255
      t.string  :status,     null: false, default: "draft"
      t.integer :number,     null: false
      t.timestamps
    end

    add_index :cards, :account_id
    add_index :cards, [:account_id, :number], unique: true
  end
end
```

### Rules
- Always reversible — prefer `change`
- Add DB constraints alongside model validations (`null: false`, unique index)
- Every model gets `account_id` for multi-tenancy
- UUIDs for primary keys
- `touch: true` on `belongs_to` for cache invalidation (no Russian doll complexity)

---

## Associations

### Default values via lambdas
```ruby
class Card < ApplicationRecord
  belongs_to :account, default: -> { board.account }  # derived from parent
  belongs_to :creator, class_name: "User", default: -> { Current.user }
end
```

### touch: true as cache invalidation
```ruby
# When a comment changes → card.updated_at changes → card cache key changes
class Comment  < ApplicationRecord; belongs_to :card, touch: true; end
class Closure  < ApplicationRecord; belongs_to :card, touch: true; end
class Goldness < ApplicationRecord; belongs_to :card, touch: true; end
```

---

## Scopes

```ruby
# State (from record presence)
scope :closed,    -> { joins(:closure) }
scope :open,      -> { where.missing(:closure) }
scope :golden,    -> { joins(:goldness) }
scope :postponed, -> { joins(:not_now) }

# Standard ordering names
scope :chronologically,         -> { order(created_at: :asc) }
scope :reverse_chronologically, -> { order(created_at: :desc) }
scope :alphabetically,          -> { order(name: :asc) }
scope :latest,                  -> { order(last_active_at: :desc) }

# Preloading — use "preloaded" as standard name
scope :preloaded, -> {
  includes(:creator, :tags, :closure, :goldness, board: [:columns])
}

# Parameterized
scope :indexed_by, ->(index) do
  case index.to_s
  when "closed" then closed
  when "open"   then open
  else all
  end
end
```

---

## N+1 Prevention

```ruby
# Bad
Card.all.each { |c| puts c.creator.name }  # N+1

# Good — includes
Card.includes(:creator, :tags).all

# Good — preloaded scope
Card.preloaded.published.latest.limit(20)

# Allows WHERE on association (LEFT JOIN)
Card.eager_load(:creator).where(users: { role: "admin" })
```

---

## Validations

```ruby
# Minimal — 37signals uses lean validations
validates :title,  presence: true, length: { maximum: 255 }
validates :email,  format: { with: URI::MailTo::EMAIL_REGEXP }
validates :status, inclusion: { in: %w[draft published archived] }

# Contextual
validates :full_name, presence: true, on: :completion
```

---

## Callbacks: Use Sparingly

~38 callback uses across a large 37signals app. Use only for:
- Data normalization (`before_save :normalize_title`)
- Derived data (`before_create :generate_slug`)
- Async broadcasts (`after_create_commit :broadcast`)

**Avoid** for emails, external calls, or anything testable in isolation.

---

## Locking

```ruby
# Pessimistic (for race conditions on unique sequences)
Card.transaction do
  card = Card.lock.find(id)
  card.with_lock { card.update!(number: account.next_number) }
end
```

---

## Bulk Operations

```ruby
Post.insert_all(records)                               # no callbacks
Post.upsert_all(records, unique_by: :slug)             # upsert
Card.where(draft: true).update_all(status: "archived") # bulk update
Card.find_each(batch_size: 500) { |c| c.reindex }      # large sets
```


## References

- [Rails Guides - Active Record](https://guides.rubyonrails.org/active_record_basics.html)
- [Rails Guides - Validations](https://guides.rubyonrails.org/active_record_validations.html)
- [Rails Guides - Associations](https://guides.rubyonrails.org/association_basics.html)
- [Rails Guides - Migrations](https://guides.rubyonrails.org/active_record_migrations.html)
- [Rails Guides - Queries](https://guides.rubyonrails.org/active_record_querying.html)
