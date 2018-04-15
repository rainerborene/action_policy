# Caching

Action Policy aims to be as much performant as possible. One of the way to accomplish that is to build a comprehensive caching system.

There are several cache layers available: rule-level memoization, local (instance-level) memoization, _external_ cache (through cache stores).

## Policies memoization

### Per-instance

There could be a situation when you need to apply the same policy to the same record multiple times during the action (e.g., request). For example:

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    authorize! @post
    render :show
  end
end
```

```erb
# app/views/posts/show.html.erb
<h1><%= @post.title %>

<% if allowed_to?(:edit?, @post) %>
  <%= link_to "Edit", @post %>
<% end %>

<% if allowed_to?(:destroy?, @post) %>
  <%= link_to "Delete", @post, method: :delete %>
<% end %>
```

In the above example, we need to use the same policy three times. Action Policy re-uses the same policy instance to avoid unnecessary objects allocation.

We rely on the following assumptions:
- parent object (e.g., controller instance) is _ephemeral_, i.e., it's a short-lived object
- all authorizations uses the same [authorization context](authorization_context.md).

We use `record.policy_cache_key` with fallback to `record.cache_key` or `record.object_id` as a part of policy identifier in the local store.

**NOTE**: policies memoization is an extension for `ActionPolicy::Behaviour` and could be included with `ActionPolicy::Behaviours::Memoized`.

**NOTE**: memoization is automatically included into Rails controllers integration (but not included into channels integration, 'cause channels are long-lived objects).

### Per-thread

Consider more complex situation:

```ruby
# app/controllers/comments_controller.rb
class CommentsController < ApplicationController
  def index
    # all comments for all posts
    @comments = Comment.all
  end
end
```

```erb
# app/views/comments/index.html.erb
<% @comments.each do |comment| %>
  <li><%= comment.text %>
    <% if allowed_to?(:edit?, comment) %>
      <%= link_to comment, "Edit" %>
    <% end %>
  </li>
<% end %>
```

```ruby
# app/policies/comment_policy.rb
class CommentPolicy < ApplicationPolicy
  def edit?
    user.admin? || (user.id == record.id) ||
      allowed_to?(:manage?, record.post)
  end
end
```

In some case, we have to initialize **two** policies for each comment: one for the comment itself and one for the comment's post (in `allowed_to?` call).

That's an example of _N+1 authorization_ problem, which in its turn could easily cause _N+1 query_ problem (if `PostPolicy#manage?` makes DB queries). Sounds bad, doesn't it?

It is likely that many comments belong to the same posts. If so, we can move our memoization one level up and use local thread store.

Action Policy provides `ActionPolicy::Behaviours::ThreadMemoized` module which provides this functionality (included into Rails controllers integration by default).

If you want to add this behaviour to your custom authorization-aware class, you should care about cleaning up thread store manually (by calling `ActionPolicy::PerThreadCache.clear_all`).

## Rules cache

### Per-instance

There could be a situation when the same rule is called multiple times for the same policy instance (for example, when using [aliases](aliases.md)).

In that case, Action Policy invokes the rule method only once, remember the result, and return it immediately for the subsequent calls.

**NOTE**: rules results memoization is available only if you inherit from `ActionPolicy::Base` or include `ActionPolicy::Policy::CachedApply` into your `ApplicationPolicy`.

### Using cache store

Some policy rules might be _performance-heavy_, e.g., make complex DB queries.

In that case, it makes sense to cache the rule application result for a long time (not just during the same request).

Action Policy provides a way to use _cache stores_ for that. You have to explicitly define which rules you want to cache in your policy class. For example:

```ruby
class StagePolicy < ApplicationPolicy
  # mark show? rule to be cached
  cache :show?
  # you can also provide store-specific options
  # cache :show?, expires_in: 1.hour

  def show?
    full_access? ||
      user.stage_permissions.where(
        stage_id: record.id
      ).exists?
  end

  private

  def full_access?
    !record.funnel.is_private? ||
      user.permissions
          .where(
            funnel_id: record.funnel_id,
            full_access: true
          ).exists?
  end
end
```

