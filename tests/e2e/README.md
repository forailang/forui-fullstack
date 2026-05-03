# End-to-end tests

Reserved for tests that exercise the assembled app — for example, booting
the server, serving `build/web/`, and driving the live RPC path with a
browser harness.

Unit tests live inline in `src/` (`test ... end` blocks beside the code).
This folder is for the cross-boundary tests that don't fit there.
