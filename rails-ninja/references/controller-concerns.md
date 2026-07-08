# Controller Concerns Catalog (37signals Style)

Controller concerns create a vocabulary of reusable behaviors that compose beautifully.
This is the 37signals "secret sauce" for keeping controllers thin.

---

## Resource Scoping Concerns

### CardScoped

For any controller nested under cards. Provides `@card`, `@board`, and `render_card_replacement`.

```ruby
# app/controllers/concerns/card_scoped.rb
module CardScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_card, :set_board
  end

  private
    def set_card
      @card = Current.user.accessible_cards.find_by!(number: params[:card_id])
    end

    def set_board
      @board = @card.board
    end

    # Shared UI update method — all card actions use this for consistency
    def render_card_replacement
      render turbo_stream: turbo_stream.replace(
        [@card, :card_container],
        partial: "cards/container",
        method: :morph,
        locals: { card: @card.reload }
      )
    end
end

# Usage — any controller under cards/:card_id/...
class Cards::ClosuresController < ApplicationController
  include CardScoped

  def create
    @card.close
    respond_to do |format|
      format.turbo_stream { render_card_replacement }
      format.json { head :no_content }
    end
  end
end

class Cards::WatchesController < ApplicationController
  include CardScoped

  def create
    @card.watch_by Current.user
    respond_to do |format|
      format.turbo_stream { render_card_replacement }
      format.json { head :no_content }
    end
  end
end
```

### BoardScoped

For controllers nested under boards.

```ruby
module BoardScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_board
  end

  private
    def set_board
      @board = Current.user.boards.find(params[:board_id])
    end

    def ensure_permission_to_admin_board
      head :forbidden unless Current.user.can_administer_board?(@board)
    end
end

# Usage
class Boards::ColumnsController < ApplicationController
  include BoardScoped

  def create
    @column = @board.columns.create!(column_params)
  end
end

class Boards::PublicationsController < ApplicationController
  include BoardScoped
  before_action :ensure_permission_to_admin_board

  def create
    @board.publish
    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to @board }
    end
  end
end
```

### ColumnScoped

```ruby
module ColumnScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_column
  end

  private
    def set_column
      @column = Current.user.accessible_columns.find(params[:column_id])
    end
end
```

---

## Request Context Concerns

### CurrentRequest — Populate Current with request data

```ruby
# app/controllers/concerns/current_request.rb
module CurrentRequest
  extend ActiveSupport::Concern

  included do
    before_action do
      Current.http_method = request.method
      Current.request_id  = request.uuid
      Current.user_agent  = request.user_agent
      Current.ip_address  = request.ip
      Current.referrer    = request.referrer
    end
  end
end

# Now models can access request context without parameter passing:
class Identity < ApplicationRecord
  def log_access
    AccessLog.create!(
      ip_address: Current.ip_address,   # No params needed
      user_agent: Current.user_agent
    )
  end
end
```

### CurrentTimezone — User timezone from cookie

```ruby
module CurrentTimezone
  extend ActiveSupport::Concern

  included do
    around_action :set_current_timezone
    helper_method :timezone_from_cookie
    etag { timezone_from_cookie }  # Critical: different timezones → different cached responses
  end

  private
    def set_current_timezone(&)
      Time.use_zone(timezone_from_cookie, &)
    end

    def timezone_from_cookie
      @timezone_from_cookie ||= begin
        timezone = cookies[:timezone]
        ActiveSupport::TimeZone[timezone] if timezone.present?
      end
    end
end

# Why etag includes timezone:
# Pages render times server-side ("3:00 PM"). If cached and served to all users:
# NYC user sees "3:00 PM" ✓
# London user gets same cache, sees "3:00 PM" ✗ (should be "8:00 PM")
# Including timezone in etag gives each timezone its own cached version.
```

### SetPlatform — Detect mobile/desktop

```ruby
module SetPlatform
  extend ActiveSupport::Concern

  included do
    helper_method :platform
  end

  private
    def platform
      @platform ||= ApplicationPlatform.new(request.user_agent)
    end
end

# In views:
# <% if platform.mobile? %><%= render "mobile_nav" %><% end %>
```

---

## Authentication Concern