You must configure a cache store in order to use this feature:

```ruby
ActionPolicy.cache_store = MyCacheStore.new
```

Or in Rails:

```ruby
# config/application.rb (or config/environments/<environment>.rb)
Rails.application.configure do |config|
  config.action_policy.cache_store = :redis_cache_store
end
```

Cache store must provide at least `#fetch(key, **options, &block)` method.

By default, Action Policy builds a cache key using the following scheme:

```ruby
"#{cache_namespace}/#{context_cache_key}" \
"/#{record.policy_cache_key}/#{policy.class.name}/#{rule}"
```

Where `cache_namespace` is equal to "acp:#{MAJOR_GEM_VERSION}.#{MINOR_GEM_VERSION}", `context_cache_key` is concatanated cache keys of all authorization contexts (in order they defined in the policy class).

When any of the objects doesn't respond to `#policy_cache_key`, we fallback to `#cache_key`. If `#cache_key` is not defined `ArgumentError` is raised.

You can define your own `cache_key` / `cache_namespace` / `context_cache_key` methods for policy class to override this logic.

#### Invalidation

There no one-size-fits-all solution here. It highly depends on your business-logic.

**Case \#1**: no invalidation required.

First of all, you should try to avoid manual invalidation at all. That could be achieved by using _good_ cache keys.

Let's consider an example.

Suppose that your users have _roles_ (i.e. `User.belongs_to :role`) and you give access to resources through `Access` model (i.e. `Resource.has_many :accesses`).

Then you can do the following:
- Keep tracking the last `Access` added/updated/deleted for resource (e.g. `Access.belongs_to :accessessable, touch: :access_updated_at`)
- Use the following cache keys:

```ruby
class User
  def policy_cache_key
    "user::#{id}::#{role_id}"
  end
end

class Resource
  def policy_cache_key
    "#{resource.class.name}::#{id}::#{access_updated_at}"
  end
end
```

**Case \#2**: discarding all cache at once.

That's pretty easy: just override `cache_namespace` method in your `ApplicationPolicy` with the new value:

```ruby
class ApplicationPolicy < ActionPolicy::Base
  # It's good to store the changing part in the constant
  CACHE_VERSION = "v2".freeze

  # or even from the env variable
  # CACHE_VERSION = ENV.fetch("POLICY_CACHE_VERSION", "v2").freeze

  def cache_namespace
    "action_policy::#{CACHE_VERSION}"
  end
end
```

**Case \#3**: discarding some keys.

That's an alternative approach to _crafting_ cache keys.

If you have a limited number of places in your application where you update access control,
you can invalidate policies cache manually. If your cache store supports `delete_matched` command (deleting keys using a wildcard), you can try the following:

```ruby
class ApplicationPolicy < ActionPolicy::Base
  # Define custom cache key generator
  def cache_key(rule)
    "policy_cache/#{user.id}/#{self.class.name}/#{record.id}/#{rule}"
  end
end

class Access < ApplicationRecord
  belongs_to :resource
  belongs_to :user

  after_commit :cleanup_policy_cache, on: [:create, :destroy]

  def cleanup_policy_cache
    # Clear cache for the corresponding user-record pair
    ActionPolicy.cache_store.delete_matched(
      "policy_cache/#{user_id}/#{ResourcePolicy.name}/#{resource_id}/*"
    )
  end
end

class User < ApplicationRecord
  belongs_to :role

  after_commit :cleanup_policy_cache, on: [:update], if: :role_id_changed?

  def cleanup_policy_cache
    # Clear all policies cache for user
    ActionPolicy.cache_store.delete_matched(
      "policy_cache/#{user_id}/*"
    )
  end
end
```