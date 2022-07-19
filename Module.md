# NOCOM_BOT Module Specification

Version: v1r13p1<br>
Last updated: 20/07/2022

## 1. Overview

NOCOM_BOT is a powerful, flexible chatbot framework that allow bot developer to extends its functionality using modules and plugins. 

Because of its flexible nature, there should be a specification to allow execution and communication with different modules. This specification aim to give modules a way to be executed and communicate using pre-defined format.

## 2. Terms and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP
14 [RFC2119] [RFC8174] when, and only when, they appear in all
capitals, as shown here.

Commonly used terms in this specification are described below.

- Core: The main process which handles all communication from/to different modules.

- Module: A process that is spawned from Core or a thread inside Core, used for all functionality of NOCOM_BOT.

## 3. Module protocol and format

### 3.1. Module programming language and format

To ensure the flexiblity of NOCOM_BOT, any programming language can be used to make Module.

If Module is a npm-compatible (Node.js) package (with `package.json`), Module will be spawned as a new Node.js-compatible process with entry point described in `main` in `package.json`.

If Module is a single Node.js script, then Module will be spawned as a thread in Core (Node.js worker).

If Module is source code in other languages and needs a transpiler (for example, Python), you MUST install that languages before you can use the Module. Additionally, module description MUST contains how to run the source code.

