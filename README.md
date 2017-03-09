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

## Upto
Page 516
Chapter 13
