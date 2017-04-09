---
title: Stripe webhooks in Connect applications
tags:
- programming
- stripe
- webhooks
- ruby
---

![](<../images/stripe-webhooks.png>)

I have recently implemented Stripe webhook integration for my application that uses [Stripe Connect](https://stripe.com/connect) with managed accounts. As it turned out, the process for working with webhook events in Connect applications is slightly different, and there is no official documentation or public blog posts that cover these nuances. This short write-up is meant to bridge this gap.

So if you’ve never worked with Stripe webhooks, here’s what a typical webhook handler looks like (this Ruby snippet comes from Stripe documentation):

```ruby
post "/my/webhook/url" do
  # Retrieve the request's body and parse it as JSON
  event_json = JSON.parse(request.body.read)

  # Verify the event by fetching it from Stripe
  event = Stripe::Event.retrieve(event_json["id"])

  # Do something with event

  status 200
end
```

The process is very straight-forward — you parse the request body into JSON that represents an Event object (its properties are described in the [API documentation](https://stripe.com/docs/api#event_object)), verify Event object by retrieving it using `id` of the parsed event, then you usually do something useful with this data and return `200` response to let Stripe know that we’ve successfully received the event.

However, if you try the same approach for events related to Connect accounts (like `account.updated`), you’ll get an error — `Stripe::InvalidRequestError: No such event`. So why is that?

The reason for this error is that some events don’t belong to your main Stripe account, but rather to connected accounts you manage. For example, when a new customer is created or a charge is refunded, these events are associated with your main Stripe account, but events related to updates of a managed account belong to the managed account itself. When retrieving such events, you need to specify explicitly the account they belong to.

In practice, it means that your Event payload will have a new `user_id` property with the identifier of a connected account. So when you make a call to retrieve Event details, you need to supply it as well:

```ruby
event = Stripe::Event.retrieve(event_json["id"], stripe_account: event_json["user_id"])
```

With this small change, everything will work as expected.
