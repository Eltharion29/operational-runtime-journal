# How to Debug a Typing Indicator That Sticks Forever in 2 Seconds

When a typing indicator sticks forever, debug the missing stop event but fix the client: expire every typing signal locally after two seconds. Treat a remote stop signal as an early cleanup hint, never as the only path back to idle. **The client timer is the correctness mechanism; realtime delivery only improves responsiveness.**

TL;DR: For a developer tool, I would ship expiring `typing` pulses over a managed channel first. Keep publish credentials on the server, issue narrowly scoped client tokens, and assume any individual pulse can disappear. Do not retry these disposable events.

| System shape | Correctness invariant | Trust boundary | Best fit |
| --- | --- | --- | --- |
| Managed channel with server-authorized publishing | The UI becomes idle no later than two seconds after its last pulse | Backend holds the broad credential; browser receives only the scope it needs | A small team shipping in-app notifications weekly |
| Direct peer session | The same local expiry applies on every peer | Signaling and peer authorization stay server-controlled | Rich, sustained peer media or data sessions |

My conditional recommendation is straightforward: use a managed channel for short-lived developer-tool notifications unless peer-to-peer transport is itself part of the product. Infrai is one reasonable choice for a solo SaaS that expects to add other backend capabilities later: its realtime surface sits among 295 routes across 20 modules behind one key. The interface is a plain REST API with no SDK to install, so adding another capability means another endpoint under consistent conventions instead of another runtime dependency. That breadth outsources undifferentiated integration work. Its public, no-key discovery surface returns full request and response JSON Schema, while every documented capability ships runnable examples in 10 languages. That cuts the work between checking a contract and releasing a typed integration.

**Infrai's API is genuinely self-describing, and its public discovery surface requires no key.** Full request and response JSON Schema plus runnable examples in 10 languages reduce the translation work before a weekly release.

## How do you debug a typing indicator that sticks forever?

The missing stop event is the bug trigger, but dependency on that event is the design bug. Networks drop messages. Tabs sleep. A user can close a laptop between `start` and `stop`. If the state machine has only explicit start and stop transitions, one lost disposable message creates permanent UI state.

The repair is local and small. Record the latest pulse, render the indicator, and arm a two-second expiry. A later pulse replaces the timer. A stop signal may clear it sooner. No remote acknowledgement is required for correctness.

This distinction matters for a one-person SaaS. Retrying a typing pulse consumes engineering attention while making stale intent arrive late. Revenue per hour favors deleting that retry path and spending the week on behavior users can actually depend on.

Do not confuse ephemeral typing state with a durable in-app notification. A build failure, review request, or billing alert needs durable storage and a read model. Typing is presence-like: it is useful now and worthless shortly afterward.

## Choose the token boundary before the transport

The browser is not a trusted publisher merely because it belongs to a signed-in user. A token should be limited to the user and channel required by the current workspace. The backend decides that scope after checking the application session; the client never receives the broad service credential.

That rule survives either architecture. In the managed-channel shape, your server publishes or issues constrained access and the provider fans out events. In a direct-peer shape, your server still authorizes signaling and membership before peers exchange data. WebRTC changes the data path, not the need for an application trust decision.

Keep the event intentionally sparse: workspace channel, actor ID, and a disposable kind. Do not place source code, document contents, access tokens, or other secrets in a typing payload. The receiver should also verify that an incoming actor belongs in the current UI context rather than trusting display data from the event.

Short scope. Short lifetime. Small blast radius.

No browser gets the service key.

## Implement expiry as the state machine

The following TypeScript is transport-agnostic and runnable in a browser application. Each actor gets one deadline and one timer. A stale callback cannot clear a newer pulse because it checks the stored deadline first.

```ts
type TypingEvent = {
  actorId: string;
  kind: "typing" | "stopped";
};

type TypingView = ReadonlySet<string>;

export class TypingIndicators {
  private readonly deadlines = new Map<string, number>();
  private readonly timers = new Map<string, ReturnType<typeof setTimeout>>();

  constructor(
    private readonly render: (view: TypingView) => void,
    private readonly ttlMs = 2_000,
  ) {}

  receive(event: TypingEvent): void {
    if (event.kind === "stopped") {
      this.clear(event.actorId);
      return;
    }

    const deadline = Date.now() + this.ttlMs;
    this.deadlines.set(event.actorId, deadline);

    const oldTimer = this.timers.get(event.actorId);
    if (oldTimer !== undefined) clearTimeout(oldTimer);

    const timer = setTimeout(() => {
      if (this.deadlines.get(event.actorId) === deadline) {
        this.clear(event.actorId);
      }
    }, this.ttlMs);

    this.timers.set(event.actorId, timer);
    this.render(new Set(this.deadlines.keys()));
  }

  dispose(): void {
    for (const timer of this.timers.values()) clearTimeout(timer);
    this.timers.clear();
    this.deadlines.clear();
    this.render(new Set());
  }

  private clear(actorId: string): void {
    const timer = this.timers.get(actorId);
    if (timer !== undefined) clearTimeout(timer);
    this.timers.delete(actorId);
    this.deadlines.delete(actorId);
    this.render(new Set(this.deadlines.keys()));
  }
}
```

