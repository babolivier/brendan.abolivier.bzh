---
title: Supporting Exchange and beyond
description: "In 2023, my team started to work on an ambitious project: supporting Microsoft Exchange in Thunderbird. In this second (and last) instalment of this duology, I'll  dive deeper into the practical and technical aspects of this project, from building new code infrastructure to implementing the required functional features."
tags:
  - thunderbird
  - email
  - exchange
  - netscape
  - ews
  - graph
publishDate: 2026-06-08T00:00:00+00:00
draft: false
thumbnail: /exchange-pt-2/thumbnail.png
---

Hey there, me again!

This is the second and last part in a duology around [Thunderbird](https://www.thunderbird.net)'s project to support Microsoft Exchange. About a month ago, I [published a blog post](https://brendan.abolivier.bzh/exchange-pt-1/) that shared some of the project's behind the scenes (as well some of my personal thoughts), in which I was focusing a lot on the high-level picture and how we defined a direction for the project.

However, that article covers discussions that happened mostly around the start of the project, and didn't get very technical. Although some of these conversations might have spanned over a longer period of time, the more practical, day-to-day progress of the project isn't really mentioned - this is what this second part is for!

We've already looked at where we were going, now let's focus on _how_ we're getting there.

# Laying the foundations

Our first task was to ask ourselves what code infrastructure we were missing, and although there was already some functionality available to us in C++, on the Rust side of things (where our protocol client was going to live) things kind of looked like the barest of wastelands. After all, this was the very first time we were going to write _any_ Rust specifically for Thunderbird.

We started by defining a baseline for what our protocol client needed to do. EWS traffic consists of of XML (SOAP) requests sent over an HTTP(S) connection, we therefore needed the ability to:

- send out network traffic, and do so asynchronously
- build HTTP request and receive responses
- serialise request data into XML, and deserialise XML responses

In addition, we needed whatever solutions to those requirements to be low on boilerplate. EWS lists many operations and even more types, and by this point we didn't have such a precise idea of how many of those we would need.

## Native asynchronicity

Let's start by looking at the first point on that list (or at least part of it): performing asynchronous operations. Like I mentioned in the first part of this series, asynchronous operations in Thunderbird rely heavily on callbacks functions to be called whenever a change occurred with an operation. On the other hand, Rust uses an async/await syntax somewhat similar to what also exists in JavaScript or Python, so we'll need a way to link the two together.

A long-standing issue of compatibility between Rust and C++ has been the lack of a stable ABI on either side making it difficult to carry types more complex than what the C ABI supports over the language boundary. A few solutions exist to address this, [cxx](https://cxx.rs/) being a pretty well-known and generic one. In our case, however, we'll use something called [XPCOM](https://firefox-source-docs.mozilla.org/xpcom/index.html) (Cross-Platform Component Object Model), which is a custom Mozilla cross-language compatibility layer used specifically by Firefox and Thunderbird.

XPCOM uses interfaces written in an Interface Description Language called [XPIDL](https://firefox-source-docs.mozilla.org/xpcom/xpidl.html). Interfaces defined this way can be implemented in either JavaScript, C++ or Rust, and once registered with a contract ID (a unique human-readable string, e.g. `@mozilla.org/my-component;1`) each implementation can also be instantiated from any of these languages (note that XPIDL also defines attributes that can constrain which language can implement or use a given interface).

In our case, the interface we're interested in is called [`nsIChannel`](https://searchfox.org/firefox-main/rev/6087999d87e808ec39f260293afd6cc27b6c5944/netwerk/base/nsIChannel.idl) (fun fact: a number of interface names in both Firefox and Thunderbird start with `ns`, which stands for "Netscape", the company the Mozilla project originated from). In Mozilla terminology, a channel represents a resource fetch, which can be performed both [synchronously](https://searchfox.org/firefox-main/rev/6087999d87e808ec39f260293afd6cc27b6c5944/netwerk/base/nsIChannel.idl#144-158) and [asynchronously](https://searchfox.org/firefox-main/rev/6087999d87e808ec39f260293afd6cc27b6c5944/netwerk/base/nsIChannel.idl#160-192).

This is interesting to us, because Necko, Firefox's networking component which we also inherit, uses channels to perform HTTP requests. We _could_ use an off-the-shelf HTTP crate like [`reqwest`](https://crates.io/crates/reqwest), but using Necko comes with a couple of main advantages:

- requests and responses can be visualised in Thunderbird's devtools (which is _another_ thing we inherit from Firefox)
- requests sent through Necko adhere to users' network and web content settings, such as cookie settings, without specific support needed on the Thunderbird side

When when a channel is "opened" asynchronously, it takes an [`nsIStreamListener`](https://searchfox.org/firefox-main/rev/6087999d87e808ec39f260293afd6cc27b6c5944/netwerk/base/nsIStreamListener.idl) as its listener. This is an interface with three methods (two of them inherited from [`nsIRequestObserver`](https://searchfox.org/firefox-main/rev/6087999d87e808ec39f260293afd6cc27b6c5944/netwerk/base/nsIRequestObserver.idl#13)):

- `OnStartRequest`, which notifies that the operation has started
- `OnDataAvailable`, which notifies that new data for the operation is available, and is called with a stream that contains this new data
- `OnStopRequest`, which notifies that the operation has stopped, and is called with a status code indicating whether the operation has succeeded or failed

On the Rust side of things, the async/await syntax is powered by the [`Future`](https://doc.rust-lang.org/std/future/trait.Future.html) trait. This trait defines an associated type `Output`, which is the type to which the future resolves (once the operation has completed), as well as a `poll` method that is called by the task scheduler to attempt to resolve the future into its `Output` type.

This `poll` method runs synchronously, and returns a variant of the [`Poll`](https://doc.rust-lang.org/std/task/enum.Poll.html) enum; `Poll::Pending` if the operation has not completed yet, and `Poll::Ready` if the operation has successfully completed, with the result of type `Self::Output` wrapped inside it. The `poll` method might be called every now and then by the scheduler to check on the state of the operation, but futures also have the option to tell the scheduler that it has finished and should be checked back on using a [`Waker`](https://doc.rust-lang.org/std/task/struct.Waker.html) (which can be retrieved from the [`Context`](https://doc.rust-lang.org/std/task/struct.Context.html) passed to every `poll` call).

With this in mind, here's what we did:

- We defined a custom implementation of `nsIStreamListener`, called [`BufferingStreamListener`](https://searchfox.org/comm-central/rev/75f4a571890aab694d6574e5c0a193cdb87a3e96/rust/xpcom_async_glue/src/async_channel_opener/buffering_listener.rs), which concatenate all of the data retrieved through `OnDataAvailable` into its inner bytes buffer (`Vec<u8>`), and records the status code given to `OnStopRequest`.
- We defined the struct [`AsyncChannelOpener`](https://searchfox.org/comm-central/rev/75f4a571890aab694d6574e5c0a193cdb87a3e96/rust/xpcom_async_glue/src/async_channel_opener.rs#24-36), which can be constructed from an `nsIChannel`, and implements the `Future` trait.

Once `AsyncChannelOpener::poll` is called for the first time, it instantiates a new `BufferingStreamListener` and calls `nsIChannel::AsyncOpen` on its inner channel with that listener. `BufferingStreamListener` is responsible for letting the scheduler know when to poll again, so it holds a `Waker` for the task, which `AsyncChannelOpener` updates each time `poll` is called. Once the request completes and the listener's `OnStopRequest` is called, the `Waker` is used and the `AsyncChannelOpener` is polled again, at which point it returns a `Result` which contains either the failure code (if the request failed), or a tuple with the `nsIChannel` and the contents of the listener's buffer. We include the channel in the return value because the consumer might need to use it to read extra information, such as the response's status code for HTTP traffic.

A quick aside: we initially built this code to be very generic and be possibly extended in the future with support for other asynchronous XPCOM operations beyond `nsIChannel::AsyncOpen`. In reality, the need for this never really materialised and this approach could only really support operations that take an `nsIStreamListener` anyway, so we [recently](https://bugzilla.mozilla.org/show_bug.cgi?id=2039586) took steps to simplify it.

## Idiomatic web requests

Now that we can turn an `nsIChannel` into a Rust-native `Future` that can be `await`ed, we need to figure out how to create this channel in the first place. This is done through an XPCOM service that implements the [`nsIIOService`](https://searchfox.org/firefox-main/rev/c90c4a59e34b5c6142691052bbb15b295ea0c37c/netwerk/base/nsIIOService.idl) interface, which offers a `NewChannel` method. This method takes a URI and instantiates a new `nsIChannel` which implementation matches that URI's protocol scheme (a component can be registered as the protocol handler that creates the channels for a given URI scheme in its registration, for example the one we use to create channels for EWS messages is registered [here](https://searchfox.org/comm-central/rev/0fea5c828d2b78e635d2e606a01169efc3f74152/mailnews/protocols/exchange/src/components.conf#61-77)). This means that if we give that method an HTTP(S) URI, it will return a channel that sends HTTP(S) requests.

With that in mind, let's have a look at how we can send a simple HTTP GET request with Necko:

```rust
use std::ptr;

use http::Method;

use nserror::nsresult;
use nsstring::nsCString;
use xpcom::{
    get_service,
    interfaces::{
        nsIContentPolicy, nsIHttpChannel, nsILoadInfo,
        nsIIOService, nsIPrincipal, nsIScriptSecurityManager,
    },
    RefPtr,
};

use xpcom_async_glue::AsyncChannelOpener;

async fn send_request(uri: String) -> Result<Vec<u8>, nsresult> {
    let iosrv = get_service::<nsIIOService>(
            c"@mozilla.org/network/io-service;1"
        ).ok_or(nserror::NS_ERROR_FAILURE)?;

    let scriptsecmgt = get_service::<nsIScriptSecurityManager>(
            c"@mozilla.org/scriptsecuritymanager;1"
        ).ok_or(nserror::NS_ERROR_FAILURE)?;

    // XPCOM methods return an `nserror::nsresult` to indicate
    // success or failure; the actual return value is an pointer
    // passed as an in/out parameter. `getter_addrefs` turns the
    // `nsresult` into a `Result` for us, and also ensures the
    // pointer is correctly integrated within the internal ref
    // counting system.
    let principal: RefPtr<nsIPrincipal> = getter_addrefs(
        |p| unsafe {
            scriptsecmgr.GetSystemPrincipal(p)
        }
    )?;

    // Use a Mozilla-internal string type. It derefs into an
    // `nsACString`, which is the "abstract" type that is generally
    // expected by XPCOM methods.
    let uri = nsCString::from(uri);

    let channel: RefPtr<nsIChannel> = getter_addrefs(
        |p| unsafe {
            iosrv.NewChannel(
                // We need to turn the URI into a
                // `*const nsACString`.
                &raw const *uri,
                ptr::null(),
                ptr::null(),
                ptr::null(),
                // Turns the `RefPtr<T>` into a
                // `*const T`.
                principal.coerce(),
                ptr::null(),
                nsILoadInfo::SEC_ALLOW_CROSS_ORIGIN_SEC_CONTEXT_IS_NULL,
                nsIContentPolicy::TYPE_OTHER,
                p
            )
        }
    )?;

    // Cast the channel into an `nsIHttpChannel`, which inherits
    // from  `nsIChannel` and exposes methods specific to HTTP.
    // There's no guarantee that the underlying `nsIChannel`
    // implementation also implements `nsIHttpChannel`, so we
    // need to catch this as an error at runtime.
    let http_channel = channel
        .query_interface::<nsIHttpChannel>()
        .ok_or(nserror::NS_ERROR_FAILURE)?;

    // Note: the request method defaults to GET, so in practice
    // we might not need this, but this works better for
    // illustrating.
    let method = nsCString::from("GET");
    unsafe {
        http_channel.SetRequestMethod(&raw const *method)
    }.to_result()?;

    let (_channel, bytes) = AsyncChannelOpener::from(channel).await?;

    Ok(bytes)
}
```

If you're like me, you'll notice a few things right off the bat:

- The multiple `unsafe` blocks, because every method from an XPCOM interface implementer might come from C++ or JavaScript and is therefore unsafe
- Pointers, pointers everywhere
- The error handling which leaves a lot to be desired
- That's a bunch of code for a simple GET request, with no body, no headers... what happens when we want to start making more complex requests?

That last question is simple to answer: it gets more complicated. If we wanted to add a body in there, we'd need to wrap our request bytes inside an `nsIInputStream` and cast our channel as an `nsIUploadChannel` to configure the channel to read the request body from that stream. You can see it in action [here](https://searchfox.org/comm-central/rev/0fea5c828d2b78e635d2e606a01169efc3f74152/rust/moz_http/src/request.rs#279-340). You also need to be sure to set the request's method _after_ you set the request body, because setting the request body might change the channel's method. Anyway, it gets annoying really quickly, and we don't want to be doing all of that every time we want to send an HTTP request.

By the way, as you can imagine this would have been even worse if we didn't have `AsyncChannelOpener`; you can see an example of what a (simpler) full XPCOM implementation looks like in [this slide](https://archive.fosdem.org/2024/events/attachments/fosdem-2024-2469-thunderbird-how-to-exchange-rot-for-rust/slides/22416/FOSDEM_-_Thunderbird_How_to_Exchange_Rot_For_Ru_lj0T6hk.pdf#page=18) from a talk a colleague and I gave at FOSDEM in 2023.

Our response to this was to take inspiration from the `reqwest` crate I already mentioned, and build an HTTP client that abstracts all of that and exposes an idiomatic, safe and natively async API to consumers. The result is [`moz_http::Client`](https://searchfox.org/comm-central/rev/0fea5c828d2b78e635d2e606a01169efc3f74152/rust/moz_http/src/client.rs), which is typically used like this:

```rust
use anyhow::Error;
use http::Method;
use url::Url;

use moz_http::Client;

async fn send_request(
    uri: String,
    method: Method,
    body: Vec<u8>
) -> Result<Vec<u8>, Error> {
    let uri = Url::parse(uri)?;

    let client = Client::new();

    // `Client::request` can error if the protocol scheme is
    // neither "http" nor "https".
    let resp = client.request(method, &uri)?
        .header("X-Foo", "Bar")
        // The content type given to `body(..)` will also be
        // used to set the `Content-Type` header.
        .body(&body, "application/octet-stream")
        .send()
        .await?;

    Ok(resp.body())
}
```

Much simpler, isn't it?

## Low-boilerplate XML serialisation

Now that we could send requests, we had to figure out what to put in there. Obviously, we'd like to be able to define the structure of a request's body using Rust types which then get effortlessly serialised into XML, so we set out to find a way to do that.

Unsurprisingly, the Rust ecosystem being quite "young", there isn't a lot of ~~love~~ support for XML available. At least not to the level that can be found for JSON, YAML, TOML, etc. Thankfully, we stumbled upon the [`quick-xml`](https://crates.io/crates/quick-xml) crate that seemed promising enough, and even came with [`serde` support](https://docs.rs/quick-xml/latest/quick_xml/de/index.html).

However, we quickly realised that this would come with too much boilerplate to be viable. Remember: we're potentially dealing with a lot of operations and data structures, so the smallest amount of boilerplate can potentially have huge consequences. In this case, the core issue was that `serde`'s own data model just doesn't fit XML that well. On top of featuring a hierarchy of nested data structures, XML documents also features attributes on elements, namespaces which need to be defined and can come with a prefix, etc., which serde cannot really accommodate.

`quick-xml`'s approach to defining XML attribute is to use `serde`'s `rename` attribute, and specify that any `rename` attribute which value starts with the character `@` is an XML attribute. Relatedly, to set a namespace on a section of the XML document, one must define an XML attribute with that same `rename` hack (e.g. `#[serde(rename = "@xmlns:t")]`), and have the type's consumer set the attribute's value. When serialising, if the namespace comes with a prefix, it also means every element in that section need to be `rename`d with that prefix (e.g. `#[serde(rename = "t:Foo")]`).

This is manageable for deserialisation, because we don't necessarily need to know and look at every single XML attribute, and we can discard namespaces entirely as long as the response's structure is correct (`quick-xml`'s parser correctly links prefixed element names with the right struct field without needing the `rename` trick). But this is obviously not tenable when defining the serialisation of dozens of operations and over a thousand data structures in total.

To be clear, I'm not admonishing `quick-xml`'s approach here. I genuinely think this might be the best solution to trying to fit XML semantics into a data model that isn't designed for them. It's a case of trying to fit a square into a triangle-shaped hole, and there's no perfect solution to it.

We therefore decided to use `quick-xml`'s deserialiser, but to build our own serialiser. We didn't want to reinvent the wheel though, so we decided to still base it on top of `quick-xml`'s writer.

The result is a crate we called [`xml_struct`](https://crates.io/crates/xml_struct). We designed it specifically to serialise EWS requests, meaning it takes some shortcuts by e.g. assuming every element needs to be serialised as PascalCase. We initially aimed to, eventually, make it more generic and support deserialisation as well, but we never got around to doing it. And, frankly, I'm not sure there's a high enough demand to warrant the effort.

This crate exposes two traits:

- `XmlSerialize`, which serialises a value as an XML element, and
- `XmlSerializeAttr`, which serialises a value as an XML attribute

It also comes with a `derive` macro with [a number of attributes](https://docs.rs/xml_struct/latest/xml_struct/derive.XmlSerialize.html) to define the namespace to use, what prefix to apply to enum variants or struct fields, etc. This means we can now define our data structures like this:

```rust
use xml_struct::XmlSerialize;

const MESSAGES_NS_URI: &str =
    "http://schemas.microsoft.com/exchange/services/2006/messages";

#[derive(Clone, Debug, XmlSerialize)]
#[xml_struct(default_ns = MESSAGES_NS_URI)]
pub struct DeleteFolder {
    pub delete_type: DeleteType,
    pub folder_ids: Vec<BaseFolderId>,
}

#[derive(Clone, Debug, XmlSerialize)]
#[xml_struct(text)]
pub enum DeleteType {
    HardDelete,
    MoveToDeletedItems,
    SoftDelete,
}

#[derive(Clone, Debug, XmlSerialize)]
#[xml_struct(variant_ns_prefix = "t")]
pub enum BaseFolderId {
    FolderId {
        #[xml_struct(attribute)]
        id: String,

        #[xml_struct(attribute)]
        change_key: Option<String>,
    },

    DistinguishedFolderId {
        #[xml_struct(attribute)]
        id: String,

        #[xml_struct(attribute)]
        change_key: Option<String>,
    },
}
```

And generating XML bytes, for example for the body of a SOAP request based on the data structures defined above, now looks like this:

```rust
const SOAP_BODY: &str = "soap:Body";

let mut writer = {
    let inner: Vec<u8> = Default::default();
    Writer::new(inner)  // This is a `quick_xml::Writer`.
};

// Write the opening `soap:Body` tag.
writer.write_event(Event::Start(BytesStart::new(SOAP_BODY)))?;

// Define our body and serialise it.
// In practice, we define the body in the consumer, which then
// calls a generic function to perform the serialisation (see below);
// here we do both at the same time as a simplification.
let body = DeleteFolder {
    delete_type: DeleteType::HardDelete,
    folder_ids: vec![BaseFolderId::FolderId {
        id: "Foo".to_string(),
        change_key: None,
    }],
};
body.serialize_as_element(&mut writer, "Foo")?;

// Write the closing `soap:Body` tag.
writer.write_event(Event::End(BytesEnd::new(SOAP_BODY)))?;

let xml_bytes = writer.into_inner();
```

Which then results in the following XML:

```xml
<soap:Body>
    <Foo xmlns="http://schemas.microsoft.com/exchange/services/2006/messages">
        <DeleteType>HardDelete</DeleteType>
        <FolderIds>
            <!-- Note: Defining the namespace for the "t" prefix has
                 been omitted from this example. -->
            <t:FolderId Id="Foo"/>
        </FolderIds>
    </Foo>
</soap:Body>
```

As you can see, the serialisation itself is still a bit more involved than ideal, since we need to generate a few bits of the resulting structure by hand. But we only need to do this in one central place (which, for our EWS types, is [here](https://github.com/thunderbird/ews-rs/blob/1fe0dc6852c1b17820f4e14372bdc1ed008376ee/src/types/soap.rs#L67-L102)), so I'd argue the cost saved when defining the types to serialise is worth the extra cost here.

# Learning and changing

We finally had the building blocks we need to build and send requests to an EWS server, and to receive and understand its responses; now it was finally time to get started on making Thunderbird's range of email features work with EWS!

Obviously this was not as easy a task as I've just made it sound. Thunderbird is a decades-old project with a complex history, and plugging our cool new Rust code into it requires a bit more work than just writing a tiny bit of glue.

We first needed to figure out how to translate the principles we [defined at the start of the project](https://brendan.abolivier.bzh/exchange-pt-1/#fighting-technical-debt-with-intentional-design) into concrete practices and architecture. Like I mentioned last time, we decided to implement the protocol client (which is responsible for every remote operation) in Rust, while we wrote the local side of things (as well as the "glue" that makes our new protocol appear just like any protocol to the rest of Thunderbird) in C++. The C++ side of things interacts with the Rust components through [a "client" XPIDL interface](https://searchfox.org/comm-central/rev/75f4a571890aab694d6574e5c0a193cdb87a3e96/mailnews/protocols/exchange/src/IExchangeClient.idl#29-317), and communication in the other direction happens through [a handful of callback interfaces](https://searchfox.org/comm-central/rev/75f4a571890aab694d6574e5c0a193cdb87a3e96/mailnews/protocols/exchange/src/IExchangeClient.idl#319-555). In practice, this approach allowed us to:

- Benefit from existing generic implementations for local operations (e.g. the local half of deleting a folder, of marking a message as read, etc.) by inheriting from the same base C++ classes as other protocols - and adding to these generic implementations as we went.
- Better control how much of Thunderbird's existing patterns bleed into the new and shiny protocol client by having a clear separation of layers with a clear API to cross the border between them.
- In the future, potentially allow for multiple protocols reusing the same implementation of the C++ components while implementing a completely separate protocol client (that implements the new client interface).

We also made a conscious effort to try and identify which parts of the code we were writing on either side needed to be specific to EWS, and which ones could theoretically be reused by other protocols in the future, both on the C++ and the Rust sides. This led to the addition of a number of shared generic utilities for [managing a folder's local storage](https://searchfox.org/comm-central/rev/75f4a571890aab694d6574e5c0a193cdb87a3e96/mailnews/protocols/common/src/OfflineStorage.h), [saving messages to disk](https://searchfox.org/comm-central/rev/75f4a571890aab694d6574e5c0a193cdb87a3e96/mailnews/protocols/common/src/SaveAsListener.h) or [handling communication failures](https://searchfox.org/comm-central/rev/75f4a571890aab694d6574e5c0a193cdb87a3e96/rust/mailnews_ui_glue), among others. We also identified parts that did not need to depend on Thunderbird's internals to function and could be helpful to reuse in other projects, which led to the publication of a few public Rust crates such as [one](https://crates.io/crates/ews) with types for EWS requests and responses, or [another one](https://crates.io/crates/operation-queue) containing the synchronisation logic we use when performing requests.

Another issue we encountered is one I've also already mentioned in the [first part](https://brendan.abolivier.bzh/exchange-pt-1/) of this series: we were lacking a lot of detailed knowledge around how our own code worked. And although our goal wasn't to make an exact replica of other protocols' implementations, we still needed to know how they fit within Thunderbird's existing protocol <del>patch</del>framework so its front-end, as well as the few existing generic components it already had, could look at EWS like any other supported protocol.

This involved a lot of archaeology into existing Thunderbird code. I didn't measure it in detail at the time, but I tend to consider that every week spend implementing a feature required another week just to figure out how that feature is expected to work; which method of which XPIDL interface being called by generic code, what triggers that call, which callbacks are being used (and how), how output and user feedback is supposed to be communicated, etc. Often this also meant walking down convoluted threads from our fairly spaghetti-fied IMAP support code, though sometimes we got lucky and got away with just using the implementation of local email folders (which is a totally separate thing, because Thunderbird's back-end is like that) as a reference. The upside of all this digging was that we'd come out of it with better understanding of our own code, which ultimately led to [better documentation](https://source-docs.thunderbird.net/en/latest/backend/copy_move.html).

From there, we moved into a steady flow of research and implementations as we tackled each feature after the new one, until we ticked the last box in our email features shopping list, got a [call for testing](https://thunderbird.topicbox.com/groups/planning/T00a28d546ed1b7d1/ews-in-thunderbird-144-0b1-call-for-testing) out to our Daily (i.e. nightly) and Beta users, and eventually [released the feature in Thunderbird 145](https://blog.thunderbird.net/2025/11/thunderbird-adds-native-microsoft-exchange-email-support/) towards the end of last year.

# Doing it all again (except not really)

Great, the feature's released, this means the project's done and we can all now relax and move onto something else, right? Well, not really - this is the project's first big milestone, not its end!

Before I go any further, I want to take a minute or two to mention that although landing email support for EWS does not constitute the whole project, it's still a monumental effort that has required over two years of hard work to bring to completion, performed by a small but incredible team of amazing individuals. I'm immensely proud to have been involved in such a huge undertaking - Thunderbird's first new native protocol in over 20 years! -, let alone be involved in its leadership.

Now, let's look at the next steps. There are two clear ways we can go from here:

- Native Exchange support is currently limited to email, eventually we'll want calendar and address book support as well.
- As I mentioned in [the first part](https://brendan.abolivier.bzh/exchange-pt-1/#what-is-exchange), EWS will soon be retired for domains hosted on Microsoft's SaaS platforms. This means we're racing against the clock to support the newer and shinier API for [Microsoft Graph](https://learn.microsoft.com/en-us/graph/overview) before this happens, so that Thunderbird users with Exchange mailboxes aren't forced back to Outlook.

Because it's the only direction with a time pressure, we have decided to move onto supporting the Graph API, this time under the leadership of the very capable and amazing [Eleanor Dicharry](https://hachyderm.io/@eleanxr) (I'm still involved in the project, and annoying people with my opinions, just not the primary person in charge anymore because everyone needs a break).

This might sound like we just finished implementing support for one protocol/API, only to jump to the starting point again and repeat the entire project with roughly similar specs. However, the effort we've put towards reusability in the first part of this project, as well as the knowledge we've built over the past couple of years, quickly started showing results once we started looking into Graph support.

As an example I [shared](https://social.abolivier.bzh/@brendan/116337181877820575) not too long ago on Mastodon, with EWS it took us about 11 months to get from the start of the project to a "full message sync" prototype, i.e. being able to sync (and display) an account's list of folders, a folder's list of messages, and a message's body. With Graph, it took us about 4 months to get to this stage, and at the time of writing these lines we seem to be mere months, if not weeks away from being able to issue a call for testing for our Graph implementation. This due, in part, to two main reasons:

- We moved a significant amount of the EWS client code to a [common internal crate](https://searchfox.org/comm-central/rev/75f4a571890aab694d6574e5c0a193cdb87a3e96/rust/protocol_shared/) so the Graph client can reuse it.
- We designed the Graph client so that it can reuse the client and callback interfaces designed for EWS, and due to the components on the C++ side being generic enough, we've been able to reuse them entirely with little changes. This means we could focus on implementing a protocol client and not on figuring out how to make this client fit within Thunderbird.

Note that this is hugely facilitated by Graph and EWS sharing similar concepts (since they work off the same back-end). We're hopeful we could reuse the same recipe (with maybe some minor changes) with other protocols (such as JMAP), but until we actually start working on them it's difficult to tell whether this is a realistic expectation or misplaced optimism.

# The end - for now

This is where the story ends for now! Like I mentioned we're still hard at work on bringing the same level of support for the Microsoft Graph API, and then expanding this support beyond email. All the ground work I've mentioned in this article represents a significant amount of work, but it's nice to think that all this infrastructure and practices we defined will help shape part of the future of a project I've held close to my heart for so long.

Anyway, thanks a lot for reading through this duology - I've enjoyed sharing all these details about this project, and I hope you felt it was a valuable read. As always, I'd also like to thank [Thibaut](https://toot.cat/@CromFr) for his help with proofreading both posts.

If you'd like to hear me talk more about some of the funky stuff we're doing in the Rust and/or Exchange area (and you understand French), I'll be [at BreizhCamp](https://www.breizhcamp.org/programme/session/1176957) in a couple of weeks, with a talk about another aspect of our Exchange protocol client (I might even remember to update this post with the recording once it's available). In any case, thanks again for passing by, and I'll see you next time!
