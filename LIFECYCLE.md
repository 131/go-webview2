# Host-owned WebView lifetime

This fork wraps the supported Microsoft WebView2 API. It does not modify the
browser engine, loader DLL, SDK, process model, or runtime. Its module path stays
`github.com/jchv/go-webview2` for source compatibility; the Git remote identifies
this explicitly maintained fork as `131/go-webview2`.

## Why more than hide/show?

A resident application can own a native UI message loop while creating a browser
only when a user opens a window. Closing the browser window must release the
browser without terminating that loop. These are separate lifetimes:

- The host owns its UI thread, message pump, tray, and application services.
- Each WebView owns its controller, COM references, window, and binding queue.
- WebView2 owns its browser processes and decides when they exit.

Hiding a window changes visibility, not lifetime. Destroying the HWND alone is
also not an explicit release of the wrapper's controller and COM references.
The upstream wrapper's WM_DESTROY handler posts WM_QUIT, coupling window
lifetime to application lifetime. Its binding queue is normally drained by its
own Run method. Those assumptions suit its demo but do not meet a host-owned
loop contract.

There is no tray code in this library. The tray is simply one reason a consumer
might want an independent host lifetime.

## Changes and their scope

| Change | Reason | When needed |
| --- | --- | --- |
| `ICoreWebView2Controller.Close` | Expose the supported native close operation, checking its HRESULT. | Explicit controller shutdown. |
| `Chromium.Close` | Close the controller, then release the wrapper's WebView, controller, and environment references. Drop consumer callbacks. | Release/recreate a browser. |
| Reference ownership | The controller getter returns an owned WebView reference; a second AddRef retains an extra reference. Settings acquired for setup also need Release. Callback parameters retained beyond the callback are AddRef'd. | Correct resource accounting across repeated creation. |
| `WebView.Close` | Destroy the HWND without WM_QUIT, remove its context, and discard pending binding work. Successful close is idempotent; native close failure preserves references for retry. | Keep the host alive after browser shutdown. |
| `WebView.ProcessDispatch` | Let a host-owned pump drain the existing per-WebView binding queue. | A consumer using its own pump rather than Run. |
| Closed-state guards | Ignore pending dispatch/Eval and late initialization completions belonging to the old instance. | Async completion crossing a close/reopen boundary. |
| Initialization cancellation | Return false on interruption, preserve WM_QUIT for the outer host pump, reject failed/null COM completions, and decode HRESULT as signed 32-bit on both architectures. | Quit/failure during creation. |
| Shared native window callback | Reuse one Windows callback instead of consuming another callback slot per created window. | Repeated creation. |
| `WindowOptions.Hidden` and `WebView.Show` | Initialize parent and controller hidden; explicitly reveal the controller after showing the parent. Visibility calls check HRESULT, not GetLastError. | Hidden construction/publishing, **not** a prerequisite for closing a browser. |

Hidden construction is a presentation choice, not proof that a dynamic WebView
requires a new windowing architecture. A consumer willing to display the window
while initialization happens does not need this option.

Microsoft documents controller Close as the way to close a WebView and release
its event handlers, independently of the native application window. Closing all
relevant controllers permits associated processes to shut down. **Successful
Close does not mean that every browser process has already exited when it
returns.** Processes can be shared by WebViews using the same user data folder;
tests must isolate their data and wait for exit, rather than kill processes.

## Consumer contract

The WebView interface gains Close, Show, and ProcessDispatch. Ordinary callers
using the constructors keep their existing calls, but custom implementations or
mocks of that interface must add these methods. Keeping the module path does not
make this interface extension universally source-compatible.

Create, show, drain dispatch, and close on the creating STA UI thread with a
working message pump. Dispatch may enqueue work from another goroutine, but
that does not make other WebView methods thread-safe.

For a consumer with its own pump:

1. Create a WebView on the UI thread, optionally using Hidden.
2. If hidden, show its native parent first, then call WebView.Show. This method
   reveals the controller; it does not show the parent for the host.
3. Call ProcessDispatch for the wrapper's WM_APP thread message (`0x8000`), and
   drain pending work after any native modal loop that consumed that message.
4. Intercept the window-close request in the host and call Close on the view.
   The default window-close behavior remains the standalone Run/Terminate
   behavior; simply calling Destroy is not the host-owned lifetime contract.