Module MUST be packed in ZIP with `module.json` describing the type of Module, and how it communicates, as below (Note: It's in TypeScript to write comments and possible value, but you MUST write JSON in the ZIP file).

```ts
{
    type: (
        "package" | // If Module is a npm-compatible package
        "script" | // If Module is a Node.js script
        "executable" | // If Module is an executable (usually compiled from other languages than JS)
        "code-src" // If Module is source code writen in other languages
    ),
    namespace: string,
    communicationProtocol: (
        "node_ipc" | // process.send, process.on("message", ...)
        "node_worker" | // require("worker_threads")
        "msgpack" // See 3.2
    ),
    scriptSrc?: string, // Only required if type = "script", describing the entry point location relative to ZIP file root.
    executable?: { // Only required if type = "executable"
        [platform: typeof NodeJS.os.platform() /* [1] */]: {
            [cpuArch: typeof NodeJS.os.arch() /* [2] */]: string // Path relative to ZIP file root.
        }
    }
    executableArgs?: string, // Only required if type = "executable"
    transplier?: string, // Only required if type = "code-src", describing how  to execute the source code.
    autoRestart: boolean
}
```

[1]: https://nodejs.org/dist/latest-v17.x/docs/api/os.html#osplatform
[2]: https://nodejs.org/dist/latest-v17.x/docs/api/os.html#osarch

Please note that, when Module is being executed, all content inside the ZIP file will be extracted to `<profile directory>/temp/${random hex}/${module namespace}/`, and the Module's current working directory (if type is "package", "executable" or "code-src") will be set to that directory.

### 3.2. Core-Module communication protocol

If Module is a Node.js-compatible process and support `process.send` and `process.on("message", ...)` then Module SHOULD use that to communicate with Core.

If Module is a thread inside Core (Node.js worker), it MUST get `parentPort` from `require('worker_threads')` and use `parentPort.on("message", ...)`, `parentPort.postMessage()` to communicate with Core.

Alternatively, Module (that isn't a thread inside Core) can read STDIN and write to STDOUT in a way that's described below to communicate with Core (it's in hexadecimal, but you MUST use raw bytes when communicating).

```
<Message header (string AAAA): 41 41 41 41> <Length in 4-byte integer, big-endian (not string!)> <MessagePack binary serialization of the object received/sent>
```

For example, this is a message containing the object `{foo:"bar"}`.

```
41 41 41 41 00 00 00 09 81 A3 66 6F 6F A3 62 61 72
```

### 3.3. Handshaking

To determine what the module should do, a handshake is required. Module MUST NOT do or send anything before receiving a handshake from Core.

Handshake MUST be done only once, and MUST start from Core.

The Core will send a message to Module with this data to initialize handshake:

```json
{
    "type": "handshake",
    "id": "Module's instance ID",
    "protocol_version": "1",
    "config": {} // user-defined config
}
```

In return, Module MUST return a message with data described below in 30 seconds or else Module will be terminated.

```json
{
    "type": "handshake_success",
    "module": "<module type that Module is acting>",
    "module_displayname": "<user-friendly name of the module (example: MongoDB database, Discord interface, ...)>",
    "module_namespace": "<MUST match with the JSON>"
}
```

If Module cannot act as the requested type, 
Module SHOULD return a message describing error and request to be terminated.

```json
{
    "type": "handshake_fail",
    "error": "<... (for example: Unknown module type)>"
}
```

### 3.4. Keep-alive (Ping)

To prevent the process from hanging, an alive-or-dead test (challenge) will be issued from Core at random to determine whether if Module is hanging or not, and Module will be killed if hanging.

Module MUST response to a challenge in 30 seconds to keep running.

The Core will send a message described below to check if Module is alive.

```json
{
    "type": "challenge",
    "challenge": "<random string>"
}
```

If Module received a challenge, it MUST response back a message described below.

```json
{
    "type": "challenge_response",
    "challenge": "<random string from Core>"
}
```

### 3.5. API call/response

> Note: Every API call is async, you should not block thread to wait for a call to return.

Module can call API commands of other modules. When Module want to call an API command, Module MUST send a message:

```json
{
    "type": "api_send",
    "call_to": "<target Module ID>",
    "call_cmd": "<target API command from target module>",
    "data": "...",
    "nonce": "<rolling number for each API call or random number>"
}
```

The target Module will receive a message indicating an API call from other modules:

```json
{
    "type": "api_call",
    "call_from": "<source Module ID>",
    "call_cmd": "...",
    "data": "...",
    "nonce": "..."
}
```

There are 3 possible responses for the target Module:

1. The API command don't exist and cannot be called
```json
{
    "type": "api_sendresponse",
    "response_to": "<source Module ID>",
    "exist": false,
    "nonce": "<exact nonce from request>"
}
```

2. The API command does exist, but the execution of it failed
```json
{
    "type": "api_sendresponse",
    "response_to": "<source Module ID>",
    "exist": true,
    "error": "...",
    "data": null,
    "nonce": "<exact nonce from request>"
}
```
> Note: Error MUST NOT be an Error object from JavaScript or the native implementation. It SHOULD be a string, or number, or even null (if there isn't an error message).

3. The API command does exist, and executed successfully
```json
{
    "type": "api_sendresponse",
    "response_to": "<source Module ID>",
    "exist": true,
    "error": null,
    "data": "...",
    "nonce": "<exact nonce from request>"
}
```

When the target Module has responded, source Module will receive an API response:

```json
{
    "type": "api_response",
    "response_from": "<target Module ID>",
    "exist": "...",
    "data": "...", // This will be null if there's an error.
    "error": "...", // This will be null if error doesn't occur.
    "nonce": "<exact nonce from request>"
}
```

## 4. Core API call

> Note: The Core's API will be available at module ID `core` and namespace `core` to not complicate things up.

### 4.1. Get registered modules (`get_registered_modules`)

Data: none

Return: 
```ts
{
    moduleID: string,
    namespace: string,
    displayname: string,
    running: boolean
}[]
```

### 4.2. Kill this module (`kill`)

Data: none

Return: `null`

### 4.3. Shutdown the Core (`shutdown_core`)

Data: none

Return: `null`

### 4.4. Restart the Core (`restart_core`)

Data: none

Return: `null`

### 4.5. Register event hook (`register_event_hook`)

Data: 
```ts
{
    callbackFunction: string,
    eventName: string
}
```

Return:
```ts
{
    success: boolean
}
```

For more information, see 6.

### 4.6. Unregister event hook (`unregister_event_hook`)

Data:
```ts
{
    callbackFunction: string,
    eventName: string
}
```

Return:
```ts
{
    success: boolean
}
```

For more information, see 6.

### 4.7. Send event (`send_event`)

Data:
```ts
{
    eventName: string,
    data: any
}
```

Return:
```ts
{
    hasSubscribers: boolean
}
```

### 4.8. Register plugin (`register_plugin`)

> Note: This API is intended for plugin handlers (or self-hosted plugin-like Module) only.

Data:
```ts
{
    pluginName: string,
    namespace: string,
    version: string,
    author: string
}
```

Return:
```ts
{
    conflict: boolean
}
```

> Note: If namespace is conflicted, it will not be registered, and other plugins will not be able to call function on that plugin.

### 4.9. Unregister plugin (`unregister_plugin`)

> Note: This API is intended for plugin handlers (or self-hosted plugin-like Module) only.

Data:
```ts
{
    namespace: string
}
```

Return: 
```ts
{
    success: boolean
}
```

### 4.10. Ask for operator's input (`prompt`)

Data:
```ts
{
    promptInfo: string,
    promptType: "string" | "yes-no",
    defaultValue?: string | boolean
}
```

Return:
```ts
{
    data: string | boolean
}
```

### 4.11. Logging (`log`)

Data:
```ts
{
    level: "verbose" | "critical" | "error" | "warn" | "info" | "debug",
    namespace: string,
    data: any[]
}
```

Return: `null`

### 4.12. Wait for module (`wait_for_module`)

Data:
```ts
{
    moduleNamespace: string,
    timeout?: number //ms
}
```

Return (on loaded): `true`

Return (timed out): `false`

> Note: Timeout option is optional, if unspecified then it will be Infinity (no timeout)

### 4.13. Get data folder location (`get_data_folder`)

Data: none

Return: string with data folder location

### 4.14. Get temp folder location (`get_temp_folder`)

Data: none

Return: string with temp folder location

### 4.15. Install NPM/PNPM dependencies (`pnpm_install`)

Data:
```ts
{
    path: string
}
```

Return:
```ts
{
    success: boolean,
    error?: string
}
```

### 4.16. Install specific NPM/PNPM dependency (`pnpm_install_specific`)

Data:
```ts
{
    path: string,
    dep: string
}
```

Return:
```ts
{
    success: boolean,
    error?: string
}
```

## 5. Application-specific API call

> Note: If you are creating an interface that is using the module types defined below, you MUST implement all API call to maintain compatibility. Additional API commands MAY be defined if Module needs that.

### 5.1. Interface handler (module type = "interface")

> Note: Only one instance per interface will be created. The Core will expect the interface handler to handle multiple accounts.

#### **5.1.1. Login (`login`)**

Data:
```ts
{
    interfaceID: number,
    loginData: any // module defined
}
```

Return: 
```ts
{
    success: boolean,
    interfaceID: number,
    accountName: string,
    rawAccountID: string,
    formattedAccountID: string,
    accountAdditionalData: any
}
```

#### **5.1.2. Logout (`logout`)**

Data:
```ts
{
    interfaceID: number
}
```

Return: `null`

#### **5.1.3. Get User info (`get_userinfo`)**

Data:
```ts
{
    userID: string
}
```

Return:
```ts
{
    userName: string,
    ... // additional data is module-defined.
}
```

#### **5.1.4. Get Thread/Channel info (`get_channelinfo`)**

Data:
```ts
{
    channelID: string
}
```

Return:
```ts
{
    channelName: string,
    ... // additional data is module-defined.
}
```

#### **5.1.5. Send message (`send_message`)**

Data:
```ts
{
    content: string,
    attachments: string[], // URL (file:// is allowed)
    threadID: string,
    replyMessageID?: string,
    ... // additional data is module-defined.
}
```

Return:
```ts
{
    success: boolean,
    messageID: string,
    ... // additional data is module-defined.
}
```

### 5.2. Database (module type = "database")

> Note: Only one instance per database type will be created. The Core will expect the database module to handle multiple connections.

#### **5.2.1. Get default config (`default_cfg`)**

Data: none

Return: default parameter for connecting to database

#### **5.2.2. List connected database (`list_db`)**

Data: none

Return: 
```ts
{
    databaseID: number,
    databaseName: string
}[]
```

#### **5.2.3. Connect database (`connect_db`)**

Data: 
```ts
{
    databaseID: number,
    params: any // module-defined
}
```

Return:
```ts
{
    success: boolean,
    databaseID: number
}
```

#### **5.2.4. Get data (`get_data`)**

Data:
```ts
{
    databaseID: number,
    table: string,
    key: string
}
```

Return: 
```ts
{
    success: boolean,
    data: any
}
```

#### **5.2.5. Set data (`set_data`)**

Data:
```ts
{
    databaseID: number,
    table: string,
    key: string,
    value: any
}
```

Return: 
```ts
{
    success: boolean
}
```

#### **5.2.6. Delete data (`delete_data`)**

Data: 
```ts
{
    databaseID: number,
    table: string,
    key: string
}
```

Return:
```ts
{
    success: boolean
}
```

#### **5.2.7. Delete table (`delete_table`)**

Data:
```ts
{
    databaseID: number,
    table: string
}
```

Return:
```ts
{
    success: boolean
}
```

#### **5.2.8. Disconnect database (`disconnect_db`)**

Data: 
```ts
{
    databaseID: number
}
```

Return: `null`

### 5.3. Plugins handler (module type = "pl_handler")

> Note: Not to be confused with Module! Plugin is used to add features in more secure, easier way. Plugin is handled by plugin handler, not Core.

#### **5.3.1. Check plugin (`check_plugin`)**

Data:
```ts
{
    filename?: string,
    pathname?: string
}
```

Return: 
```ts
{
    compatible: boolean,
    pluginName?: string,
    namespace?: string,
    version?: string,
    author?: string
}
```

#### **5.3.2. Load plugin (`load_plugin`)**

Data:
```ts
{
    filename?: string,
    pathname?: string
}
```

Return: 
```ts
{
    loaded: boolean,
    error?: string,
    pluginName?: string,
    namespace?: string,
    version?: string,
    author?: string
}
```

> Note: Plugin handler MUST register plugin (4.8) even when Core call this API.

#### **5.3.3. Unload plugin (`unload_plugin`)**

Data:
```ts
{
    namespace: string
}
```

Return: 
```ts
{
    error?: string
}
```

#### **5.3.4. Call function inside plugin (`plugin_call`)**

Data:
```ts
{
    namespace: string,
    funcName: string,
    args: any[]
}
```

Return: 
```ts
{
    error?: string,
    returnData: any
}
```

### 5.4. Command handler (module type = "cmd_handler")

> Note: Namespaced commands call will automaticially redirect to correct plugin (as Command handler already knows what module handle that namespace). 

#### **5.4.1. Register command (`register_cmd`)**

Data:
```ts
{
    namespace: string,
    command: string,
    funcName: string,
    funcDescAPI: string
}
```

Return:
```ts
{
    success: boolean,
    error?: string
}
```

#### **5.4.2. Unregister command (`unregister_cmd`)**

Data:
```ts
{
    namespace: string,
    command: string
}
```

Return:
```ts
{
    success: boolean,
    error?: string
}
```

#### **5.4.2. Get registered command list (`cmd_list`)**

Data: none

Return:
```ts
{
    commands: {
        namespace: string,
        command: string,
        funcName: string,
        funcDescAPI: string
    }[],
    count: number
}
```

## 6. Events

Sometimes you need to broadcast to a lot of modules interested in a topic without knowing which modules subscribed. This is where Events come in.

Events is part of the Core module. To register/unregister an event, use API call 4.5 and 4.6. Use API call 4.7 to send events.

The callback API will be called from Core with the following data when a module send an event:
```ts
{
    calledFrom: string, // module ID
    eventName: string,
    eventData: any
}
```

### 6.1. Application-specific events

#### **6.1.1. Message from bot users (`interface_message`)**

Data:
```ts
{
    cmd: string,
    args: string[],
    attachments: {
        filename: string,
        url: string // http(s)/file protocol is possible.
    }[],
    interfaceHandlerName: string,
    interfaceID: number,
    additionalInterfaceData?: any
}
```

