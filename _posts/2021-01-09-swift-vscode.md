---
layout: post
title:  "Swift on VS Code"
date:   2021-01-09
---

# Swift on VS Code

The Swift programming language can be setup with VS Code with most IDE features Xcode has like code completion, static analysis, indexing and debugging on Linux. 

Requirements:

- Swift 5.3.3
- Ubuntu Focal, Debian Buster
- x86_64, ARM64

1. Install Swift on your system and make sure it is part of your path. For ARM64 use [`swift-arm64` repository](https://github.com/futurejones/swift-arm64) and for x86_64 download from [Swift.org](https://swift.org).
2. Install VS Code from the [official repository](https://code.visualstudio.com/Download) or [`code-server`](https://github.com/cdr/code-server) to remotely run VS Code in the browser.
3. Install [CodeLLDB](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb) and [SourceKit-LSP](https://marketplace.visualstudio.com/items?itemName=pvasek.sourcekit-lsp--dev-unofficial) extensions. If running a VS Code fork like `code-server` then just download the extension and install the [VSIX file](https://code.visualstudio.com/docs/editor/extension-gallery#_install-from-a-vsix).
4. Create configurations in `.vscode` folder to build and debug the target. This can vary based on the package's executables and libraries. The following is an example for `Foo` package.

`.vscode/tasks.json`

```
{
    "version": "2.0.0",
    "tasks": [
        // compile your SPM project
        {
            "label": "swift-build",
            "type": "shell",
            "command": "swift build"
        },
        // compile your SPM tests
        {
            "label": "swift-build-tests",
            "type": "process",
            "command": "swift",
            "group": "build",
            "args": [
                "build",
                "--build-tests"
            ]
        }
    ]
}
```

`.vscode/launch.json`

```
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug",
            "program": "${workspaceFolder}/.build/debug/FooTool",
            "args": [],
            "cwd": "${workspaceFolder}",
            "preLaunchTask": "swift-build"
        },
        {
            "type": "lldb",
            "request": "launch",
            "name": "Test",
            "program": "./.build/debug/FooPackageTests.xctest",
            "preLaunchTask": "swift-build-tests"
        }
    ]
}
```

- Notes:
   * The Swift package needs to be manually built (`swift build`) to update indexing and completion features. This also applies to warning about missing dependencies.
   * To replicate Xcode's `Swift Error Breakpoint`, create a breakpoint for the symbol `swift_willThrow` in VS Code.

