---
layout: post
title:  Building a Visual Studio Code Extension from Scratch adding a new feature of unescaping and escaping JSON text
categories: AI Development
---

In the previous articles, I've introduced how to build a Visual Studio Code extension from scratch using Qwen Code and Lingma and fix a bug in the extension. Today, I'm going to add a new feature of unescaping and escaping JSON text to the extension.

## A use case

In our programmer's work, we often receive a JSON text that is in escaped format. We need to unescape it to get the original JSON text and then format or compact it. So I want to add a new feature to the extension to unescape and escape JSON text. For example, the escaped JSON text `{\"a\": \"x\", \"b\": 1, \"c\": true}` cannot be formatted before we unescape it.

## Let's prompt.

First, I use the prompt to ask Qwen Code to add a new feature to the extension.

```shell
>  Now we want to extend the json format and compact functionality of this repository. Escaping JSON text is a common use case in a programmer's work, please add  VS code commands to escape and unescape JSON text. For example when escaping this JSON text {"a":"x","b":1,"c":true}, the output is {\"a\": \"x\", \"b\": 1, \"c\": true}, and the unescaping command do the opposite work.
```

With that prompt, Qwen Code generated code to add the functionality. The summary Qwen Code generated is as follows:

![Escaping and unescaping summary](/images/build_vscode_ext_qwen/escaping_unescaping_v1_result.png)

Now the extension can escape and unescape JSON text. But when I tried to unescape the formated escaped JSON text, it failed with an error.

```shell
Failed to unescape text: Bad control character in string literal in JSON at position 3 (line 1 column 4)
```

So, I asked Qwen Code to fix the bug.

```shell
> Well done. But there is another problem when unescaping a json text with control character, consider this string  {
      \"a\":\"x\",   
       \"c\":2, 
       \"d\":false                                                      
    }, the control characher will be the cause of fail when unescaping
```
Just with that prompt, Qwen Code generated code to fix the bug. And the summary Qwen Code generated is as follows:

![Fix the unescaping bug within control character](/images/build_vscode_ext_qwen/fix_unescaping_bug.png)

Now we can escape and unescape JSON text in Visual Studio Code.

![Unescape Format Compact escape](/images/build_vscode_ext_qwen/unescape_format_compact_escape.gif)

## Optimize Menu items

Now beside the four commands in the Command Palette, we also add menu items to the Right Click Popup Menu. So let's group the menu items into a menu and show these menu items in the expanded menu list.

```shell
> Today we add two command menu items into the right click menu, there will be more command to be added. So can we group by the command menu items into a CodeJsonX menu item and expand the command menu items from the CodeJsonX menu item?
```

And after that prompt, Qwen Code generated code to group the command menu items into a CodeJsonX menu item. The menu items now are as follows:

![Group by menu items](/images/build_vscode_ext_qwen/group_by_menu_items.png)

## Conclusion

With the help of code assistant tools like Qwen Code, we added a new feature to our extension. Qwen Code understands the prompt and generates code to implement unescaping and escaping JSON text. It is a great relief to use Qwen Code to build our software.