5. Discard the old view after a successful Close. Create a new instance to reopen.
6. Only the host's Quit operation stops the host message loop.

Respect Microsoft's reentrancy rule: do not synchronously open another WebView
or a modal loop from inside a WebView2 callback. Schedule that work after the
callback returns. This patch does not make unsupported reentrancy safe.

## Alternatives considered

| Alternative | What it avoids | Tradeoff / assessment |
| --- | --- | --- |
| Keep the WebView and only hide/show it | Dependency lifetime work. | Simplest if acceptable, but retains browser resources after the GUI closes. Does not meet the requested release contract. |
| Separate UI process | Custom in-process controller lifetime: normal process exit releases the UI. | Legitimate fallback, not inherently excessive. Requires an IPC protocol for actions/state, authentication, disconnection, and reopening; changes the current single-process architecture. Not implemented here. |
| Use only the original wrapper's Destroy | New dependency API. | Does not explicitly release the wrapper-owned COM references and posts WM_QUIT; not equivalent to independent browser shutdown in the inspected upstream revision. |
| Use a different maintained Go binding | Owning this fork. | Wails uses `wailsapp/go-webview2` v1.0.22 in its inspected v2 go.mod. That binding exposes Show/Hide and GetController, but the inspected controller source does not expose Close. Its API is not the original top-level WebView API. Worth evaluating, but not a demonstrated drop-in lifetime solution. |
| Use the C++ webview binding | Custom Go COM lifetime code. | Its inspected Windows backend already closes the controller and releases COM objects. A serious alternative, with a C/C++/cgo toolchain and host-loop integration to evaluate. No migration trial was performed. |
| Use Microsoft's native SDK directly | Another wrapper's assumptions. | Fully supported API, but moves windowing/COM/binding responsibilities into the application; more code owned by us, not less. |
| Submit the bounded changes upstream | Long-term fork maintenance. | Desirable if accepted. Acceptance and release are external dependencies, not assumed. |

The chosen local implementation is an explicit, bounded fork because the
operator named this repository as the work scope. That is not evidence that all
alternatives are worse, nor authorization to publish or migrate an application.

## Validation and limits

Windows tests exercise explicit close ordering and failure, idempotency, dropped
binding completions, initialization failure/cancellation, settings Release, and
HRESULT visibility handling. COM call tests use synthetic vtables, not a real
browser. They verify wrapper behavior, not the browser's implementation.

Build on Windows amd64 and 386. Run `go test ./...` on Windows.
For vet, use `go vet -unsafeptr=false ./...`: the inherited HWND return and native
LPARAM conversion trigger the unsafe-pointer analyzer. Other analyzers remain
enabled. Do not describe this as a clean default vet run.

Before migrating a consumer, additionally run isolated real-browser tests:
visible renderer children after publishing, repeated open/close, late replies,
Quit during initialization, and browser-process exit after the last controller
closes. These integration checks belong in the consuming application as well as
any future native example. This repository currently has no standalone
real-browser lifecycle test.

This is not a complete COM-wrapper audit. Other inherited APIs still use legacy
error handling, and callback/Go-pointer lifetime assumptions remain upstream's.
No claim is made that this patch fixes every initialization or resource issue.

## Primary sources inspected

- [Microsoft controller API: Close and visibility](https://learn.microsoft.com/en-us/microsoft-edge/webview2/reference/win32/icorewebview2controller)
- [Microsoft threading and reentrancy contract](https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/threading-model)
- [Microsoft process model](https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/process-model)
- [Wails v2 dependencies](https://github.com/wailsapp/wails/blob/master/v2/go.mod)
- [Wails binding Chromium v1.0.22](https://github.com/wailsapp/go-webview2/blob/v1.0.22/pkg/edge/chromium.go)
- [Wails controller v1.0.22](https://github.com/wailsapp/go-webview2/blob/v1.0.22/pkg/edge/ICoreWebView2Controller.go)
- [C++ webview Windows backend](https://github.com/webview/webview/blob/master/core/include/webview/detail/backends/win32_edge.hh)

Moving branch sources describe the inspected snapshot, not a permanent promise
about those projects' capabilities.
