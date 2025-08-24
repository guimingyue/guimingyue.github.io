---
layout: post
title:  Building a Visual Studio Code Extension from Scratch publishing the extension package
categories: AI Development
---

In the previous articles, we have implemented formatting/compacting and unescaping/escaping JSON text in a file. Today, I'm going to publish the extension to the [Visual Studio Code extension Marketplace](https://marketplace.visualstudio.com/vscode). Still using Qwen Code.

## Guide to publish to the Visual Studio Code extension Marketplace

The official guide to publish to the Visual Studio Code extension Marketplace is [here](https://code.visualstudio.com/api/working-with-extensions/publishing-extension). First we should build a published package of our extension. And I'm not gonna to execute the build command by myself. So I ask Qwen Code to build the extension package.

## Build the extension package

The prompt building the extension package is as follows:

```shell
> package this codebase to a VS Code extension package that can be published to VS Code extension market
```
After Qwen Code finish this job, It gave us a summary of what it did.

![Publis extension v0.0.1](/images/build_vscode_ext_qwen/publish_codex_v0.0.1.png)

So I uploaded the extension package to the VS Code extension marketplace and install it in Visual Studio Code. But when I tried to use the extension to format a json text, I found that it didn't work. And the error message was: `command 'code-jsonx.formatJson' not found`. So I asked Qwen Code to fix the bug.

```shell
> I published the package code-jsonx-0.0.1.vsix to the VS Code extension market place, and installed. But when I use the Format JSON command, an error occured: command 'code-jsonx.formatJson' not found
```
Qwen Code gave the summary on what it had done. Besides fixing the error, Qwen Code also build a new package. So I installed the new package and the error was fixed.

![Fix extension bug](/images/build_vscode_ext_qwen/published_v1.0.0_not_found_bugfix.png)

Publish the extension to the Visual Studio Code Marketplace is easy, So I published the extension to the Visual Studio Code Marketplace by myself as the instructions of the official help content.

## Conclusion

Qwen Code did a great job. Today, I build the extension package without any trobules. If I did't have Qwen Code, I would have spent a lot of time to build the package and fix the bug I encountered. I think it's a great tool to help developers build software, fix bugs and improve software quality.
