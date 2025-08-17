---
layout: post
title:  Building a Visual Studio Code Extension from Scratch Bugfix Using Qwen Code
categories: AI Development
---

In the previous article, I've introduced how to build a Visual Studio Code extension from scratch using Qwen Code and Lingma. Today, I'm going to walk you through fixing a bug in the extension.

## The Bug
The bug is that when the selected json text is formatted or compacted, the whole text in the file will be formatted or compacted and the selected text will be replaced by the result text. So I think this is a bug easy to fix.

![Text Selected format and compact bug](/images/build_vscode_ext_qwen/codex_json_bug.gif)

## Fixing the Bug

Then, I use the prompt to ask Qwen Code to fix the bug.

```
> This is a vs code extension that formatting and compacting json text in a file. The extension user can format and compact the whole text or selected text. Now we have a bug that when formatting or compacting the selected json text, the whole text in the file will be formated or compacted and the selected text will be replaced by the result text. Now your work is to fix this bug and test the code you added or modified before you finish you work.
```
And Qwen Code did a great job. With that prompt, Qwen Code generated code to fix the bug. The summary Qwen Code generated is as follows:

![fix Text Selected bug](/images/build_vscode_ext_qwen/summary_of_qwen_code_fixing_codex_json_bug.png)

The picture below shows formatting and compacting the selected json text after fixing the bug.

![fix Text Selected bug](/images/build_vscode_ext_qwen/codex_json_fixbug.gif)

## Conclusion

With the help of code assistant tools like Qwen Code and Lingma, fixing bugs becomes easier and more efficient. In this article, we've seen how to use Qwen Code to fix a bug in a VS Code extension. Just with prompts and a few clicks, you can fix bugs quickly and efficiently.
