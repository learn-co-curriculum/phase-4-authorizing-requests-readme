# Authorizing Requests

## Learning Goals

- Understand the difference between _authentication_ and _authorization_
- Restrict access to routes to authorized users only

## Introduction

So far, we've been talking about how to **authenticate** users, i.e., how to
confirm that a user is who they say they are. We've been using their username as
our means of authentication; in the future, we'll also add a password to our
authentication process.

In addition to **authentication**, most applications also need to implement
**authorization**: giving certain users permission to access specific resources.
For example, we might want **all** users to be able to browse blog posts, but
only **authenticated** users to have access to premium features, like creating
their own blog posts. In this lesson, we'll learn how we can use the session
hash to authenticate users' requests, and give them explicit permission to
access certain routes in our application.

## First Pass: Manual Checks

Let's say we have a `DocumentsController`. Its `show` method looks like this:

```ruby
def show
  document = Document.find(params[:id])
  render json: document
end
```

Now let's add a new requirement: documents should only be shown to users when
they're logged in. From a technical perspective, what does it actually mean for
a user to _log in_? When a user logs in, all we are doing is using cookies to
add their `:user_id` to the `session` hash.

The first thing you might do is to add a **guard clause** as the first line of
`DocumentsController#show`:

```ruby
def show
  return render json: { error: "Not authorized" }, status: :unauthorized unless session.include? :user_id
  document = Document.find(params[:id])
  render json: document
end
```

Unless the session includes `:user_id`, we return an error. `status:
:unauthorized` will return the specified HTTP status code. In this case, if a
user isn't logged in, we return `401 Unauthorized`.

## Refactor

This code works fine, so you use it in a few places. Now your
`DocumentsController` looks like this:

```ruby
class DocumentsController < ApplicationController
  def show
    return render json: { error: "Not authorized" }, status: :unauthorized unless session.include? :user_id
    document = Document.find(params[:id])
    render json: document
  end

  def index
    return render json: { error: "Not authorized" }, status: :unauthorized unless session.include? :user_id
    documents = Document.all
    render json: documents
  end

  def create
    return render json: { error: "Not authorized" }, status: :unauthorized unless session.include? :user_id
    document = Document.create(author_id: session[:user_id])
    render json: document, status: :created
  end

  def update
    return render json: { error: "Not authorized" }, status: :unauthorized unless session.include? :user_id
    document = Document.find(params[:id])
    # code to update a document
  end
end
```

That doesn't look so DRY. Wouldn't it be great if there were a way to ask Rails
to run some code **before** any controller **action**?

Fortunately, Rails gives us a solution: [`before_action`][filters]. We can
refactor our code like so:

```ruby
class DocumentsController < ApplicationController
  before_action :authorize

  def show
    document = Document.find(params[:id])
    render json: document
  end

  def index
    documents = Document.all
    render json: documents
  end

  def create
    document = Document.create(author_id: session[:user_id])
    render json: document, status: :created
  end

  private

  def authorize
    return render json: { error: "Not authorized" }, status: :unauthorized unless session.include? :user_id
  end
end
```

We've moved our guard clause into its own method and added the following code at
the top of our `DocumentsController`:

```ruby
before_action :authorize
```

This is a call to the `ActionController` class method `before_action`.
`before_action` registers a filter. A filter is a method which runs **before**,
**after**, or **around** a controller's action. In this case, the filter runs
before all `DocumentsController`'s actions, and kicks requests out with
`401 Unauthorized` unless they're logged in.

## Skipping Filters for Certain Actions

What if we wanted to let anyone see a list of documents, but keep the
`before_action` filter for other `DocumentsController` methods? We could do
this:

```ruby
class DocumentsController < ApplicationController
  before_action :authorize
  skip_before_action :authorize, only: [:index]

  # ...
end
```

This class method tells Rails to skip the `authorize` filter only on the `index`
action:

```ruby
skip_before_action :authorize, only: [:index]
```

## Conclusion

To **authorize** a user for specific actions, we can take advantage of the fact
that all logged in users in our application will have a `:user_id` saved in the
session hash. We can use a `before_action` filter to run some code that will
check the `:user_id` in the session and only authorize users to run those
actions if they are logged in.

## Check For Understanding

Before you move on, make sure you can answer the following questions:

1. What is the difference between authentication and authorization?
2. What Rails method can we use to add an authorization step before each of the
   actions in our controller? What Rails method can we use to exclude one or
   more of the actions from the authorization step?

## Resources

- [Action Controller Overview: Filters][filters]

[filters]: http://guides.rubyonrails.org/action_controller_overview.html#filters
