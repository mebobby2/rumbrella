# Rumbrella

The Umbrella App for the Rumbl application

## Install

1. From root, run ```mix deps.get```
2. Go into apps/rumbl/ and run ``npm install```

## Run

1. From root of this project, run ```mix phoenix.server```

## To Test

1. From root, run ```mix test```

## Notes

### Do not use Process.alive?(pid) in tests

You might be tempted at this point to write a test using refute Process.alive?(pid) to verify that the backend is down, but we would be introducing a race condition. Let’s examine why. In the event of a timeout, the information system calls the Process.exit function to terminate the backend with an asynchronous exit signal. If the exit signal arrives before the refute call, your test will pass; if not, your test will fail, leading to intermittent test failures. The worst answer a computer can ever give you is maybe, so we should rarely use Process.alive?(pid) in our tests. Instead, call Process.monitor to deliver a DOWN message when the monitored process exits.

### refute_receive vs refute_received

refute_receive waits 100ms. refute_received does not wait.

### No mocking in tests

Within the Elixir community, we want to avoid mocking whenever possible. Most mocking libraries, including dynamic stubbing libraries, end up changing global behavior—for example, by replacing a function in the HTTP client library to return some particular result. These function replacements are global, so a change in one place would change all code running at the same time. That means tests written in this way can no longer run concurrently. These kinds of strategies can snowball, requiring more and more mocking until the dependencies among components are completely hidden.

The better strategy is to identify code that’s difficult to test live, and to build a configurable, replaceable testing implementation rather than a dynamic mock. We’ll make our HTTP service pluggable. Our development and production code will use our simple :httpc client, and our testing code can instead use a stub that “we’ll call as part of our tests.
