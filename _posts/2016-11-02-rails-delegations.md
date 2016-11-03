---
layout: post
title: "Delegating Methods in Rails"
comments: false
description: "Delegating methods from one class to another."
keywords: "ruby rails delegate"
---

Sometimes you have a class that needs to know some information about one
of its relationships. In order to achieve something like this, it may be
wise to consider using a [delegation design pattern](https://en.wikipedia.org/wiki/Delegation_pattern).

In a basic sense, a delegation pattern is a way to achieve reusable code.
Sounds a lot like inheritance, right? In a sense, delegation
kind of *is* inheritance, just done a little differently. Perhaps you
have a subclass inheriting from a superclass where the subclass really
only *needs* one or two of the superclass' public methods. Instead
of inheriting for the sake of using one inherited method, we can
delegate that method out to the appropriate class.

In addition to creating reusable code, sometimes delegation makes syntactic
sense in itself. Let's say that I have an `OperationRequest` class, and a
`Status` class, where an `OperationRequest` has_one `Status`, and a
`Status` has_many `OperationRequest`'s. An `OperationRequest` is either
approved or denied based on it's status. Therefore, a `Status` might have
a field for content of "approved" or "denied".

Initially, I might persuade myself to write something like this:

{% highlight ruby %}
class Status < ActiveRecord::Base
  has_many :operation_requests

  def approved?
    content == "approved"
  end

  def denied?
    content == "denied"
  end
end
{% endhighlight %}

And...

{% highlight ruby %}
class OperationRequest < ActiveRecord::Base
  belongs_to :status
end
{% endhighlight %}

This way I could check whether an `OperationRequest` was approved or denied
with something like `@operation_request.status.approved?` or
`@operation_request.status.denied?`.

But really what I want to know is whether or not the *operation request* was
approved/denied, and, although that result is dependent on the status,
I don't really care to see how the sausage gets made.

So instead, let's refactor our `OperationRequest` to have it's own set of
`approved?` and `denied?` methods.

{% highlight ruby %}
class OperationRequest < ActiveRecord::Base
  belongs_to :status

  def approved?
    status.content == "approved"
  end

  def denied?
    status.content == "denied"
  end
end
{% endhighlight %}

Now we can interact with our `OperationRequest` class with:
`@operation_request.approved?` or `@operation_request.denied?`.

Okay, this is a little bit better. We don't have to chain methods explicitly when
checking the for an operation requests status. However, we more or less wrote the
same two methods in two different classes, which isn't super DRY.

Let's have one more crack at it:

{% highlight ruby %}
class OperationRequest < ActiveRecord::Base
  delegate :approved?, to: :status
  delegate :denied?, to: :status
end
{% endhighlight %}

Nice. This lets us still interact with our class like before, with `@operation_request.approved?`,
but with significantly less code. By doing this, we're basically telling our
OperationRequest class to create two instance methods of `approved?` and `denied?`
and *delegate* them out to whatever the `Status` class has defined as those methods.
Additionally, we're making it explicit to other developers who come across our code that a requests approval status is
explicitly dependent on it's relationship to status.

Delegation is a handy tool, albeit one that is handy in a select few niche
situations. When extracting responsibilities from your classes, it's always
best to determine whether inheritance or delegation makes more sense for your use
case.
