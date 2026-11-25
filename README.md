# apogee-mail

Mail and notifications for [Apogee](https://github.com/venomous-maker), as
Laravel's:

- **Mailables:** a view, Markdown in a mail theme with components, HTML or
  text, plus attachments, tags and headers.
- **Transports:** SMTP, log, array, sendmail, failover and round robin.
- **Mailers:** several by name. Mail can be queued through apogee-queue, and
  there's a fake for tests.
- **Notifications:** over mail, a database inbox, a log, or channels of your
  own, sent now or on the queue.

```typescript
import "std"
import "apogee-mail"

class OrderShipped extends Mailable {
    public orderId: long = 0
    constructor(orderId: long = 0) { self.orderId = orderId }
    fn build() {
        self.subject("Order #${self.orderId} shipped")
            .markdown("# On its way!\n\nOrder **#${self.orderId}** left the warehouse.\n\n@button(https://shop.test/orders/${self.orderId})Track it@endbutton")
            .attach("/var/invoices/${self.orderId}.pdf")
    }
}

let mailer = new Mailer("smtp", SmtpTransport.fromUrl("smtp://user:secret@smtp.example.com:587"), container)
mailer.alwaysFrom = Address.parse("Shop <hello@shop.test>")
mailer.to("ada@example.com", "Ada").send(new OrderShipped(42))
mailer.to("ada@example.com").queue(new OrderShipped(42).onQueue("mail"))    // a worker sends it
```

## Mailables

A mailable fills itself in, in `build()`. The container fills in
`build()`'s parameters.

| Part | Methods |
|---|---|
| Addresses | `from`, `to`, `cc`, `bcc`, `replyTo`: `"Name <email>"`, or the email and the name |
| Subject | `subject(text)`; a mail without one gets its class name as words ("Order Shipped") |
| Body | `view(name, data)` + `textView(name)` (apogee-view templates); `html(…)` + `text(…)`; `markdown(source)` or `markdown(viewName, true)` |
| Data for views | `with(key, value)`, `withAll(map)` |
| Attachments | `attach(path, name, mime)`, `attachData(bytes, name, mime)`; the type is guessed from the extension |
| Extras | `tag(t)`, `metadata(k, v)` (as `X-Tags` and `X-Metadata-*` headers), `header(name, value)`, `priority(1..5)` |
| Queueing | `onQueue`, `onConnection`, `delay(seconds)`, `mailer(name)` |

- **Plain-text part:** a mail with only HTML gets one made from it, with links
  as `text (url)`.
- **Markdown with view tags:** Markdown source that holds `{{ }}` is rendered
  as a view first.
- **Tests:** `hasTo`, `hasCc`, `hasBcc`, `hasSubject`, `hasAttachment` and
  `hasTag` read a mailable.

## Markdown and the theme

`Markdown.toHtml` covers CommonMark's everyday parts:

- **Blocks:** headings, lists (nested, ordered, task lists), quotes, fenced and
  indented code, rules, GitHub tables, and HTML blocks.
- **Inline:** emphasis, strike, code, links, images, autolinks and escapes.

`Markdown.toText` gives the same text readable without markup.

**The mail theme** (`mailer.theme`) puts Markdown mail into a layout with
inline CSS, since mail clients ignore style sheets:

- **What it uses:** the app's name and URL, an accent colour, a button colour,
  and a footer.
- **Components:**
  - `@button(url)Label@endbutton`, with `success` or `error` as a second
    argument for green or red
  - `@panel … @endpanel`
  - `@subcopy … @endsubcopy`

The text part turns a button into `Label: url`.

## Transports

| | |
|---|---|
| `SmtpTransport(host, port)` / `SmtpTransport.fromUrl("smtp://user:pass@host:587?security=none")` | std's SMTP client: STARTTLS or TLS (`smtps://`), login, a Message-ID per mail |
| `LogTransport(path)` / `LogTransport.to(fn)` | the MIME text, to a file or a function (development) |
| `ArrayTransport` | kept in memory (tests, previews) |
| `SendmailTransport(command)` | the local `sendmail -t -i` |
| `FailoverTransport([a, b])` | the first that sends |
| `RoundRobinTransport([a, b])` | in turn, falling over to the others |

## Mailers

- **Recipients:** `mailer.to(…).cc(…).bcc(…)`, then `.send(m)`, `.queue(m)` or
  `.later(seconds, m)`.
- **Quick mail:** `mailer.raw(to, subject, text)` sends one without a class.
- **Previews:** `mailer.preview(m)` gives the HTML, for a route while you
  design a mail.
- **Defaults:** `alwaysFrom` and `alwaysReplyTo`. `alwaysTo` redirects every
  mail, so in development no customer gets a test mail.
- **Events:** `MessageSending` comes first; a listener can change the mail or
  `cancel()` it. `MessageSent` carries the id. Both fire when `mailer.events`
  is set.
- **Queueing:** `mailer.queueManager` sends through apogee-queue as a
  `SendQueuedMailable` job, which retries 3 times with backoff. It needs a
  `MailManager` in the container to find the mailer by name.
- **Several mailers:** `MailManager` holds them by name, with a default.

**Testing:**

```typescript
mailer.fake()
code.thatSendsMail()
mailer.assertSent::<OrderShipped>(fn(m: OrderShipped) -> bool { return m.hasTo("ada@example.com") && m.orderId == 42 })
mailer.assertSent::<OrderShipped>(null, 1)       // exactly once
mailer.assertQueued::<Welcome>()   mailer.assertNotSent::<Invoice>()   mailer.assertNothingSent()
```

## Notifications

```typescript
class InvoicePaid extends Notification {
    public amount: long = 0
    constructor(amount: long = 0) { self.amount = amount }
    fn via(n: Notifiable) -> Array<string> { return ["mail", "database", "sms"] }
    fn toMail(n: Notifiable) -> MailMessage {
        return new MailMessage().subject("Payment received").greeting("Hi!").line("We got ${self.amount} KES.")
            .action("View receipt", "https://x.io/r").line("Thanks for your business.")
    }
    fn toArray(n: Notifiable) -> HashMap<string, any> { return {"amount": self.amount} }
}

let notifications = new NotificationSender()
notifications.mailer = mailer
notifications.extend("sms", fn(n: Notifiable, x: Notification) { sms.send(n.routeNotificationFor("sms"), "Paid") })
notifications.send(user, new InvoicePaid(500))
notifications.send(user, new InvoicePaid(500).onQueue("notifications"))      // on the queue (sender.queueManager)
notifications.route("mail", "ops@example.com").route("sms", "+254700000000")   // no user needed
```

**Who can be notified:** anything that implements `Notifiable`:

- `routeNotificationFor(channel)` says where each channel sends.
- `notifiableKey()` says who it is.

**Channels:**

| Channel | What it does |
|---|---|
| `mail` | a `MailMessage` (greeting, lines, an action button, salutation, subcopy) in the theme |
| `database` | `toDatabase()` (or `toArray()`) in a `NotificationStore`; methods below |
| `log` | a line per notification |
| your own | added with `extend(name, fn)` |

**Database inbox:** the store is `MemoryNotificationStore`, or
`DatabaseNotificationStore(pool)` for a table. Either way it has
`forNotifiable(key, unreadOnly)`, `markAsRead`, `markAllAsRead`, `delete` and
`unreadCount`.

**Other features:**

- `shouldSend(n, channel)` can stop one channel.
- `NotificationSent` and `NotificationFailed` fire as events.
- The fake: `fake()`, `assertSentTo::<N>(user, check)`, `assertNotSentTo`,
  `assertCount`.

## Tests

```
grav test tests/mail.grav
```

The tests send real SMTP to a small server they start on a local port.