```ruby
# app/controllers/concerns/authentication.rb
module Authentication
  extend ActiveSupport::Concern

  included do
    before_action :require_authentication
    helper_method :authenticated?
  end

  class_methods do
    def allow_unauthenticated_access(**options)
      skip_before_action :require_authentication, **options
      before_action :resume_session, **options
    end
  end

  private
    def authenticated?
      Current.user.present?
    end

    def require_authentication
      resume_session || authenticate_by_bearer_token || request_authentication
    end

    def resume_session
      if session = find_session_by_cookie
        set_current_session session
      end
    end

    def find_session_by_cookie
      Session.find_signed(cookies.signed[:session_token])
    end

    def start_new_session_for(identity)
      identity.sessions.create!(
        user_agent: request.user_agent,
        ip_address: request.remote_ip
      ).tap { |session| set_current_session session }
    end

    def set_current_session(session)
      Current.session = session
      cookies.signed.permanent[:session_token] = {
        value: session.signed_id,
        httponly: true,
        same_site: :lax
      }
    end

    def request_authentication
      redirect_to sign_in_path
    end
end
```

---

## Security Concerns

### BlockSearchEngineIndexing

```ruby
module BlockSearchEngineIndexing
  extend ActiveSupport::Concern

  included do
    after_action :block_search_engine_indexing
  end

  private
    def block_search_engine_indexing
      headers["X-Robots-Tag"] = "none"
    end
end
# Include in ApplicationController for private apps
```

### RequestForgeryProtection — Modern CSRF via Sec-Fetch-Site

```ruby
module RequestForgeryProtection
  extend ActiveSupport::Concern

  included do
    after_action :append_sec_fetch_site_to_vary_header
  end

  private
    SAFE_FETCH_SITES = %w[same-origin same-site]

    def verified_request?
      request.get? || request.head? || !protect_against_forgery? ||
        (valid_request_origin? && safe_fetch_site?)
    end

    def safe_fetch_site?
      SAFE_FETCH_SITES.include?(sec_fetch_site_value) ||
        (sec_fetch_site_value.nil? && api_request?)
    end

    def api_request?
      request.format.json?
    end
end
# Modern browsers set Sec-Fetch-Site automatically — can't be spoofed by JS
```

---

## Turbo Concerns

### TurboFlash — Flash messages via Turbo Stream

```ruby
module TurboFlash
  extend ActiveSupport::Concern

  included do
    helper_method :turbo_stream_flash
  end

  private
    def turbo_stream_flash(**flash_options)
      turbo_stream.replace(:flash,
        partial: "layouts/shared/flash",
        locals: { flash: flash_options })
    end
end

# Usage in controller:
render turbo_stream: [
  turbo_stream.append(:comments, @comment),
  turbo_stream_flash(notice: "Comment added!")
]
```

### ViewTransitions — Disable on refresh

```ruby
module ViewTransitions
  extend ActiveSupport::Concern

  included do
    before_action :disable_view_transitions, if: :page_refresh?
  end

  private
    def disable_view_transitions
      @disable_view_transition = true
    end

    def page_refresh?
      request.referrer.present? && request.referrer == request.url
    end
end
```

---

## Composing Concerns

Concerns can include other concerns:

```ruby
module DayTimelinesScoped
  extend ActiveSupport::Concern
  include FilterScoped  # inherits all of FilterScoped

  included do
    before_action :set_timeline
  end
end

# A controller can mix many concerns:
class Events::Days::ColumnsController < ApplicationController
  include DayTimelinesScoped  # which includes FilterScoped
  include BoardScoped

  def show
    @column = @board.columns.find(params[:id])
  end
end
```

### Concern composition rules

1. Use `included do; before_action; end` — not `prepend_before_action`
2. Provide shared private helper methods (like `render_card_replacement`)
3. Use `helper_method` to expose to views
4. Add to `etag` for HTTP caching when the value affects response content
5. Concerns can build on other concerns via `include`

---

## FilterScoped — Complex filtering

```ruby
module FilterScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_filter
  end

  private
    def set_filter
      if params[:filter_id].present?
        @filter = Current.user.filters.find(params[:filter_id])
      else
        @filter = Current.user.filters.from_params(filter_params)
      end
    end

    def filter_params
      params.reverse_merge(**Filter.default_values).permit(*Filter::PERMITTED_PARAMS)
    end
end

# Key insight: Filters are persisted records — users can save and name them
# The Filter model does the heavy query lifting, not the controller
```
