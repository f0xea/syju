---
title: Minecraft Brigadier API详解————以Fabric为例
publishDate: 2025-12-07
updatedDate: 2025-12-10
description: '带你了解Brigadier API'
tags: 
    - game
    - minecraft
    - fabric
    - brigadier
    - command
    - development
math: true
---
## Brigadier API
Brigadier 是一个由 Mojang 为 Minecraft: Java Edition 开发的开源指令解析和调度工具，你可以在 [GitHub](https://github.com/Mojang/brigadier) 上找到这个项目的源代码。

Fabric, Forge, Paper 等模组加载器或服务器核心提供了对 Brigadier API 的支持，使开发者能够轻松地创建和管理自定义指令，包括注册新的指令、添加参数、实现自动补全等功能。

本篇文章将以 Fabric 为例，介绍如何使用 Brigadier API 来创建和管理 Minecraft 指令。

## Commands
`Command` 是一个函数式接口，它接受一个泛型参数 `CommandContext<S>` ，并返回一个表示指令执行结果的整数值，并且可能会抛出 `CommandSyntaxException ` 异常。我们通常使用 Lambda 表达式或者方法引用作为 `Command` 的实现。

```java
Command<ServerCommandSource> command = context -> {
    // 指令的具体实现逻辑
    // 当指令成功执行的时候，一般会返回 1 
    return 0;
};
```

## 注册指令
在了解了 `Command` 接口之后，我们还需要了解注册指令的方式。在 Fabric 中，我们使用 ModInitializer#onInitialize 方法来执行注册指令的逻辑，我们可以将其写在模组主类中，但是我更建议你将其写在一个单独的类中。

```java
public class ModCommand implements ModInitializer {
    // ...
    @Override
    public void onInitialize() {
        CommandRegistrationCallback.EVENT.register((dispatcher, registryAccess, environment) -> {
            dispatcher.register(
                    Commands.literal("testcommand").executes(command)
            );
            // 可以在这里注册其它的指令
        });
    }
}
```

这里 `executes()` 传入的参数 `command` 是我们上一节创建的 `Command` 对象哦。通常，我们直接传入一个 Lambda 表达式或者一个方法引用。

我们重新实现一下 `command`。

```java
Command<ServerCommandSource> command = context -> {
    // 指令的具体实现逻辑
    context.getSource().sendSuccess(() -> Component.literal("Called /testcommand"), false);
    return 1;
};
```

然后不要忘记在资源文件下的 `fabric.mod.json` 中配置模组的入口点。

> 所有实现了 ModInitializer 接口的类都应该写入这个 JSON 文件中。
{: .prompt-info }

```json
"entrypoints": {
    // ...
    "main": [
        "org.f14a.fabricdemo.Fabricdemo",
        "org.f14a.fabricdemo.command.ModCommand"
    ]
}, 
```

现在运行游戏，当 `/testcommand` 被调用时，`command` 将会向执行指令的玩家（或服务器控制台）发送一条消息。请注意，`sendSuccess()` 的第一个参数是一个 `Supplier<Component>`， 需要提供一个 **Lambda** 表达式，第二参数的含义是是否向所有 op 发送指令的执行结果，应该根据指令的具体情况来决定。

现在让我们把指令实现写的稍微复杂一点。

```java
public class ModCommand implements ModInitializer {
    private static int executeDedicatedServer(CommandContext<CommandSourceStack> context) {
        context.getSource().sendSuccess(() -> Component.literal("You are running a dedicated server. "), false);
        return 1;
    }
    private static int executeClientServer(CommandContext<CommandSourceStack> context) {
        context.getSource().sendSuccess(() -> Component.literal("You are running a client integrated server. "), false);
        return 1;
    }
    // ...
    @Override
    public void onInitialize() {
        CommandRegistrationCallback.EVENT.register((dispatcher, registryAccess, environment) -> {
            dispatcher.register(
                    // Differentiate between dedicated server and client integrated server.
                    if(environment.includeDedicated) {
                        dispatcher.register(Commands.literal("whoami").executes(ModCommand::executeDedicatedServer));
                    }
                    else {
                        dispatcher.register(Commands.literal("whoami").executes(ModCommand::executeClientServer));
                    }
            )
        });
    }
}
```

这是 [Fabric 文档](https://docs.fabricmc.net/develop/commands/basics#registration-environment) 中对参数 `environment` 的演示代码（我稍微改过了一下），`includeDedicated` 和 `includeIntegrated` 分别在游戏运行在专用服务器（也就是单独启动的游戏服务端）和集成在客户端中的服务端时值为 `true` ，然后就没有别的用途了。

这里我们使用方法引用传入 `executes()`。

在实际使用中，我们有时会设计一些只有 op 才能使用的指令，这类指令对游戏的影响通常比较大。我们可以使用 `requires()` 进行权限检测。

```java
public class ModCommand implements ModInitializer {
    // ...
    @Override
    public void onInitialize() {
        CommandRegistrationCallback.EVENT.register((dispatcher, registryAccess, environment) -> {
            // Display the words if the user has specific permission.
            dispatcher.register(Commands.literal("amianop").requires(source -> source.hasPermission(1)).executes(context -> {
                context.getSource().sendSuccess(() -> Component.literal("You are an OP"), false);
                return 1;
            }));
        });
    }
}
```

`requires()` 接受一个 `Predicate<S>`，当且仅当 `Predicate<S>` 在某种情况下返回 `true` 时，指令才能被执行，并且会出现在自动补全的列表中。在这里我们使用 `hasPermission()` 来判断执行者是否有对应的权限。

当然，你可以为这个 `Predicate<S>` 设计独特的逻辑，比如小游戏中未加入任何队伍的玩家才能使用加入队伍的指令。

另外注意到我们这次为 `executes()` 传入的是一个 Lambda 表达式，具体是直接传入 `Command<ServerCommandSource>` 的变量，还是 Lambda 表达式，还是方法引用，应该根据你的自身喜好和具体实现来决定，比如如果指令实现逻辑复杂的话，使用 Lambda 表达式就不太合适了。

## 为指令添加子指令
为指令添加子指令，我们只需使用 `then()`。

```java
public class ModCommand implements ModInitializer {
    // ...
    @Override
    public void onInitialize() {
        CommandRegistrationCallback.EVENT.register((dispatcher, registryAccess, environment) -> {
            // All time say the words.
            dispatcher.register(Commands.literal("testcommand").executes(context -> {
                // sendSuccess(Supplier<Component> message, boolean broadcastToOps)
                context.getSource().sendSuccess(() -> Component.literal("Called /testcommand"), false);
                // Throw an exception when something goes wrong.
                //throw CommandSyntaxException.BUILT_IN_EXCEPTIONS.dispatcherUnknownArgument().create();
                // Return 1 when everything is ok.
                return 1;
            }).then(Commands.literal("subcommand").executes(context -> {
                    context.getSource().sendSuccess(() -> Component.literal("Called /testcommand subcommand"), false);
                    return 1;
                    }))
            );
        });
    }
}
```

如果觉得括号太多了，看起来很复杂，可以将代码复制到编辑器里面。

以下为简化版本。

```java
public class ModCommand implements ModInitializer {
    // ...
    @Override
    public void onInitialize() {
        CommandRegistrationCallback.EVENT.register((dispatcher, registryAccess, environment) -> {
            // All time say the words.
            dispatcher.register(
                    Commands.literal("testcommand").executes(/*something*/)
                    .then(
                            Commands.literal("subcommand").executes(/*something*/)
                    )
            );
        });
    }
}
```

发现了吗，`then()` 的参数实际上是一个新的 `Commands.literal()` ，而这个新的对象，同样可以使用 `then()` 继续套娃，得到一个树状的指令结构。

读者在此一定要分清楚， `then()` 是哪个对象调用的，就意味着传入的参数表示的指令是这个对象的子节点。即，区分

```java
A.then(B).then(C).then(D); // A -> B
                           //   -> C
                           //   -> D
A.then(B.then(C).then(D)); // A -> B -> C
                           //        -> D
A.then(B).then(C.then(D)); // A -> B
                           //   -> C -> D
A.then(B.then(C.then(D))); // A -> B -> C -> D 链式结构
```

之间的区别。

在设计比较复杂的指令结构时，最好经常换行缩进，或者写清楚注释。否则代码会难以阅读。另外，如果你觉得这种方式比较丑陋，后面我会补充一种比较有美感（个人认为）的写法。

## 为指令添加参数
在了解如何构造树状结构的指令后，我们还可以进一步让指令功能更完善，比如，就像原版的大部分指令一样，我们可以为指令添加参数。

```java
public class ModCommand implements ModInitializer {
    // ...
    @Override
    public void onInitialize() {
        CommandRegistrationCallback.EVENT.register((dispatcher, registryAccess, environment) -> {
            // Command with arguments
            dispatcher.register(Commands.literal("command_with_args")
                    .then(Commands.argument("value", IntegerArgumentType.integer(0)).executes(context -> {
                        int value = IntegerArgumentType.getInteger(context, "value");
                        context.getSource().sendSuccess(() -> Component.literal("You call the command with argument: %d".formatted(value)), false);
                        return 1;
                    }))
            );
        });
    }
}
```

我们不再使用 `literal()` 来创建节点，而是使用 `argument()` 来创建一个节点，它接受两个参数，第一个参数是参数的名称（如果没有自动补全，它会显示在文字框上面），第二个参数是参数的类型，一个 `ArgumentType<T>` 的对象。`integer()` 的唯一参数指定了参数的最小值， `integer()` 也有无参数和接受两个参数的重载版本，分别表示无范围限制和指定范围。

> 要说明的是，参数类型不仅仅在 `com.mojang.brigadier.arguments` 中（这个包中仅有一些泛用的、与 Minecraft 无关的参数类型），还可以在 `net.minecraft.commands.arguments` 包中找到。
{: .prompt-tip }

下面是一个更复杂的例子，它为这个指令添加了多个参数。

```java
public class ModCommand implements ModInitializer {
    // ...
    @Override
    public void onInitialize() {
        CommandRegistrationCallback.EVENT.register((dispatcher, registryAccess, environment) -> {
            // Command with arguments
            dispatcher.register(Commands.literal("command_with_args")
                    .then(Commands.argument("value", IntegerArgumentType.integer(0)).executes(context -> {
                        int value = IntegerArgumentType.getInteger(context, "value");
                        context.getSource().sendSuccess(() -> Component.literal("You call the command with argument: %d".formatted(value)), false);
                        return 1;
                    }).then(Commands.argument("player", StringArgumentType.string()).executes(context -> {
                        int value = IntegerArgumentType.getInteger(context, "value");
                        String player = StringArgumentType.getString(context, "player");
                        context.getSource().sendSuccess(() -> Component.literal("%s call the command with argument: %d".formatted(player, value)), false);
                        return 1;
                    })))
            );
        });
    }
}
```

## 自定义参数类型
下面我将演示如何创建一个自定义的参数类型。

## 为参数实现自动补全

## 在客户端注册指令

