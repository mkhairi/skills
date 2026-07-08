# API Reference (37signals Style)

37signals uses REST + Turbo — not GraphQL. The same controllers often serve both HTML/Turbo
and JSON responses. API-only endpoints use bearer token auth, not sessions.

---

## Same Controllers, Multiple Formats

```ruby
class Cards::CommentsController < ApplicationController
  include CardScoped

  def create
    @comment = @card.comments.create!(comment_params)

    respond_to do |format|
      format.turbo_stream   # renders create.turbo_stream.erb
      format.json { head :created, location: card_comment_path(@card, @comment) }
    end
  end

  def destroy
    @comment.destroy

    respond_to do |format|
      format.turbo_stream { render turbo_stream: turbo_stream.remove(@comment) }
      format.json { head :no_content }
    end
  end
end
```

---

## HTTP Status Codes

| Action | Success Status |
|--------|---------------|
| GET | 200 OK |
| POST (created) | 201 Created + `Location` header |
| PUT/PATCH | 204 No Content |
| DELETE | 204 No Content |
| Validation error | 422 Unprocessable Entity |
| Not authenticated | 401 Unauthorized |
| Forbidden | 403 Forbidden |
| Not found | 404 Not Found |

---

## Bearer Token Authentication

```ruby
# In ApplicationController or API concern
private
  def authenticate_by_bearer_token
    if token = request.authorization&.match(/^Bearer (.+)$/)&.[](1)
      if access_token = AccessToken.find_by_token(token)
        set_current_session_from_access_token(access_token)
      end
    end
  end
```

---

## Versioning

```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :cards, only: [:index, :show, :create, :update, :destroy]
    resources :boards, only: [:index, :show]
  end
end
```

```
app/controllers/api/v1/
  application_controller.rb
  cards_controller.rb
  boards_controller.rb
```

---

## Serialization (keep it simple)

37signals doesn't use heavy serializer gems. For simple APIs:

```ruby
# Inline hash (simple cases)
render json: {
  id: @card.id,
  title: @card.title,
  closed: @card.closed?,
  created_at: @card.created_at
}

# Presenter PORO (for reuse)
class CardPresenter
  def initialize(card)
    @card = card
  end

  def as_json(*)
    {
      id: @card.id,
      title: @card.title,
      closed: @card.closed?,
      closed_at: @card.closed_at,
      closed_by: @card.closed_by&.name
    }
  end
end

render json: CardPresenter.new(@card)
```

If you need a serializer gem, prefer **Blueprinter** or **Alba** over ActiveModel::Serializers.

---

## Error Handling

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound,       with: :not_found
  rescue_from ActiveRecord::RecordInvalid,        with: :unprocessable_entity
  rescue_from ActionController::ParameterMissing, with: :bad_request

  private
    def not_found(e)
      respond_to do |format|
        format.html { render "errors/404", status: :not_found }
        format.json { render json: { error: e.message }, status: :not_found }
      end
    end

    def unprocessable_entity(e)
      respond_to do |format|
        format.html { redirect_back fallback_location: root_path, alert: e.message }
        format.json { render json: { errors: e.record.errors.full_messages }, status: :unprocessable_entity }
      end
    end
end
```

---

## Rate Limiting

```ruby
# Rails 8 built-in
class ApiController < ActionController::API
  rate_limit to: 100, within: 1.minute, by: -> { request.remote_ip }
end

# Rails 7 — use rack-attack gem
```

---

## Pagination

37signals uses `geared_pagination` (their own gem — cursor-based):

```ruby
# Gemfile
gem "geared_pagination"

# Controller
set_page_and_extract_portion_from @cards.published.latest

# View
<%= link_to "Next", cards_path(page: @page.next_param) if @page.next? %>
```

For simpler pagination, Kaminari works fine:
```ruby
@cards = Card.published.latest.page(params[:page]).per(20)
render json: {
  cards: @cards.map { CardPresenter.new(_1).as_json },
  meta: { current_page: @cards.current_page, total_pages: @cards.total_pages }
}
```
