---
title: WebSockets support in ASP.NET Core
author: wadepickett
description: Learn how to get started with WebSockets in ASP.NET Core.
monikerRange: '>= aspnetcore-3.1'
ms.author: wpickett
ms.custom: mvc
ms.date: 07/09/2025
uid: fundamentals/websockets
---
# WebSockets support in ASP.NET Core

[!INCLUDE[](~/includes/not-latest-version.md)]

:::moniker range=">= aspnetcore-9.0"

This article explains how to get started with WebSockets in ASP.NET Core. [WebSocket](https://wikipedia.org/wiki/WebSocket) ([RFC 6455](https://tools.ietf.org/html/rfc6455)) is a protocol that enables two-way persistent communication channels over TCP connections. It's used in apps that benefit from fast, real-time communication, such as chat, dashboard, and game apps.

[View or download sample code](https://github.com/dotnet/AspNetCore.Docs/tree/main/aspnetcore/fundamentals/websockets/samples) ([how to download](xref:index#how-to-download-a-sample), [how to run](#sample-app)).

## Http/2 WebSockets support

Using WebSockets over HTTP/2 takes advantage of new features such as:

* Header compression.
* Multiplexing, which reduces the time and resources needed when making multiple requests to the server.

These supported features are available in Kestrel on all HTTP/2 enabled platforms. The version negotiation is automatic in browsers and Kestrel, so no new APIs are needed.

.NET 7 introduced WebSockets over HTTP/2 support for Kestrel, the SignalR JavaScript client, and SignalR with Blazor WebAssembly.

> [!NOTE]
> HTTP/2 WebSockets use CONNECT requests rather than GET, so your own routes and controllers may need updating.
> For more information, see [Add HTTP/2 WebSockets support for existing controllers](#add-http2-websockets-support-for-existing-controllers) in this article.
>
> Chrome and Edge have HTTP/2 WebSockets enabled by default, and you can enable it in FireFox on the `about:config` page with the `network.http.spdy.websockets` flag.

WebSockets were originally designed for HTTP/1.1 but have since been adapted to work over HTTP/2. ([RFC 8441](https://www.rfc-editor.org/rfc/rfc8441))

## SignalR

[ASP.NET Core SignalR](xref:signalr/introduction) is a library that simplifies adding real-time web functionality to apps. It uses WebSockets whenever possible.

For most applications, we recommend SignalR rather than raw WebSockets. SignalR:

* Provides transport fallback for environments where WebSockets isn't available.
* Provides a basic remote procedure call app model.
* Has no significant performance disadvantage compared to using raw WebSockets in most scenarios.

WebSockets over HTTP/2 is supported for:

* ASP.NET Core SignalR JavaScript client
* ASP.NET Core SignalR with Blazor WebAssembly

For some apps, [gRPC on .NET](xref:grpc/index) provides an alternative to WebSockets.

## Prerequisites

* Any OS that supports ASP.NET Core:  
  * Windows 7 / Windows Server 2008 or later
  * Linux
  * macOS  
* If the app runs on Windows with IIS:
  * Windows 8 / Windows Server 2012 or later
  * IIS 8 / IIS 8 Express
  * WebSockets must be enabled. See the [IIS/IIS Express support](#iisiis-express-support) section.  
* If the app runs on [HTTP.sys](xref:fundamentals/servers/httpsys):
  * Windows 8 / Windows Server 2012 or later
* For supported browsers, see [Can I use](https://caniuse.com/?search=websockets).

## Configure the middleware

Add the WebSockets middleware in `Program.cs`:

:::code language="csharp" source="~/fundamentals/websockets/samples/8.x/WebSocketsSample/Snippets/Program.cs" id="snippet_UseWebSockets":::

The following settings can be configured:

* <xref:Microsoft.AspNetCore.Builder.WebSocketOptions.KeepAliveInterval%2A> - How frequently to send "ping" frames to the client to ensure proxies keep the connection open. The default is two minutes.
* <xref:Microsoft.AspNetCore.Builder.WebSocketOptions.AllowedOrigins%2A> - A list of allowed Origin header values for WebSocket requests. By default, all origins are allowed. For more information, see [WebSocket origin restriction](#websocket-origin-restriction) in this article.

:::code language="csharp" source="~/fundamentals/websockets/samples/8.x/WebSocketsSample/Program.cs" id="snippet_UseWebSockets":::

## Accept WebSocket requests

Somewhere later in the request life cycle (later in `Program.cs` or in an action method, for example) check if it's a WebSocket request and accept the WebSocket request.

The following example is from later in `Program.cs`:

:::code language="csharp" source="~/fundamentals/websockets/samples/8.x/WebSocketsSample/Snippets/Program.cs" id="snippet_AcceptWebSocketAsync" highlight="7":::

A WebSocket request could come in on any URL, but this sample code only accepts requests for `/ws`.

A similar approach can be taken in a controller method:

:::code language="csharp" source="~/fundamentals/websockets/samples/8.x/WebSocketsSample/Controllers/WebSocketController.cs" id="snippet_Controller_Connect":::

When using a WebSocket, you **must** keep the middleware pipeline running for the duration of the connection. If you attempt to send or receive a WebSocket message after the middleware pipeline ends, you may get an exception like the following:

```
System.Net.WebSockets.WebSocketException (0x80004005): The remote party closed the WebSocket connection without completing the close handshake. ---> System.ObjectDisposedException: Cannot write to the response body, the response has completed.
Object name: 'HttpResponseStream'.
```

If you're using a background service to write data to a WebSocket, make sure you keep the middleware pipeline running. Do this by using a <xref:System.Threading.Tasks.TaskCompletionSource%601>. Pass the `TaskCompletionSource` to your background service and have it call <xref:System.Threading.Tasks.TaskCompletionSource%601.TrySetResult%2A> when you finish with the WebSocket. Then `await` the <xref:System.Threading.Tasks.TaskCompletionSource%601.Task> property during the request, as shown in the following example:

:::code language="csharp" source="~/fundamentals/websockets/samples/8.x/WebSocketsSample/Snippets/Program.cs" id="snippet_AcceptWebSocketAsyncBackgroundSocketProcessor":::

The WebSocket closed exception can also happen when returning too soon from an action method. When accepting a socket in an action method, wait for the code that uses the socket to complete before returning from the action method.

Never use `Task.Wait`, `Task.Result`, or similar blocking calls to wait for the socket to complete, as that can cause serious threading issues. Always use `await`.

## Add HTTP/2 WebSockets support for existing controllers

.NET 7 introduced WebSockets over HTTP/2 support for Kestrel, the SignalR JavaScript client, and SignalR with Blazor WebAssembly. HTTP/2 WebSockets use CONNECT requests rather than GET. If you previously used `[HttpGet("/path")]` on your controller action method for Websocket requests, update it to use `[Route("/path")]` instead.

[!code-csharp[](~/fundamentals/websockets/samples/8.x/WebSocketsSample/Controllers/WebSocketController.cs?name=snippet_Controller_Connect&highlight=3)]

### Compression

> [!WARNING]
> Enabling compression over encrypted connections can make an app subject to CRIME/BREACH attacks.
> If sending sensitive information, avoid enabling compression or use `WebSocketMessageFlags.DisableCompression` when calling `WebSocket.SendAsync`.
> This applies to both sides of the WebSocket. Note that the WebSockets API in the browser doesn't have configuration for disabling compression per send.

If compression of messages over WebSockets is desired, then the accept code must specify that it allows compression as follows:

```csharp
using (var webSocket = await context.WebSockets.AcceptWebSocketAsync(
    new WebSocketAcceptContext { DangerousEnableCompression = true }))
{

}
```

`WebSocketAcceptContext.ServerMaxWindowBits` and `WebSocketAcceptContext.DisableServerContextTakeover` are advanced options that control how the compression works.

Compression is negotiated between the client and server when first establishing a connection. You can read more about the negotiation in the [Compression Extensions for WebSocket RFC](https://datatracker.ietf.org/doc/html/rfc7692#section-7).

> [!NOTE]
> If the compression negotiation isn't accepted by either the server or client, the connection is still established. However, the connection doesn't use compression when sending and receiving messages.

## Send and receive messages

The `AcceptWebSocketAsync` method upgrades the TCP connection to a WebSocket connection and provides a <xref:System.Net.WebSockets.WebSocket> object. Use the `WebSocket` object to send and receive messages.

The code shown earlier that accepts the WebSocket request passes the `WebSocket` object to an `Echo` method. The code receives a message and immediately sends back the same message. Messages are sent and received in a loop until the client closes the connection:

:::code language="csharp" source="~/fundamentals/websockets/samples/8.x/WebSocketsSample/Snippets/Program.cs" id="snippet_Echo":::

When accepting the WebSocket connection before beginning the loop, the middleware pipeline ends. Upon closing the socket, the pipeline unwinds. That is, the request stops moving forward in the pipeline when the WebSocket is accepted. When the loop is finished and the socket is closed, the request proceeds back up the pipeline.

## Handle client disconnects

The server isn't automatically informed when the client disconnects due to loss of connectivity. The server receives a disconnect message only if the client sends it, which can't be done if the internet connection is lost. If you want to take some action when that happens, set a timeout after nothing is received from the client within a certain time window.

If the client isn't always sending messages and you don't want to time out just because the connection goes idle, have the client use a timer to send a ping message every X seconds. On the server, if a message hasn't arrived within 2\*X seconds after the previous one, terminate the connection and report that the client disconnected. Wait for twice the expected time interval to leave extra time for network delays that might hold up the ping message.

## WebSocket origin restriction

The protections provided by CORS don't apply to WebSockets. Browsers do **not**:

* Perform CORS pre-flight requests.
* Respect the restrictions specified in `Access-Control` headers when making WebSocket requests.

However, browsers do send the `Origin` header when issuing WebSocket requests. Applications should be configured to validate these headers to ensure that only WebSockets coming from the expected origins are allowed.

If you're hosting your server on "https://server.com" and hosting your client on "https://client.com", add "https://client.com" to the <xref:Microsoft.AspNetCore.Builder.WebSocketOptions.AllowedOrigins%2A> list for WebSockets to verify.

:::code language="csharp" source="~/fundamentals/websockets/samples/8.x/WebSocketsSample/Snippets/Program.cs" id="snippet_UseWebSocketsOptionsAllowedOrigins" highlight="6-7":::

> [!NOTE]
> The `Origin` header is controlled by the client and, like the `Referer` header, can be faked. Do **not** use these headers as an authentication mechanism.

### IIS/IIS Express support

Windows Server 2012 or later and Windows 8 or later with IIS/IIS Express 8 or later has support for the WebSocket protocol, but not for WebSockets over HTTP/2.

> [!NOTE]
> WebSockets are always enabled when using IIS Express.

### Enabling WebSockets on IIS

To enable support for the WebSocket protocol on Windows Server 2012 or later:

> [!NOTE]
> These steps are not required when using IIS Express

1. Use the **Add Roles and Features** wizard from the **Manage** menu or the link in **Server Manager**.
1. Select **Role-based or Feature-based Installation**. Select **Next**.
1. Select the appropriate server (the local server is selected by default). Select **Next**.
1. Expand **Web Server (IIS)** in the **Roles** tree, expand **Web Server**, and then expand **Application Development**.
1. Select **WebSocket Protocol**. Select **Next**.
1. If additional features aren't needed, select **Next**.
1. Select **Install**.
1. When the installation completes, select **Close** to exit the wizard.

To enable support for the WebSocket protocol on Windows 8 or later:

> [!NOTE]
> These steps are not required when using IIS Express

1. Navigate to **Control Panel** > **Programs** > **Programs and Features** > **Turn Windows features on or off** (left side of the screen).
1. Open the following nodes: **Internet Information Services** > **World Wide Web Services** > **Application Development Features**.
1. Select the **WebSocket Protocol** feature. Select **OK**.

### Disable WebSocket when using socket.io on Node.js

If using the WebSocket support in [socket.io](https://socket.io/) on [Node.js](https://nodejs.org/), disable the default IIS WebSocket module using the `webSocket` element in *web.config* or *applicationHost.config*. If this step isn't performed, the IIS WebSocket module attempts to handle the WebSocket communication rather than Node.js and the app.

```xml
<system.webServer>
  <webSocket enabled="false" />
</system.webServer>
```

## Sample app

The [sample app](https://github.com/dotnet/AspNetCore.Docs/tree/main/aspnetcore/fundamentals/websockets/samples) that accompanies this article is an echo app. It has a webpage that makes WebSocket connections, and the server resends any messages it receives back to the client. The sample app supports WebSockets over HTTP/2 when using a targeted framework of .NET 7 or later.

Run the app:

* To run app in Visual Studio: Open the sample project in Visual Studio, and press Ctrl+F5 to run without the debugger.
* To run the app in a command shell: Run the command [`dotnet run`](/dotnet/core/tools/dotnet-run) and navigate in a browser to `http://localhost:<port>`.

The webpage shows the connection status:

:::image source="~/fundamentals/websockets/_static/start.png" alt-text="Initial state of webpage before WebSockets connection":::

Select **Connect** to send a WebSocket request to the URL shown. Enter a test message and select **Send**. When done, select **Close Socket**. The **Communication Log** section reports each open, send, and close action as it happens.

:::image source="~/fundamentals/websockets/_static/end.png" alt-text="Final state of webpage after WebSockets connection and test messages are sent and received":::

:::moniker-end

[!INCLUDE[](~/fundamentals/websockets/includes/websockets8.md)]
[!INCLUDE[](~/fundamentals/websockets/includes/websockets7.md)]
[!INCLUDE[](~/fundamentals/websockets/includes/websockets6.md)]
[!INCLUDE[](~/fundamentals/websockets/includes/websockets3-5.md)]
