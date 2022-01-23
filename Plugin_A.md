# NOCOM_BOT A-Type Plugin Specification

Version: v0r1p0 (draft)<br>
Last updated: 23/01/2022

## 1. Overview

NOCOM_BOT is a powerful, flexible chatbot framework that allow bot developer to extends its functionality using modules and plugins. To make things easier for bot developers, this specification will define a new format that is more friendly than writing code that access the Core directly.

The A-Type plugin format is specifically designed for C3CBot v1.x, but other bot developers can install the handler with no restriction included.

## 2. Terms and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP
14 [RFC2119] [RFC8174] when, and only when, they appear in all
capitals, as shown here.

Commonly used terms in this specification are described below.

- Core: The main process which handles all communication from/to different modules.

- Module: A process that is spawned from Core or a thread inside Core, used for all functionality of NOCOM_BOT.

- Plugin handler: A module that is used to parse/execute A-type plugin securely.

- Plugin: Code/file that can be parsed/executed by plugin handler. In this case, it's specifically in A-type plugin format.

## 3. Plugin format

A-Type plugin MUST use ZIP format as the container for all the required files (not including files must be downloaded from Internet on-demand).

The structure of the ZIP file SHOULD be like the tree below.

```
plugin.zip (root)
├─ assets/
│  ├─ anything...
├─ entry.js (!)
├─ jsfunc1.js
├─ jsfunc2.js
├─ plugin.json (*)
├─ tsconfig.json (subclass 1 only)
├─ package.json
├─ pnpm-lock.json
```

Note: 
- If a file/folder is marked (*), then that file/folder MUST be present, and the format of the file marked is in this specification.
- If a file/folder is marked (!), then that file/folder MUST be present, but can be renamed (should be following the specification).
- If a file/folder is not marked, then that file/folder is optional, but recommended.

## 4. File content format

### 4.1. plugin.json

The content of `plugin.json` file SHOULD be following this format:

```ts
{
    formatVersion: 0,
    subclass: (0 | 1), // See note [1]
    entryPoint: string, // can be redefined, but SHOULD be entry.js / entry.ts
    author: string,
    pluginName: string,
    pluginNamespace: string,
    pluginVersion: string // MUST be SemVer-compatible
}
```

[1]: Subclass MUST be 0 if you are writing in JavaScript (A0), or MUST be 1 if you are writing in TypeScript (A1).

### 4.2. package.json + pnpm-lock.json

This file is optional, but recommended to have, especially when you are going to use extenal node modules from npm or GitHub.

Plugin resolver will automaticially install **all** dependencies found in these two files, and the packages installed can be used like usual (`import something from "package"`)

Note that the A-type format specifically uses pnpm, and we recommended that you also uses pnpm in every project, even projects not related to this, as pnpm can saves your disk space.

### 4.3. entry.js / entry.ts

Depending on the subclass, you MUST write either JavaScript or TypeScript code in ES module format on this file. CommonJS format is disallowed in this plugin format.

The `import` keyword can also be used to import code in other files relative to the current file root.

Top-level `await` is **enabled** and **supported**.

Before using, you MUST import the function module by inserting this command on the top of entry file (other files are not required to import this file, unless it also needs to use function from this file):

```ts
import * as NOCOM_AType from "@nocom_bot/nocom-atype-v0";
```

The `NOCOM_AType` variable will have these functions:

```ts
function verifyPlugin(allow: boolean): void
function callFuncPlugin(namespace: string, funcName: string, ...args: any): Promise<any>
function registerFuncPlugin(funcName: string, callback: Function): Promise<boolean>
function callAPI(moduleID: string, cmd: string, value: any): Promise<any>
function registerCommand(commandName: string, commandCallback: (data: {
    cmd: string,
    args: string[],
    attachments: {
        filename: string,
        url: string // http(s)/file protocol is possible.
    }[],
    additionalInterfaceData?: any
}) => Promise<{
    content: string,
    attachments: {
        filename?: string,
        data: Buffer | string // if string then point to url, http(s)/file protocol allowed
    }[],
    additionalInterfaceData?: any
}>): Promise<boolean>
function registerCommandFuncPlugin(commandName: string, funcName: string): Promise<boolean>
function exit(exit_code: number, exit_reason?: string): void
```

### 4.4. tsconfig.json (subclass 1 only)

See [https://www.typescriptlang.org/docs/handbook/tsconfig-json.html](this) for the format of this file.

(TBD)
