---
title: 基于 SlackKit 创建 Slack bot
date: 2017-08-08 09:36:46 
---
前段时间研究了一下 Slack bot，目的是想通过 Slack 的对话式方式来完成一些机械式的任务，比如日常打包之类工作（有人也许会问为什么不用 Jenkins ？其实用了，只是顺带着研究了一把 Slack bot）。一开始尝试着用 [Botkit](https://github.com/howdyai/botkit) 来实现了一下与机器人简单进行对话，后来发现有人用 Swift 实现了一个基于 Slack API 的工具，叫 [SlackKit](https://github.com/SlackKit/SlackKit)，基于 Swift 用着比较熟的前提，就基于 Swift 的 protocol 一步步来实现一个简陋的通过对话来完成一些命令的功能。
<!-- more -->

我们一步一步来。

首先，如果只需要进行简单的对话，比如我问一下“你是谁”，机器人回复一下“我是xxx”，那就很简单。用 Swift Package Manager 新建一个引入 SlackKit 的 macOS 的命令行程序（这个步骤不详细说明了），然后新建一个 Robot 类，


``` swift
class Robot {
	let bot: SlackKit
    
	init(token: String) {
		bot = SlackKit()
		bot.addRTMBotWithAPIToken(token)
		bot.addWebAPIAccessWithToken(token)
		bot.notificationForEvent(.message) { (event, _) in
		    guard let message = event.message else {
		        return
		    }
		    self.handleMessage(message)
		}
	}
    
	func handleMessage(_ message: Message) {
 		guard let text = message.text, let channel = message.channel else {
			return
   	}
		if text == "Who are you" {
			bot.webAPI?.sendMessage(channel: channel, text: "👋👋", success: nil, failure: { error in
				print(error)
			})
			return
		}
	}
}

// main.swift

let robot = Robot.init(token: "your-token")
RunLoop.main.run()
```


运行程序，然后通过 Slack 像你的机器人发送 “Who are you”，就可以得到“👋👋”的回复了。

这样，是能够开始进行对话了，但是这样的方式只能够实现一个唯一的字符串命令对应一个唯一的回复文本，局限性很大。比如说我现在想实现这样一个对话：

1. 我发送给 bot 的文本："build"
2. bot 回复给我的文本："select app：1. first app；2. second app"
3. 我发送给 bot 的文本："1"
4. bot 回复给我的文本: "select scheme：1. xxx 2. xxx-inhouse"
5. 我发送给 bot 的文本："1"
......

上面这个任务，我期望在第 5 步之后得到一个针对于 xxx scheme 的操作，但是第 5 步和第 3 步我回复给 bot 的文本其实是一致的，这样就失去了上面所说的唯一的命令对应唯一的回复这样的前提了。再比如说，上面这个对话只是我想实现的其中一个打包 app 的任务，但是我还有很多别的任务，比如发邮件之类的，不同的任务之间都会有重复的一个命令，但是这些命令又想让它进行不同的回复或者说执行不同的操作，这时候该怎么做。

那我们不妨把事情分解一下，我现在有很多个任务，比如有 "build" 任务用来进行打包操作，有 "mail" 任务用来进行发送邮件的操作，其中每个任务里面又有很多个步骤，比如 "build" 任务中需要我分别输入我想要打包的 app ，所要打包 app 的分支、选择 scheme 等等，而 "mail" 任务呢需要我分别输入我想要发送邮件的 subject、receipent、body 等。所以按照这样的需求的话，我可以将我们想要实现的功能分为“任务“、”步骤”这两块，一个 Slack bot 可以执行不同的任务，但是同一时间只能执行一个任务，每个任务包含多个步骤，一个任务内一个步骤执行完开始执行下一个，直到任务内所有步骤执行完成之后该任务结束。

那怎样把上面所说的“任务”和“步骤”映射到 SlackKit 的使用中呢？可以这样，每个任务都有一个触发命令，一旦向 Slack bot 发送一个可以触发的命令后，就锁定该任务，开始按顺序执行该任务下的所有步骤，每个步骤执行也是通过一个发送给 bot 的文本来触发执行的。因为一个任务一旦开始执行到退出的阶段内，都会只去判断该任务下各个步骤的触发逻辑，这样就能够实现不同任务或者同一个任务下的多个步骤具有相同的触发文本也能够按照想要的逻辑来实现，也就是说能够实现上面所说的 1、2、3、4、5 这几个步骤。

好，基于以上的分析，那就很好做了，我们从小到大来进行。先说步骤，步骤是最小单元，是通过同 robot 的对话来进行的，用户端一个对话发送给 robot ，触发步骤，也就是说我们需要一个能够触发这个步骤的判断以及开始执行这个“步骤”的接口，我们将步骤称为 Step，所以映射到 Swift 代码就是这样：


``` swift
protocol Step {
	func isValid(msg: String) -> Bool
	func run(robot: SlackKit, msg: String, channel: String)
}
```


<code>isValid(msg: String)</code> 方法是用来判断用户所发送给 bot 的文本是否合法，换种说法就是用户发送给 bot 的文本是否能够触发这个步骤，如果能够触发的话，就调用 <code>run(robot: SlackKit, msg: String, channel: String)</code> 方法来开始执行。

有了步骤之后，我们可以开始进行任务的创建了，一个任务包含多个步骤，任务负责依次执行每一个步骤，当然，一个任务也需要有一个命令来进行触发，比如我向 bot 输入 "build" 那就应该触发 build 这个任务。我们把任务称为 Action，映射到 Swift 代码如下所示：


``` swift
protocol Action {
	var stepIndex: Int { get set }
	var triggerCommand: String { get }
	var steps: [Step] { get }
    
	func robot(_ robot: SlackKit, respondTo message: Message)
	mutating func robot(_ robot: SlackKit, runStepWithMessage message: Message)
}
```


<code>steps</code> 包含了这个任务中的所有步骤，<code>triggerCommand</code> 代表触发这个任务所需要的指令，当触发一个合法的指令时，通过 <code>robot(_ robot: SlackKit, respondTo message: Message)</code> 这个方法对用户进行一个回复，提示用户已经进入到了该任务当中，<code>stepIndex</code> 表征当前执行到了第几步，每个步骤的执行都通过方法 <code>mutating func robot(_ robot: SlackKit, runStepWithMessage message: Message)</code> 来进行，执行完一个步骤之后 <code>stepIndex</code> 加 1，所以这个方法是 mutating 的。

另外，既然有了 Action 的 <code>triggerCommand</code>，我们就可以通过这个属性来判断这个 Action 是否可以执行，所以我们为这个 Action 添加一个 extension 方法来判断这个 Action 是否可以响应：


```swift
func canRespond(_ command: String) -> Bool {
    return command == triggerCommand
}
```



有了上述这些方法的铺垫，接下来我们只需要进行任务的调用就可以了，任务的调用都是在 <code>func handleMessage(_ message: Message)</code> 方法中进行。因为一次只能进行一个任务，所以我们在 <code>Robot</code> 这个类中增加 


```swift
var currentAction: Action?
```


用来代表当前执行的“任务”。另外，这个机器人可以执行多项任务，所以我们创建一个这么多任务的集合:


```swift
var actions: [Action] = []
```


既然一次只能执行一个任务，且这个任务在执行中，那么就需要有方法能够退出当前正在执行中的任务，我们这里就规定发送 "exit" 来退出当前执行中的命令。所以在 <code>func handleMessage(_ message: Message)</code> 方法中应该变成这样：


```swift
func handleMessage(_ message: Message) {
    guard let text = message.text, let channel = message.channel else {
        return
    }
    if text.hasPrefix("exit") {
        currentAction = nil
        return
    }
    if currentAction == nil {
        actions.forEach {
            if $0.canRespond(text) {
                currentAction = $0
                return
            }
        }
        if let action = currentAction {
            action.robot(bot, respondTo: message)
        } else {
            bot.justForFun(message: text, inChannel: channel)
        }
    } else {
        currentAction?.robot(bot, runStepWithMessage: message)
    }
}
```


当 bot 每次收到一个文本时，先去看该文本是否为 "exit"，如果是的话就不管三七二十一把当前 action（无论是否有当前任务）都置空。如果不是当前文本的话，就先去看当前是不是在执行某个 action，是的话就依次执行该 action 下的 steps，如果不是在某项任务中，则去看该 bot 所能相应的所有 actions 中是否有某一个 action 可以相应 bot 收到的文本，能响应则创建 <code>currentAction</code> ，开始执行。

到了这一步，其实还是不太完善，因为 <code>handleMessage</code> 这个方法不仅仅是在用户给 bot 发送文本时候会触发，当 bot 给用户返回文本的时候也会触发，所以我们在调用该方法之前进行一下拦截，在 <code>bot.notificationForEvent(.message)</code> 的回调方法中判断当前文本是否是用户来触发的，用户触发的回调中的 event 会带有 user id，所以该方法变成这样：


```swift
bot.notificationForEvent(.message) { (event, _) in
	guard let _ = event.user?.id else {
		return
	}
	guard let message = event.message else {
		return
	}
	self.handleMessage(message)
}
```


至此，基本框架搭建完毕了，基于以上的框架，就可以把任务创建起来了。我尝试创建了一个叫做 Build 的任务，包含 BuildStepOne 和 BuildStepTwo 两个步骤，具体代码的话可以直接去 [Github](https://github.com/iCell/slack-robot) 上看一下。可以自己去 Slack 上弄一个 bot 的 token 尝试着玩一下，还是很有意思的。