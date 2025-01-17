---
title: "FAQ"
linkTitle: "FAQ"
weight: 8
description: >

---

### Why does the client-side middleware use Arc<Req>?

If you've paid close attention, you'd notice that in the generated code on the client-side, we wrap the user-provided `Req` into `Arc<Req>` before actually executing the volo_thrift client's `call` method.
However, on the server-side, we directly use `Req`.

The reason for this design is that the client-side requires more complex service governance logic compared to the server-side. 
Especially, some service governance logic conflicts with Rust's ownership model. For instance, if a connection fails, we might need to retry on a different node or even implement more complex timeout-retry logic. 
If we were to directly use `Req`, once we execute the inner service's `call` for the first time, the ownership would have already been moved, preventing us from implementing retry logic.

Additionally, using `Arc` helps us avoid problems caused by concurrent access under middleware (such as scenarios involving mirror/diff), without going into excessive detail here.

Furthermore, the client-side itself shouldn't modify `Req`, hence eliminating the need for mutable access.

On the other hand, the server-side doesn't have such complex usage scenarios, and ultimately, ownership needs to be passed to the user's handler. Therefore, using `Req` ownership directly on the server-side suffices.

### Why is the code generated by volo-cli separately split into the volo-gen crate?

This separation is because Rust's compilation operates on a crate-by-crate basis. Creating the generated code as a separate crate allows for better utilization of the compile cache (IDL generally doesn't change frequently).

### How compatible is it with Kitex?

Volo is fully compatible with Kitex, including functionalities like metadata transmission.

### Where did poll_ready (backpressure) go?

In Tower's Service, there is a method called `poll_ready`, used to ensure downstream Services have sufficient processing capacity before making requests and to provide backpressure when processing capacity is insufficient. 
This is a very ingenious design, and Tower elaborates on the reasons for this design in the article [inventing-the-service-trait](https://tokio.rs/blog/2021-05-14-inventing-the-service-trait).

However, based on our real development experience, we have summarized the following insights:

- The majority of `poll_ready` implementations directly call `self.inner.poll_ready(cx)`; the remaining implementations are even simpler, directly returning `Poll::Ready(Ok(()))`.
- `poll_ready` generally does not actually check the load across services (i.e., it does not send a request downstream asking, "Hey buddy, can you handle more?"), so it usually involves evaluating certain conditions within local middleware (such as Tower's example of rate limiting middleware).
- Based on the above two points, almost all `poll_ready` scenarios can achieve the same effect directly within the `call` method. In practice, the outer layer of the `service` simply waits when returning `Poll::Pending`, so it is more ergonomic to write code using `async-await` directly.
- As for the potential issue of resource wastage, intercepting middleware is generally best placed as early as possible. Therefore, by properly arranging the order of middleware, this issue can be resolved.

Therefore, following the principle of "if it isn't necessary, don't include it" and to enhance usability, we have ultimately decided not to include the `poll_ready` method in our design.
