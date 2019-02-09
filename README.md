# Pitch
A lightweight, non intrusive, low dependency library to communicate with APNS over HTTP/2 built with Swift NIO.

# Motivations
APNS is used to push billions of pushes a day, (7 billion per day in 2012). Many of us using Swift on the backend are using it to power our iOS applications. Having a community supported APNS implementation would go a long way to making it the fastest, free-ist, and simplest solution that exists.

Also too many non standard approaches currently exist. Some use code that depends on Security (Doesn't work on linux) and I haven't found one that uses NIO with no other dependencies. Some just execute curl commands.

#### Existing solutions
- https://github.com/moritzsternemann/nio-apns
- https://github.com/kaunteya/APNSwift
- https://medium.com/@nathantannar4/supporting-push-notifications-with-vapor-3-3f6cc959c789
- https://github.com/PerfectlySoft/Perfect-Notifications
- https://github.com/hjuraev/VaporNotifications

Almost all require a ton of dependencies.

# Proposed Solution

Develop a Swift NIO HTTP2 solution with minimal dependencies.

A proof of concept implementation exists [here](https://github.com/kylebrowning/swift-nio-http2-apns)

### What it does do

- Provides an API for handling connection to Apples HTTP2 APNS server
- Provides proper error messages that APNS might respond with.
- Uses custom/non dependency implementations of JSON Web Token specific to APNS (using [rfc7519](https://tools.ietf.org/html/rfc7519)
- Imports OpenSSL for SHA256 and ES256
- Reads your `.p8` APNS Push key from local file.
- Signs your token request
- Sends push notifications to a specific device.
- [Adheres to guidelines Apple Provides.](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server/establishing_a_token-based_connection_to_apns)

### What it doesn't do YET
- Use an OpenSSL implementation that **is not** `CNIOOpenSSL`


### What it won't do.
- Store/register device tokens
- Build an HTTP2 generic client
- Google Cloud Message
- Refresh your token no more than once every 20 minutes and no less than once every 60 minutes. (up to connection handler)
- Provide multiple device tokens to send same push to.


### What it could do
- [Use the SSWG HTTP2 client](https://forums.swift.org/t/generic-http-client-server-library/18290/11)
- Be the APNS library for all of Server Side Swift projects!

### Usage
```swift 
let sslContext = try SSLContext(configuration: TLSConfiguration.forClient(applicationProtocols: ["h2"]))
let group = MultiThreadedEventLoopGroup(numberOfThreads: 1)
var verbose = true
let apnsConfig = APNSConfig.init(keyId: "9UC9ZLQ8YW", teamId: "ABBM6U9RM5", privateKeyPath: "/Users/kylebrowning/Downloads/key.p8", topic: "com.grasscove.Fern", env: .sandbox)
let apns = try APNSConnection.connect(apnsConfig: apnsConfig, sslContext: sslContext, on: group.next()).wait()
if verbose {
    print("* Connected to \(apnsConfig.getUrl().host!) (\(apns.channel.remoteAddress!)")
}
let alert = Alert(title: "Hey There", subtitle: "Subtitle", body: "Body")
let aps = Aps(alert: alert, category: nil, badge: 1)
let res = try apns.send(deviceToken: "223a86bdd22598fb3a76ce12eafd590c86592484539f9b8526d0e683ad10cf4f", APNSRequest(aps: aps, custom: nil)).wait()
print("APNS response: \(res)")
try apns.close().wait()
try group.syncShutdownGracefully()
exit(0)
```