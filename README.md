# NOCOM_BOT Module Specification

Version: v1r0 (draft)<br>
Last updated: 11/09/2021

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

## 3. Specification
### 3.1. Module language and format

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
    moduleNamespace: string,
    moduleCapable: ("chat_interface", "plugins_resolver", "database")[], // What this module can do.
    scriptSrc?: string, // Only required if type = "script", describing the entry point location relative to ZIP file root.
    executable?: { // Only required if type = "executable"
        [platform: (
            "win32" | 
            "aix" | 
            "darwin" | 
            "freebsd" | 
            "linux" | 
            "openbsd" | 
            "sunos"
        )]: {
            [cpuArch: (
                "arm" |
                "arm64" |
                "ia32" |
                "mips" |
                "mipsel" |
                "ppc" |
                "ppc64" |
                "s390" |
                "s390x"
                "x32" | 
                "x64"
            )]: string // Path relative to ZIP file root.
        }
    }
    executableArgs?: string, // Only required if type = "executable"
    transplier?: string // Only required if type = "code-src", describing how to execute the source code.
}
```

Please note that, when Module is being executed, all content inside the ZIP file will be extracted to `<Core's current working directory>/.data/temp/runtime-${random hex}/${module namespace}/`, and the Module's current working directory (if type = "package" or "executable" or "code-src") will be set to that directory.

### 3.2. Core-Module protocol

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
    "module": "module type that Module should act",
    "id": "Module's instance ID"
}
```

In return, Module MUST return a message with data described below in 30 seconds or else Module will be terminated.

```json
{
    "type": "handshake_success",
    "module": "<module type that Module is acting>",
    "module_displayname": "<user-friendly name of the module (example: MongoDB database, Discord interface, ...)>"
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