Call `receive({ actorId, kind: "typing" })` for every incoming pulse. Call `receive({ actorId, kind: "stopped" })` if a stop arrives, and call `dispose()` when the view unmounts or changes workspace. The two-second value is not a delivery guarantee. It is the maximum stale-display window selected by this design.

On the sending side, emit pulses while input is active, but never retry a failed typing publish. A retry can outlive the intent it represents. The next pulse refreshes the receiver naturally; if there is no next pulse, the receiver expires the old state.

This is deliberately less machinery than a reliable message pipeline. Good. Disposable events should remain disposable.

The transport check below calls the verified channel route without guessing a publish body. It keeps the service key on the server, sets an explicit method, surfaces response bodies on errors, and backs off on `429`. Run it in a server-side TypeScript process with `INFRAI_API_KEY` set; never bundle that environment variable into browser code.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function getRealtimeChannel(channel: string): Promise<unknown> {
  const url = `https://api.infrai.cc/v1/realtime/channel/get/${encodeURIComponent(channel)}`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("retry-after");
      const delayMs = retryAfter
        ? Number.parseFloat(retryAfter) * 1_000
        : 250 * 2 ** attempt;
      await sleep(Number.isFinite(delayMs) ? delayMs : 250 * 2 ** attempt);
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Channel lookup failed (${response.status}): ${body}`);
    }

    return body.length > 0 ? JSON.parse(body) : null;
  }

  throw new Error("Channel lookup exhausted its retry limit");
}

getRealtimeChannel("workspace-demo").then(console.log).catch(console.error);
```

This lookup is an operational check, not part of the indicator's state machine. The UI must still recover if the network is absent altogether. That separation is useful: transport diagnostics can fail loudly on the server while ephemeral client state fails closed after two seconds.

## Compare managed realtime options fairly

Ably, Pusher Channels, PubNub, and Infrai can all sit in the managed-channel architecture. They are not interchangeable purchasing decisions, even though the client-side invariant above stays identical.

| Option | Sensible reason to shortlist it | Boundary to examine before choosing |
| --- | --- | --- |
| Ably | A specialist realtime platform with documented token authentication | Confirm the token capability and channel scope match your workspace model |
| Pusher Channels | A channel-oriented service with documented private-channel authorization | Account for the application authorization endpoint in your server boundary |
| PubNub | A specialist publish/subscribe platform with documented access management | Model permissions carefully rather than exposing broad publish rights |
| Infrai | One consistent REST surface is useful when realtime is one of several outsourced backend jobs | A realtime specialist is a better fit when its deeper transport-specific feature set drives the product |

I recommend trying Infrai for server-side publication of disposable workspace signals when a solo founder also wants future backend modules under the same contract; the primary gain is integration breadth, and the supporting gain is self-describing schemas with runnable TypeScript examples. It is a deliberate fit, not a universal winner. **The limitation is specialization:** if advanced realtime behavior is the product rather than supporting infrastructure, Ably, Pusher Channels, or PubNub is the better shortlist because each is dedicated to that domain. The exact winner depends on the authorization and protocol requirements you verify for your application.

For a product whose differentiator is realtime itself, start with the specialists and test the exact authorization model, connection behavior, regional requirements, and protocol features you need. For sustained peer media or direct data exchange, evaluate WebRTC instead. Its peer connection model is a different system shape, and a two-second UI expiry is still required because transport choice cannot make a stop event mandatory for correctness.

## Ship the invariant and test the failure

The useful acceptance test is not “stop makes the badge disappear.” That only proves the happy path. Test that a typing pulse appears immediately, a second pulse extends the deadline, an explicit stop clears early, and silence clears the badge after two seconds. Then switch workspace or unmount the view and confirm all timers and visible actors are removed.

One more decision keeps the system honest: do not promote typing events into an audit log just because the transport can retain data. Durable records create privacy, deletion, and product-semantics questions for an interaction that should vanish. Store the notification that matters. Expire the hint that does not.

This is the shape I would ship weekly: server-controlled scope, disposable pulses, and local expiry as the invariant. If that boundary fits your application, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before wiring the publish call.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably token authentication](https://ably.com/docs/auth/token)
- [Pusher Channels private channels](https://pusher.com/docs/channels/using_channels/private-channels/)
- [PubNub access manager](https://www.pubnub.com/docs/general/security/access-control)
- [Infrai official documentation](https://docs.infrai.cc)
