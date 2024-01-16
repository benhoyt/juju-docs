> <small> [Event](/t/6361) > [List of events](/t/6657) > [Lifecycle events](/t/4455) > `<container>-pebble-custom-notice`</small>
>
> Source: [`ops.PebbleCustomNoticeEvent`](https://ops.readthedocs.io/en/latest/index.html#ops.PebbleCustomNoticeEvent)

<!--
> - [How to operate workload containers](/t/6542)
> - [Interact with Pebble](/t/4554)
-->

Juju emits the `<container>-pebble-custom-notice` event when a Pebble notice of type "custom" occurs for the first time or repeats. There is one `<container>-pebble-custom-notice` event for each container defined in `metadata.yaml`. This event allows the charm to respond to custom events that happen in the workload container.

[note] 
This event is specific to Kubernetes sidecar charms and is only ever fired on Kubernetes deployments.
[/note]

To learn more about Pebble notices, and see an example of handling a custom notice, read the [full Pebble Notices documentation](/t/4554#heading--custom-notices).
