---
title: "一起用 Chord 构建共享待办事项应用"
date: "2026-11-25T02:44:19.000Z"
description: "从一台空服务器开始，逐步构建一个小应用，直到两个人通过一个页面和一个终端使用它。文中涉及一个主题、三种语言、两人保持连接时应用的一次修改，以及对一次点击跨进程的追踪。"
tags: ["Chord"]
math: true
draft: false
---

教程 · Chord v2 及其应用套件 · 2026-11-03

# 共同构建它  

从一台空服务器开始，逐步构建一个小应用，直到两个人通过一个页面和一个终端使用它。文中涉及一个主题、三种语言、两人保持连接时应用的一次修改，以及对一次点击跨进程的追踪。

**11**步骤，一个可运行的示例

**4**宿主种类一种插件运行在

**1**每个列表的工作进程

**0**为新的主题、语言或编辑进行的重启

该应用是 `packages/chord/test/examples/61-tutorial/`，可通过其 `main.ts` 中的行运行。下方的每个代码块均引用自其文件，每个控制台输出均源自其步骤的打印内容：`test/examples/61-tutorial/tutorial.test.ts` 在引用或输出发生偏差时失败。设计是 `packages/chord/docs/design.md`；当本教程提到“规则”时，design.md 包含完整陈述。最后一章关于 p2，其代码位于此仓库之外。

## 目录

1. [在我们开始之前](#before-we-begin)
2. [I. 一个服务器](#server)
3. [II. 一个专属工作进程的列表](#worker)
4. [III. 路由](#routes)
5. [IV. 一个页面](#page)
6. [V. 一个终端](#terminal)
7. [VI. 一个主题](#theme)
8. [VII. 第二个用户](#user)
9. [VIII. 语言](#languages)
10. [IX. 更改代码](#reload)
11. [X. 移动列表](#export)
12. [XI. 查看发生了什么](#telemetry)
13. [XII. 测试一个插件](#testing)
14. [XIII. 编写 p2 插件](#p2)
15. [陷阱与开放性问题](#traps-and-open-questions)
16. [附录 A：应用程序的文件](#the-app-s-files)
17. [附录 B：引用](#reference)

## 开始之前

Mara 在她的笔记本电脑上保存着家庭的购物清单。她希望它能在浏览器标签页和终端中同时实时显示。Ada，她的室友，应该能够通过手机添加咖啡，无需账户：只需选择一个名字即可。Mara 还希望在应用运行时进行修改：在晚上添加深色主题，修复添加待办事项时的一个小问题，而无需将任何人赶出。

这是一个小型应用中四种程序（服务器、每个列表一个进程、页面、终端客户端）和三种变更（新数据、新人、新代码）。手动连接意味着需要一个 Web 服务器、一个套接字协议、一种启动和停止进程及会话的方法、一个页面打包器以及每个页面的重载故事。Chord 负责连接，而其 **app kit** (`@earendil-works/chord/app`) 代为做出应用本应自行做出的决定：数据存放何处、人员如何登录、插件如何被发现和发布。

本教程构建该应用。每一章添加一个功能，展示代码，展示运行时的效果，并陈述其背后的规则。该应用位于一个目录中，`test/examples/61-tutorial/`；从 `packages/chord` 一次性运行每一步：

```
node --conditions=source --experimental-strip-types test/examples/61-tutorial/main.ts
```

从 Node 26.5.1 或更高版本开始，第五章的终端客户端也会运行（其沙箱需要该 Node）；对于旧版本，该步骤会说明这一点，其余部分则继续运行。

### 十件事

1. **应用即其 API、插件和启动脚本。** `startServer` 创建数据目录、存储、宿主服务器、带登录的 HTTP、工作进程和插件根目录；`startWorker` 执行工作进程的一半。
2. **宿主是一个参与者。**服务器、每个列表的工作进程、每个浏览器标签页和每个终端都是宿主，通过链接连接。标签页和终端是客户端：它们连接并提问，没有人向它们提问。
3. **对象通过其契约被访问。**契约命名了对象类型、其参数及其成员；一个 `remote`契约还说明了哪个宿主为其提供服务。任何宿主请求 `Todos { list: "groceries" }`，Chord 将请求路由到该列表的工作进程。
4. **谁调用会被标记，永不发送。**每个调用都携带一个身份主体，该身份主体来自登录时的标记。工厂为每个身份主体选择一个变体；待办事项记录的是由哪个身份主体添加的，而不是由其参数添加的。
5. **每个插件每种宿主类型对应一个文件。** `server.ts`, `worker.ts`, `web.ts`, `tui.ts`: 每个声明该种宿主提供什么以及贡献什么。插件仅导入应用的 API。
6. **插件在注册表中相遇。**页面显示 `Panels`，终端 `Screens`，套件 `Themes` 重新样式化两者；插件贡献，一个插件（或入口）组合。
7. **一旦你这么说，更改就会生效。**保存文件会标记其插件为脏状态；`applyChanges`构建并部署到服务器、工作进程、页面和终端，就地生效。
8. **文本按每个查看者渲染。**插件的消息是类型化代码，翻译是插件，每个用户选择一种语言，服务器存储键，从不存储句子。
9. **一次点击即一次追踪。**服务器将遥测数据存储（此处为数据目录中的一个 SQLite 文件）；页面和工作进程发送其跨度；调用中的载具将页面的点击、服务器的中继和工作进程的方法合并为一次追踪，该追踪由 `TelemetryStore` 读取。`ctx.trace` 添加您自己的跨度。
10. **六个陷阱：** 在使用前加载；引用充当创建它的任何人；等待你开始的内容；监听从当前值开始；普通的 `t` 没有自己的语言；追踪从 span 开始。

### 贯穿全文的术语

一个 **宿主** 是应用的一个参与者，如果其他方可以访问它（服务器，**）则有一个** 地址`List({ id })`，否则没有（一个 **客户端**：一个页面、一个终端）。一个 **链接** 连接两个宿主。一个 **身份主体** 是发起调用的一方，由应用定义类型，由链接盖章。一个 **对象** 是某个身份下的状态和方法；其 **契约**（`local`仅针对其自身宿主，`remote`针对任何宿主）命名其 **参数**（即实例）和成员。远程契约的 **变体** 是不同身份主体获得的成员集合。一个 **工厂** 声明对象的组成部分：**文档** 和 **表**（存储的数据，在 **事务** 中更改），**实时查询**（Chord 保持表的当前行），**派生值**（由其他值计算得出），**原生组件**（打开和关闭的事物，如文件或数据库游标）。一个 **引用** 是请求对象时返回的内容。一个 **功能** 是针对一种宿主类型的插件代码；一个 **注册表** 是插件 **贡献** 的扩展点。一个 **消息** 是带有键的文本，a **catalog** 一种插件在一个语言中的消息，a **locale** 一种语言标签 (`de`, `pt-BR`)。A **rollout** 按顺序将新插件代码应用到宿主。A **span** 是宿主记录的一个定时工作片段，a **trace** 是跨越多个宿主的一个请求的片段，由 **carriers** 调用连接。

I

## 一个宿主

API、身份主体和启动脚本：套件运行的最小应用程序，以及谁可以进入。

使用套件构建的应用具有固定形状：其 API 在 `src/api/v<n>/`，其内置插件在 `plugins/`，一个启动脚本和一个工作进程入口（设计 §23，“应用套件”）。其余部分均为插件。本章编写这三段基础代码并启动服务器。

### 身份主体

A **身份主体**说明谁在调用。应用定义类型；套件需要服务器一个身份主体，每个工作进程一个（没有工作进程种类的应用则省略），以及一个身份主体，由所有者命名。待办事项应用给所有者一个自身的变体，`{ kind: "owner" }`；另一个应用可以让所有者成为其用户之一：

```
// The todo app's principals: its server and list workers, its owner, and the users who signed in with a name.

/** Who someone is in the todo app. */
export type TodoPrincipal =
	| { readonly kind: "server" }
	| { readonly kind: "worker"; readonly list: string }
	| { readonly kind: "owner" }
	| { readonly kind: "user"; readonly id: string; readonly name: string };
```

用户变体用于第七章。身份主体随每次调用旅行，是对象身份的一部分，因此保持小巧：一种类和 id。用户的档案，如果应用有的话，将是按 id 键值的数据对象。

### API

插件从不导入 Chord。它们导入应用的 **API**，即应用在其路径中指定版本的模块（`@todo/app/v1`）。它重新导出应用支持的 Chord 部分，并添加应用自身的部分：

```
export { BACKGROUND_CONTEXT } from "../../../../../../src/context/index.ts";
export {
	arr,
	bool,
	type Ctx,
	defineMessages,
	defineState,
	defineTable,
	eq,
	forbidden,
	int,
	invalid,
	local,
	MessageSchema,
	type MessagesOf,
	method,
	nul,
	num,
	obj,
	PluginError,
	query,
	remote,
	type Static,
	str,
	type Theme,
	Themes,
	type Translation,
	Translations,
	union,
} from "../../../../../../src/index.ts";
export type { TodoPrincipal } from "../../principals.ts";
```

`defineMessages`、`Translations`和消息模式见第八章。`invalid`和`forbidden`会引发 Chord 自身的错误：一个拒绝参数的插件会抛出`invalid("args", message)`，一个拒绝调用方的插件会抛出`forbidden(message)`，而调用方会收到与 Chord 内置检查相同的`{ code, reason }`（第九章）。`PluginError`用于插件自身的代码。

并且一个 `defineFacet` 绑定到身份主体类型，因此每个工厂都看到一个 `TodoPrincipal`：

```
/** Defines a facet of the todo app: its factories get the app's principals. */
export function defineFacet(setup: (host: FacetHost<TodoPrincipal>, self: FacetSelf) => void) {
	return chordFacet<TodoPrincipal>(setup);
}
```

The API 是一个 **宿主包**：插件的构建会将其排除，每个进程在评估插件时向其提供自己的实例。插件看到的是进程拥有的 `Themes`注册表和 `List`宿主类型，而不是副本。路径中的版本可以让应用与 `v2`后续 `v1`一起发布；一个方面使用一个版本（设计 §21，"插件模型"）。

API 也是一个真正的包：它是应用程序仓库中的一个 **workspace 包**，在 `package.json` 中命名，每个模块都有一个 `exports` 条目：

```
{
	"name": "@todo/app",
	"version": "0.0.1",
	"private": true,
	"description": "The tutorial's todo app: its API (v1 and its per-kind modules), base code and built-in plugins",
	"type": "module",
	"exports": {
		"./v1": "./src/api/v1/index.ts",
		"./v1/server": "./src/api/v1/server.ts",
		"./v1/web": "./src/api/v1/web.ts",
		"./v1/tui": "./src/api/v1/tui.ts"
	}
}
```

仓库的根 `package.json` 将其列在 `workspaces`（此处 Chord 的仓库列 `packages/chord/test/examples/61-tutorial`）中，因此 `npm install` 链接 `node_modules/@todo/app` 到它。然后 Node、TypeScript、测试和 esbuild 从基础代码、插件的文件以及测试导入插件的 `contract.ts` 中解析 `@todo/app/v1`，插件构建仍然将其排除在外并传递进程的实例。

### 启动脚本

`startServer` 执行应用决定并处理其余部分。`start.ts` 使用此进程模块、facet 种类（插件可能拥有的文件）和身份主体命名 API：

```
export const API = {
	name: "@todo/app",
	versions: ["v1"],
	modules: { "@todo/app/v1": v1, "@todo/app/v1/server": v1server },
};

/** The app's facet kinds: a plugin has one file per kind it runs on (`server.ts`, `worker.ts`, `web.ts`, `tui.ts`). */
export const KINDS = {
	server: { platform: "node" },
	worker: { platform: "node" },
	web: { platform: "browser", packages: { react: {}, "react/jsx-runtime": {} } },
	tui: { platform: "node" },
} as const;
```

```
export function startTodo(
	dataDir: string,
	root: string,
	telemetry: TodoTelemetry,
	context: Context,
): Promise<App<TodoPrincipal>> {
	return startServer<TodoPrincipal>(
		{
			name: "todo",
			dataDir,
			root,
			listen: { host: "127.0.0.1", port: 0 },
			api: API,
			kinds: KINDS,
			principal: {
				server: { kind: "server" },
				worker: (address) => ({ kind: "worker", list: address.key }),
				owner: { kind: "owner" },
			},
			// The store the server writes: its own spans and metrics, and what the workers and recording pages send.
			telemetry: telemetry.store,
			// Who a span was for: a short key and a user's name (each worker's `startWorker` gets the same hook).
			principalAttributes,
			workers: [
				{
					kind: v1.List,
					entry: fileURLToPath(new URL("./worker.ts", import.meta.url)),
					execArgv: ["--conditions=source", "--experimental-strip-types", "--no-warnings"],
					start: () => ({}),
					// Who may reach a list's worker: everyone signed in, and the server itself.
					authorize: (principal) => principal.kind !== "worker",
				},
			],
			// The page, and the owner's trace viewer: the same page, which shows the viewer at `/traces`.
			web: { entry: fileURLToPath(new URL("./src/web/main.ts", import.meta.url)), page, paths: ["/", "/traces"] },
			tui: { entry: fileURLToPath(new URL("./src/tui/main.ts", import.meta.url)) },
			onError: (error) => console.error("todo:", error.kind, error.error),
		},
		context,
	);
}
```

`workers` 是第二章，`web` 是第四章（其第二条路径，`/traces`，第十一章），`tui` 是第五章，`telemetry` 和 `principalAttributes` 是第十一章。`listen` 要求像示例一样在回环上获取一个空闲端口，而真实应用程序则选择一个端口。工作进程的条目只有几行（其 `principalAttributes` 也是第十一章的）：

```
// The todo app's worker entry: one process per list. `startWorker` opens the list's store in its directory, connects
// to the server and loads the plugins' worker facets. The worker records its spans in memory and sends them to the
// server, which stores them: its methods called through the server continue the caller's trace.

import { startWorker } from "../../../src/node/app/index.ts";
import { principalAttributes } from "./src/principals.ts";
import { API, KINDS } from "./start.ts";

startWorker({ api: API, kinds: KINDS, principalAttributes, onError: (error) => console.error("list:", error) });
```

### 运行它

`main.ts` 的第 1 步启动应用程序，尝试未登录访问页面，然后执行浏览器对所有者链接的处理：

```
const telemetry = await openTelemetry(telemetryFile(dataDir));
const app = await startTodo(dataDir, root, telemetry, context);
const link = new URL(app.ownerLink());
const url = link.origin;
console.log("1. owner link:", `http://127.0.0.1:<port>${link.pathname}#<token>`);
console.log("   data directory:", (await readdir(dataDir)).sort().join(", "));
console.log("   GET / signed out:", (await fetch(url)).status);
const signIn = await fetch(`${url}/_chord/owner`, {
	method: "POST",
	headers: { "content-type": "application/json" },
	body: JSON.stringify({ token: link.hash.slice(1) }),
});
const cookie = signIn.headers.get("set-cookie")!.split(";")[0]!;
console.log("   the owner signs in:", signIn.status, cookie.replace(/=.*/, "=…"));
console.log("   GET / signed in:", (await fetch(url, { headers: { cookie } })).status);
```

```
1. owner link: http://127.0.0.1:<port>/_chord/owner#<token>
   data directory: cache, logs, owner-token, plugins, server, telemetry.sqlite, telemetry.sqlite-shm, telemetry.sqlite-wal, workers
   GET / signed out: 401
   the owner signs in: 200 chord-session=…
   GET / signed in: 200
```

`startServer` 创建了数据目录：`server/` (其存储、cookie 密钥、其文件)，`workers/` (每个工作进程一个目录)，`plugins/` (用户插件根目录，位于应用内置目录旁)，`cache/` (每个工作进程类型的基目录，每次启动时重新写入，以及工作进程的 Node 编译缓存)，`logs/` 和 `owner-token`；`telemetry.sqlite` (包含 SQLite 的 `-shm` 和 `-wal` 文件) 是应用自己的，在服务器启动前打开 (第十一章)。页面回答了 401：HTTP   **默认拒绝**，除非路由声明 `public: true`，否则在运行任何插件代码之前。

The **owner link** (`/_chord/owner#<token>`) 是所有者进入的方式：一个单次使用的令牌，有效期十分钟，每次启动时生成。它位于 URL 的片段中，浏览器从不发送，因此它永远不会到达日志；链接所在的页面会发布它并接收一个会话 Cookie。同一台机器上的命令行客户端读取 `owner-token` 而不是它（模式 0600，第三章）。没有任何东西是所有者，因为它来自回环：任何本地进程都可以。

II

## 在自身工作进程上的列表

一个宿主类型，一个远程合约，一个带有表和实时查询的工作进程面，以及添加待办事项的身份主体。

每个列表都有自己独立的进程，一个**工作进程**, 拥有自己的存储：一个行为不当的列表无法让其他列表倒下，而一个空闲列表的成本为零。本章声明了列表的驻留位置、列表提供的功能，以及维护其待办事项的工作进程代码。

### 列表所在的宿主

一个 **host kind** 声明带有参数的可寻址宿主；`List({ id: "groceries" })` 是一个 **address**。应用的 host kinds 是基础代码，在 API 中：

```
/** A todo list: a worker process of its own, with its own store. */
export const List = hostKind("list", { id: str() });
```

`remote` 合约说明，与 `hostAddress` 一起，其对象驻留的地址。列表的 todos 的合约是 lists 插件的，在其 `contract.ts` 中，其每种类型的 facets 导入，因此步骤和测试也是如此：

```
import {
	arr,
	bool,
	int,
	List,
	MessageSchema,
	method,
	nul,
	num,
	obj,
	remote,
	type Static,
	str,
	union,
} from "@todo/app/v1";

/** A todo: its number in the list, its text, whether it is done, who added it and when. */
export const Item = obj({ id: int(), text: str(), done: bool(), by: str(), at: num() });

/** A todo (see `Item`). */
export type Item = Static<typeof Item>;

/** The list's latest notice, as data each viewer renders in its own language; `null` before the first. */
export const Notice = union([MessageSchema, nul()]);

/**
 * A list's todos as one principal sees them: the newest 50 (live) and the list's notice, adding one and ticking one off
 * (or on again); the owner also clears the done ones.
 */
export const Todos = remote("todo.todos", {
	version: 1,
	params: { list: str() },
	hostAddress: ({ list }) => List({ id: list }),
	variants: {
		member: {
			state: { items: arr(Item), notice: Notice },
			methods: { add: method(str()), tick: method(obj({ id: int(), done: bool() })) },
		},
		owner: {
			state: { items: arr(Item), notice: Notice },
			methods: { add: method(str()), tick: method(obj({ id: int(), done: bool() })), clear: method() },
		},
	},
});
```

其 `notice` 是作为数据的消息，第八章将其渲染为每个查看者的语言。 `params` 选择实例： `Todos { list: "groceries" }` 是每个身份主体一个对象。没有人询问时会命名一个宿主：调用方从参数中计算地址并将请求发送到该处；无法为其服务的宿主将其传递给其父节点或路由（设计 §3，"宿主类型和地址"）。套件中的 `workers` 选项是路由：

```
			workers: [
				{
					kind: v1.List,
					entry: fileURLToPath(new URL("./worker.ts", import.meta.url)),
					execArgv: ["--conditions=source", "--experimental-strip-types", "--no-warnings"],
					start: () => ({}),
					// Who may reach a list's worker: everyone signed in, and the server itself.
					authorize: (principal) => principal.kind !== "worker",
				},
			],
```

`authorize` 在路由到列表的每个请求之前、任何内容开始之前运行；地址的第一个请求启动其工作进程，工作进程空闲五分钟（默认 `idleMs`）则停止。**变体**是成员集提供的合约：所有者的 `Todos` 有 `clear`，成员的没有。工厂为每个身份主体选择一个。

### 工作进程面

The lists plugin 是一个目录，其中包含 `package.json`（其名称为插件的名称）以及每个其运行的 host 类型一个文件。`worker.ts` 是其 **worker facet**：每个 list 的 worker 所服务的。它将 todos 保存在 worker 的 store 的 **table**中：

```
/** Every todo of the list, by its number. */
const Items = defineTable("todo.items", {
	version: 1,
	columns: { id: int(), text: str(), done: bool(), by: str(), at: num() },
	key: ["id"],
});

/** The newest todos first. */
const NEWEST = query(Items).orderBy((item) => item.id, "desc");

/** How many todos the live query holds. */
const WINDOW = 50;
```

一张表持有许多记录，包含查询和页面；它存在于对象中，从不存在于 facet 中。持久化数据仅存在于 `local` 对象中（每个宿主和类型一个），且其地址不取决于谁发起请求（设计 §9，"Lifetime, persistence"）。该对象是插件的存储，`Store`，即其自身文件的本地契约：

```
/** The list's todos, once per worker: empty params, so it lives as long as the worker and may keep data in its store. */
const Store = local("todo.store", {
	version: 1,
	params: {},
	members: {
		items: arr(Item),
		notice: Notice,
		add: method(obj({ text: str(), by: str() })),
		tick: method(obj({ id: int(), done: bool() })),
		clear: method(obj({ by: str() })),
	},
});
```

一个 `local` 对象在宿主存在期间存活：它随工作进程启动，且从不因引用计数结束。其工厂声明其组成部分并返回其成员：

```
	host.serve(Store, (_params, self) => {
		const items = self.table(Items);
		// The newest todos, live: read before the store goes live, read again after every commit writing the table.
		const newest = self.query(items, NEWEST, WINDOW);
		const notices = self.state(Notices);
		return {
			// Oldest first, as the list shows them.
			items: self.derive([newest], ([rows]) => [...rows].reverse()),
			notice: self.derive([notices], ([value]) => value.notice),
```

A **factory** 是同步的且仅声明；Chord 随后激活该对象。`self.table` 声明了表格。`self.query(items, NEWEST, WINDOW)` 声明了一个 **live query** 用于它：`NEWEST` 的行，最多 50，作为值该对象可以暴露或从中派生。Chord 在存储上线前运行它，并在存储每次提交后再次运行，该提交写入 `items`，并仅向监听者发送已更改的内容。存储的派生将行反转，按最旧顺序排列，用于列表。

一个实时查询仅读取其自身对象的表：只有存储会写入它们，因此其自身的提交正是行可以改变的瞬间，且一个写入无表或查询无表的提交成本为零。限制是必需的：每个此类提交会再次运行一次查询，最多 `WINDOW` 行。这对于一个人编辑的列表来说很便宜，但对于每秒写入数百次的表来说则是错误的（设计 §11，"there may be dragons"）。

写入通过 **事务**：`self.tx(fn, ctx)` 在 `tx` 中运行 `fn`，当 `fn` 返回时，它所暂存的内容一次性提交，或者如果它抛出异常则完全不提交。实时查询紧随同一提交之后；存储不保留其他任何内容在内存中，甚至不保留下一个待办事项的编号，该编号是在事务中读取的：

```
			add: async ({ text, by }, ctx) => {
				// A span of its own under the method's: a trace shows the check, and why it refused.
				await ctx.trace("todos.validate", { chars: text.length }, async () => {
					// Refused as Chord refuses arguments that fail their schema: the caller gets `invalid/args`.
					if (text === "") throw invalid("args", "a todo needs text");
				});
				await self.tx(async (tx) => {
					// The number after the newest todo's: the store's transactions run one after another.
					const { items: last } = await tx.page(items, NEWEST, 1);
					tx.put(items, { id: (last[0]?.id ?? 0) + 1, text, done: false, by, at: Date.now() });
				}, ctx);
				return null;
```

没有文本的待办事项会被拒绝，错误为 `invalid("args", …)`，来自 API：错误会跨越所有链接，因此页面或终端中的调用方会得到 `{ code: "invalid", reason: "args" }`，就像那些违反合同模式的参数一样。检查运行在一个独立的 span 中（`ctx.trace`），第 XI 章在 trace 中展示了这一点。

### 每个身份主体的视图

`Todos` 是存储的远程端，每个身份主体对应一个对象。其工厂获取带有链接戳记的身份主体：

```
	host.serve(Todos, (_params, principal, self) => {
		const store = self.get(Store, {});
		// Who adds a todo comes from the principal the link stamped, never from the arguments.
		const by = principal.kind === "user" ? principal.name : principal.kind;
		const members = {
			items: self.derive([store.items], ([items]) => items),
			notice: self.derive([store.notice], ([notice]) => notice),
			add: (text: string, ctx: Ctx) => store.add({ text, by }, ctx),
			tick: (tick: { id: number; done: boolean }, ctx: Ctx) => store.tick(tick, ctx),
		};
		if (principal.kind !== "owner") return { variant: "member", members };
		return { variant: "owner", members: { ...members, clear: (ctx: Ctx) => store.clear({ by }, ctx) } };
	});
```

`self.get(Store, {})` 声明了一个依赖：视图在存储上线后开始运行，并在其整个生命周期内持有   **对它的**  引用。`by` 来自身份主体。客户端可以将任何内容放入 `add` 的参数中；它无法更改其链接的印章。这就是每个插件的规则：身份来自身份主体，而非来自参数。

### 运行它

Step 2 向作为所有者的根和服务器本身的根分别两次索取购物清单：

```
const OWNER = { kind: "owner" } as const;
await app.host.root(
	async (ctx) => {
		const todos = await ctx.get(Todos, { list: "groceries" });
		await todos.add("milk", ctx);
		await todos.add("eggs", ctx);
		await todos.tick({ id: 1, done: true }, ctx);
		await until(() => todos.items.value[0]?.done === true);
		console.log("2. as the owner:", todos.variant, "|", show(todos.items.value));
	},
	context,
	{ principal: OWNER },
);
await app.host.root(async (ctx) => {
	const todos = await ctx.get(Todos, { list: "groceries" });
	await todos.add("bread", ctx);
	await until(() => todos.items.value.length === 3);
	console.log("   as the server:", todos.variant, "|", show(todos.items.value.slice(-1)));
}, context);
```

```
2. as the owner: owner | [x] milk (owner), [ ] eggs (owner)
   as the server: member | [ ] bread (server)
   workers: list|groceries
   workers/list/list%7Cgroceries: store.sqlite, store.sqlite-shm, store.sqlite-wal
```

A **root** (`host.root(fn, ctx, { principal })`) 是对象外代码的调用作用域；其 `ctx` 携带身份主体和取消信息，而引用 `ctx.get` 在 `fn` 结束时失效。第一个请求启动了列表的 `list|groceries` 工作进程，该工作进程拥有独立的 `workers/list/list%7Cgroceries/` 存储。

### 陷阱：引用充当什么

两个根节点做出了相同的调用并获得了不同的待办事项：`milk (owner)`并且`bread (server)`. **引用充当创建它的任何人。** `ctx.get`在根或方法中充当该上下文的身份主体；`host.get`, 和`self.get`在一个`local`对象，作为宿主；`self.get`在远程对象中充当该对象的身份主体（设计 §4）。因此，来自所有者的请求到达工作进程时，其身份主体为所有者（第三章），而访客的调用无法借用服务器的权限。令人惊讶的是相反的情况：一个希望作为工作进程的服务器的服务器方法不能使用其调用方的 `ctx.get`；它通过在本地服务器对象中或在服务器自身的根中获取引用来使用 `self.get`。

### 陷阱：等待你所启动的内容

每一个写操作都会被等待，这并非风格问题。由上下文启动的内容以该上下文结束：当根节点的范围密封时`fn`settles，一个方法的上下文在其调用结束时结束。A`void todos.add(text, ctx)`在立即返回的根节点中，它在提交之前被取消；Chord 拒绝它并报告`transaction_cancelled`通过`onError`, 所以为空`catch`无法隐藏从未发生过的写入（设计 §8）。仅在与上下文不结束的情况下使用“放之四海而皆准”，正如页面所做的那样`BACKGROUND_CONTEXT`在第四章。

III

## 路由

一个服务器 facet：一个供命令行客户端使用的 JSON API，由发送请求的任何人运行。

该页面将通过 WebSocket 进行 Chord 通信。Mara 机器上的脚本更喜欢 HTTP。**server facet** 将路由贡献给 kit 的 `Http` 注册表，该注册表由 API 的服务器模块重新导出：

```
export { Http } from "../../../../../../src/node/http/index.ts";
```

```
// The lists plugin on the server: a JSON API for command-line clients. A route runs as whoever sent the request (the
// kit answered 401 before this code runs when nobody signed in), so its `ctx.get` reaches the list's worker as them.

import { defineFacet } from "@todo/app/v1";
import { Http } from "@todo/app/v1/server";
import { Todos } from "./contract.ts";

export default defineFacet((_host, self) => {
	self.contribute(Http, {
		routes: {
			"GET /api/lists/:list": async (request, ctx) => {
				const todos = await ctx.get(Todos, { list: request.params.list! });
				return { kind: "json", value: todos.items.value };
			},
			"POST /api/lists/:list": async (request, ctx) => {
				const todos = await ctx.get(Todos, { list: request.params.list! });
				await todos.add(String(await request.json()), ctx);
				return { kind: "json", value: "added" };
			},
		},
	});
});
```

**registry** 是一个扩展点：一个所有者声明贡献如何组合（此处 kit 的 HTTP 组件构建路由器），插件各自贡献一个值。`self.contribute` 在 facet 中，当插件加载时添加插件的路由。路由在根中运行，使用请求的身份主体，因此 `ctx.get(Todos, …)` 会到达列表的工作进程，无论谁登录，都会获取他们的变体。

步骤 3 是一个命令行客户端，拥有者令牌文件为：

```
const token = (await readFile(app.ownerTokenFile, "utf8")).trim();
const cli = { authorization: `Bearer ${token}` };
const posted = await fetch(`${url}/api/lists/groceries`, {
	method: "POST",
	headers: { ...cli, "content-type": "application/json" },
	body: JSON.stringify("butter"),
});
console.log("3. POST /api/lists/groceries:", await posted.json());
const listed = (await (await fetch(`${url}/api/lists/groceries`, { headers: cli })).json()) as Item[];
console.log("   GET /api/lists/groceries:", show(listed));
console.log("   GET without a token:", (await fetch(`${url}/api/lists/groceries`)).status);
```

```
3. POST /api/lists/groceries: added
   GET /api/lists/groceries: [x] milk (owner), [ ] eggs (owner), [ ] bread (server), [ ] butter (owner)
   GET without a token: 401
```

`butter` 是所有者：路由充当请求的身份主体。没有令牌时，套件在路由运行前返回 401。另一台机器没有令牌文件；它   **配对** ：它向 `POST /_chord/pair` 请求代码，所有者在已登录的浏览器中批准代码，客户端的下次轮询返回其令牌（示例 54）。

IV

## 一个页面

一个 Web 入口，一个带有 React、面板注册表和监听器从当前值开始的 Web 面。

一个页面是浏览器中的客户端宿主。该套件在服务器监听后（Chord、React、应用的 Web 模块；页面的首次请求等待它）立即构建其 **基础捆绑包**，并在其旁提供插件的 Web 面作为小型脚本，然后通过应用的模板提供页面（设计 §23，"Web 和 TUI"）。

### 页面的注册表

插件通过 API 的 web 模块声明的注册表将面板放置在页面上，Chord 的 React hooks 为它们重新导出：

```
import type { ComponentType } from "react";
import { defineRegistry } from "../../../../../../src/index.ts";

export { useHold, useHost, useMessages, usePageState, useWatch } from "../../../../../../src/react/index.ts";

/** A panel of the page: its name (a key, never shown) and its React component, which shows its own title. */
export type Panel = { readonly name: string; readonly Component: ComponentType };

/** The page's panels, one per contributing plugin, in plugin-name order. */
export const Panels = defineRegistry<Panel, readonly Panel[]>("todo.panels", { version: 1 });
```

面板显示其标题，页面语言；其 `name` 只是一个键。网页入口为 `Panels` 服务（其 `compose` 按插件名称顺序保持贡献），渲染每个面板，并启动页面的连接：

```
/** The page's own facet: it serves `Panels`, composing the contributions in plugin-name order. */
const core = defineFacet((host) => host.serve(v1web.Panels, { compose: (table) => table.map(({ value }) => value) }));

function Page() {
	const panels = useRegistry(v1web.Panels) ?? [];
	// The page's one other path (`start.ts` serves the page at `/` and `/traces`).
	if (location.pathname === "/traces") return h("main", { className: "wide" }, h("h1", null, "Todo"), h(TraceViewer));
	return h(
		"main",
		null,
		h("h1", null, "Todo"),
		...panels.map(({ name, Component }) => h("section", { key: name }, h(Component))),
	);
}

void startPage({
	connect: () => new WebSocket(`${location.origin.replace(/^http/, "ws")}/chord`),
	modules: { "@todo/app/v1": v1, "@todo/app/v1/web": v1web, react: React, "react/jsx-runtime": jsx },
	core: [{ name: "@todo/web", facet: core }],
	// The page records its clicks and the calls under them, and sends them to the server every few seconds.
	telemetry: true,
	render: (host) => createRoot(document.getElementById("root")!).render(h(HostProvider, { host }, h(Page))),
});
```

路径 `/traces` 是所有者 trace viewer，而 `telemetry` 展示的内容（第十一章）。`startPage` 使页面的宿主，将其连接到 `/chord`（断开后重连），加载当前插件脚本与 entry 的核心 facets，并保持它们最新。`HostProvider` 提供宿主 React 树。

### Web 端的 facet

`plugins/lists/web.ts` 贡献了一个面板。其组件在挂载时持有列表的 `Todos`，并在其项目更改时重新渲染：

```
function TodoList() {
	const { t } = useMessages(messages);
	const host = useHost();
	const list = location.hash.slice(1) || "groceries";
	const todos = useHold(Todos, { list });
	const items = useWatch(todos?.items) ?? [];
	const notice = useWatch(todos?.notice);
	// Kept by the page, not the component: the new code's component finds the half-typed todo.
	const [text, setText] = usePageState(`lists.draft.${list}`, "");
	const add = (event: { preventDefault(): void }) => {
		event.preventDefault();
		// An empty todo too: the list's store refuses it, and the click's trace shows where.
		if (todos === undefined) return;
		// The click is a trace of its own: a root span of the page, the call to the list's worker under it.
		const trace = { name: "page.add", attributes: { chars: text.length } };
		void host.root((ctx) => todos.add(text, ctx), context, { trace }).catch(() => {});
		setText("");
	};
	return h(
		"div",
		{ className: "lists-panel" },
		h("h2", null, t("title")),
		h(
			"ul",
			null,
			...items.map((item) =>
				h(
					"li",
					{
						key: item.id,
						className: item.done ? "lists-done" : undefined,
						onClick: () => void todos?.tick({ id: item.id, done: !item.done }, context),
					},
					item.text,
					h("span", { className: "lists-by" }, item.by),
				),
			),
		),
		h(
			"form",
			{ onSubmit: add },
			h("input", {
				value: text,
				placeholder: t("placeholder"),
				onChange: (event: { target: { value: string } }) => setText(event.target.value),
			}),
		),
		todos?.variant === "owner"
			? h("button", { type: "button", onClick: () => void todos.clear(context) }, t("clearDone"))
			: null,
		notice === undefined || notice === null ? null : h("p", { className: "lists-notice" }, t.render(notice)),
	);
}

export default defineFacet((_host, self) => {
	self.contribute(Translations, messages.translations);
	self.contribute(Panels, { name: "groceries", Component: TodoList });
});
```

`useMessages(messages)` 给出页面语言中的插件文本（第八章；`messages` 是插件的 `messages.ts`，且该 facet 将其目录贡献给页面的 `Translations`）。`useHold` 在组件挂载时持有引用，并在卸载时释放；它在首次渲染时 `undefined`。`useWatch` 跟随一个成员。`usePageState` 是 `useState`，页面将其值保存在一个键下而不是组件：第九章解释了为什么半类型的待办事项需要它。点击调用带有 `BACKGROUND_CONTEXT` 的方法，该方法永不结束，因此点击的调用不会在处理器返回时被取消；运行待办事项在 `host.root`（`useHost` 给出页面的宿主）中执行，这使得点击成为自己的追踪（第十一章）。`todos?.variant === "owner"` 缩小引用：只有所有者的 `clear`，且 TypeScript 知道这一点。面板的外观读取活动主题的 CSS 变量，类名前缀为插件名称（一个 `*.module.css` 文件会对其作用域）：

```
.lists-panel input { width: 100%; box-sizing: border-box; padding: 0.4rem; background: var(--bg); color: var(--fg);
	border: 1px solid var(--accent); border-radius: var(--radius, 0); }
```

在 `start.ts` 中的模板将套件制作的资产放入应用自己的 HTML 中：

```
function page(request: { readonly nonce: string }, assets: PageAssets): string {
	return `<!doctype html><html lang="en"><head><meta charset="utf-8"><title>Todo</title>${pageScripts(assets, request.nonce)}</head><body><div id="root"></div></body></html>`;
}
```

### 运行它

Step 4 以浏览器读取页面的方式读取页面，并监听来自服务器的列表：

```
const html = await (await fetch(url, { headers: cli })).text();
console.log("4. the page loads:");
for (const [, asset] of html.matchAll(/(?:src|href)="([^"]+)"/g)) console.log(`   ${hashless(asset!)}`);
const groceries = await app.host.get(Todos, { list: "groceries" }, context);
const watch = await groceries.items.watch(context);
console.log("   watch.value:", done(watch.value), "done");
const changed = Promise.withResolvers<readonly Item[]>();
watch.start(async (call) => {
	if (call.kind === "value") changed.resolve(call.value);
});
await groceries.tick({ id: 2, done: true }, context);
console.log("   the listener's first call:", done(await changed.promise), "done");
await watch.stop();
```

```
4. the page loads:
   /_chord/web/base.<hash>.css
   /_chord/web/plugin/@todo~lists.<hash>.<hash>.css
   /_chord/web/plugins.<hash>.js
   /_chord/web/base.<hash>.js
   watch.value: 1 done
   the listener's first call: 2 done
```

页面是模板，基础捆绑包（`base.<hash>.js` 和 `.css`），每个插件的一个合并文件包含所有 Web 脚本，每个插件一个样式表，CSS：冷加载时两个脚本，通过其哈希值永久可缓存。在所有者打开所有者链接后，页面显示列表：

![待办事项页面](http://47.93.196.89/content/images/chord-tutorial/page.png)

图 1. 所有者登录后显示的页面：杂货清单，实时显示，每个待办事项显示由谁添加；所有者的变体显示“清除已完成”。

### 陷阱：监听从当前值开始

`watch.value` 是监听时持有的值（一项待办完成）；监听器的第一次调用是之后的变化（两项完成）。 **监听器的监听器只会获取变化。** 仅从监听器渲染的代码在第一次变化之前不会绘制任何内容。先阅读 `handle.value`，然后阅读 `start`（设计 §11，“可监听对象”）。`useWatch` 和终端的 `TuiScope.watch` 会为您完成此操作；`self.on`，一个对象的例程，是第一个调用即为当前值的那个消费者。

V

## 一个终端

终端中的相同插件：一个带有 TuiScope 的终端 facet，在启动器的沙箱中运行。

终端客户端也是一个客户端宿主，由一个 **launcher** 运行：一个受信任的 Node 程序，它进行登录、下载终端的基础包和插件的终端脚本，并在一个 **sandbox** 中运行它们，这是一个没有文件系统、没有网络的子进程。拥有自己进程或终端。启动器转发其链接并绘制其框架。来自服务器的代码从不直接触及 Mara 的机器（决策“终端的沙箱是一个子进程”).

API 的终端模块声明了屏幕的组成部分：

```
export type Screen = {
	readonly name: string;
	readonly open: (scope: TuiScope, theme: TerminalTheme) => () => readonly string[];
};

/** The terminal's parts, in plugin-name order. */
export const Screens = defineRegistry<Screen, readonly Screen[]>("todo.screens", { version: 1 });
```

终端入口服务于 `Screens` 并在每次更改时重绘。一个 **`TuiScope`** 持有引用，跟踪值和注册表，在每次更改时调用一个回调，并在销毁时释放所有内容；它是任何终端库中 React hooks 的对应物：

```
/** The terminal's own facet: it serves `Screens`, composing the contributions in plugin-name order. */
const core = defineFacet((host) => host.serve(v1tui.Screens, { compose: (table) => table.map(({ value }) => value) }));

void startTerminalRuntime({
	modules: { "@todo/app/v1": v1, "@todo/app/v1/tui": v1tui },
	core: [{ name: "@todo/tui", facet: core }],
	render: (runtime) => {
		const { terminal, theme } = runtime;
		const screens = new TuiScope(runtime.host, () => draw()).registry(v1tui.Screens);
		let opened: { screens: readonly v1tui.Screen[]; scope: TuiScope; lines: (() => readonly string[])[] } | undefined;
		const draw = (): void => {
			const current = screens() ?? [];
			if (opened?.screens !== current) {
				opened?.scope.dispose();
				const scope = new TuiScope(runtime.host, () => draw());
				opened = { screens: current, scope, lines: current.map((screen) => screen.open(scope, theme)) };
			}
			const lines = opened.lines.flatMap((read) => read());
			terminal.write(`\x1b[2J\x1b[H${lines.join("\r\n")}\r\n`);
		};
		theme.onChange(draw);
		terminal.onInput((data) => {
			if (data === "q") runtime.exit(0);
		});
		draw();
	},
});
```

当重新加载更改 `Screens` 时，条目会销毁旧的 scope 并重新打开屏幕。列表插件的终端 facet 绘制其标题、待办事项和列表的通知；`scope.messages(messages)` 是终端的 `useMessages`（第八章）：

```
export default defineFacet((_host, self) => {
	self.contribute(Translations, messages.translations);
	self.contribute(Screens, {
		name: "groceries",
		open: (scope, theme) => {
			const t = scope.messages(messages);
			const todos = scope.hold(Todos, { list: "groceries" });
			const items = scope.watch(todos.items);
			const notice = scope.watch(todos.notice);
			return () => {
				const shown = notice();
				return [
					theme.fg("accent", t("title")),
					...(items() ?? []).map((item) =>
						item.done ? theme.fg("muted", `[x] ${item.text}`) : `[ ] ${item.text} ${theme.fg("muted", item.by)}`,
					),
					...(shown === undefined || shown === null ? [] : [theme.fg("warning", t.render(shown))]),
				];
			};
		},
	});
});
```

`theme.fg("muted", …)` 按角色着色，绝不按固定颜色着色。步骤 5 在脚本终端上运行启动器，其语言固定（默认情况下为机器的系统区域设置）：

```
if (sandboxAvailable()) {
	const output = Object.assign(new PassThrough(), { columns: 60, rows: 12 });
	output.on("data", (chunk: Buffer) => frames.push(chunk.toString()));
	terminal = await startTerminal({
		url,
		credentials: { kind: "file", path: app.ownerTokenFile },
		input: new PassThrough(),
		output,
		colors: "truecolor",
		// The machine's languages (its system locale by default), here fixed for the book's output.
		locales: ["en-US"],
	});
	await until(() => screen(frames).includes("butter"));
	console.log("5. the terminal shows:");
	for (const line of screen(frames).trim().split("\r\n")) console.log(`   ${line}`);
} else console.log("5. (the terminal client needs Node 26.5.1 or later for its sandbox)");
```

```
5. the terminal shows:
   Groceries
   [x] milk
   [x] eggs
   [ ] bread server
   [ ] butter owner
```

VI

## 一个主题

主题是在注册表中存储的数据；在应用运行时添加的插件会就地重新设计页面和终端。

Mara 想要晚上的外观。主题是数据：按角色配色，几个网页附加项，终端覆盖。套件中的 `Themes` 注册表收集它们；每个用户选择一个深色和一个浅色主题，系统偏好选择（决策“外观和主题”）。Harbor 主题是一个插件，其数据在一个文件中：

```
export const harbor: Theme = {
	id: "harbor",
	name: "Harbor",
	mode: "dark",
	colors: {
		accent: "#f2a65a",
		bg: "#12232e",
		fg: "#d8e2e8",
		muted: "#6b8796",
		error: "#e5676b",
		warning: "#e9c46a",
		success: "#7fb685",
	},
	web: { radius: "6px" },
	tui: { muted: "66" },
};
```

以及两个为其做出贡献的方面，一个用于页面，一个用于终端（注册表是按宿主划分的，因此每种类型都有自己的贡献）：

```
import { defineFacet, Themes } from "@todo/app/v1";
import { harbor } from "./theme.ts";

export default defineFacet((_host, self) => self.contribute(Themes, harbor));
```

插件在启动时不在应用的根目录。步骤 6 在一切运行时将其复制并应用更改：

```
await cp(fileURLToPath(new URL("./later/harbor/", import.meta.url)), join(root, "harbor"), { recursive: true });
await until(() => app.plugins.state.value.set);
const added = await app.applyChanges(context);
console.log("6. harbor added, rolled out to:", added.steps.map(({ name }) => name).join(", "));
const scripts = await app.host.get(WebScripts, {}, context);
console.log("   the page's plugins:", scripts.set.value.plugins.map(({ name }) => name).join(", "));
scripts.release();
```

```
6. harbor added, rolled out to: server, list, web, tui
   the page's plugins: @todo/french, @todo/harbor, @todo/lists
```

```
   the terminal redraws its title in Harbor's accent: "\u001b[38;2;242;166;90mGroceries"
```

出现在插件根目录中的目录标记了插件集已更改（`app.plugins.state.value.set`）；直到 `applyChanges` 发生任何操作，该操作构建新插件并将其部署：服务器、工作进程列表，然后是页面和终端表面。页面的宿主通过 Chord 获悉新的脚本集，获取 Harbor 的 Web 脚本并就地加载；终端对其终端脚本执行相同操作，并在 Harbor 的风格下重绘。当浏览器偏好 dark:

![Harbor 主题中的页面](http://47.93.196.89/content/images/chord-tutorial/harbor.png)

图 2. Harbor 插件添加并应用后的同一页面：无需重新加载，CSS 变量已生效 `:root` 已更改。

VII

## 第二个用户

一个单行登录插件，Ada 的客户端宿主，以及为什么宿主必须在它询问之前加载。

Ada 使用名称加入。套件随附一个临时的登录作为 facet（`POST /_chord/login { name }` 以 `{ kind: "user", id, name }` 的身份登录，不存储账户）；API 的服务器模块将其导出供应用的 principal 类型使用：

```
/** The kit's ephemeral login: `POST /_chord/login { name }` signs in as `{ kind: "user", id, name }`. */
export const login = ephemeralLoginFacet<TodoPrincipal>();
```

并且应用将其作为 **单文件插件**添加，该文件名为 `<name>.<kind>.ts`，位于根目录顶部：

```
// The kit's pick-a-name login, added to the app as a single-file plugin: a server facet, one line.

import { login } from "@todo/app/v1/server";

export default login;
```

Step 7 执行了 Ada 页面所做的操作：登录，然后通过 links 端点使用她的 cookie 连接她自己的客户端宿主：

```
const login = await fetch(`${url}/_chord/login`, {
	method: "POST",
	headers: { "content-type": "application/json" },
	body: JSON.stringify({ name: "ada" }),
});
const adaCookie = login.headers.get("set-cookie")!.split(";")[0]!;
const ada = createHost<TodoPrincipal>({ onError: (error) => console.error("ada:", error.kind) });
try {
	ada.hold(Todos, { list: "groceries" }, context);
} catch (error) {
	console.log("7. ada's hold before load:", (error as Error).message);
}
await ada.load([], context);
const socket = new WebSocket(`${url.replace(/^http/, "ws")}/chord`, { headers: { cookie: adaCookie } });
ada.connect(webSocketLink(socket as unknown as WebSocketLike), { principal: { kind: "server" } }, context);
const adas = await ada.get(Todos, { list: "groceries" }, context);
await adas.add("coffee", context);
console.log("   ada's variant:", adas.variant);
if (adas.variant === "member") {
	// @ts-expect-error `clear` is the owner's: TypeScript refuses it, and so does the worker.
	const clearing: Promise<null> = adas.clear(context);
	await clearing.catch((error: ChordError) => console.log("   ada's clear:", `${error.code}/${error.reason}`));
}
await until(() => groceries.items.value.some(({ by }) => by === "ada"));
console.log("   the server's reference sees:", show(groceries.items.value.slice(-1)));
```

```
7. ada's hold before load: todo.todos: the host has not loaded; call host.load first (a host with nothing to load calls load([]))
   ada's variant: member
   ada's clear: invalid/member
   the server's reference sees: [ ] coffee (ada)
```

### 陷阱：在使用前加载

Ada 的第一次`hold`一次性抛出：**在宿主之前，它不提供任何绑定`load`.**在它等待无人发布的发布之前就会等待，因此 Chord 拒绝它们`not_ready`("调用方`host.load`first"); 一个没有自带插件的客户端调用`load([])`(设计 §3).`startPage`, 启动器和套件会为您完成；使宿主自身必须记住的代码不能遗忘。

Ada 获得了 `member` 变体；她的 `clear` 被拒绝了两次，一次由 TypeScript（即 `@ts-expect-error`）拒绝，另一次由工作进程（`invalid/member`）拒绝，因为引用所提供的内容取决于提供者的变体，而非调用方的代码。她的待办事项属于她：`coffee (ada)`。

VIII

## 语言

代码中的消息，一个法语插件，一种现场选择的语言，以及每位观众各自阅读的通知。

Mara 读德语；Ada 更倾向于读法语。两者查看同一份列表，该列表由同一个工作进程提供，因此其文本不能为所有人统一格式化：**文本人们阅读的内容是逐个观看者渲染的，绝不由服务器生成**（决策“应用套件中的消息和语言”）。插件只需在代码中编写一次文本；各语言通过贡献项提供，每个用户可以自行设置语言偏好。

### 代码中的消息

列表插件将文本保存在同一个文件中，`messages.ts`, 每个渲染导入（以及下方的工作进程）的面：

```
export const messages = defineMessages("todo.lists", {
	en: {
		title: "Groceries",
		placeholder: "Add a todo",
		clearDone: "Clear done",
		open: (n: number) => (n === 1 ? "1 open" : `${n} open`),
		cleared: (by: string) => `${by} cleared the done todos`,
	},
	de: {
		title: "Einkäufe",
		placeholder: "Neues Todo",
		clearDone: "Erledigte entfernen",
		open: (n) => `${n} offen`,
		cleared: (by) => `${by} hat die erledigten Todos entfernt`,
	},
});
```

第一个目录定义了键及其参数类型，且英文目录是权威来源；德语目录从中获取函数的参数类型，缺失的键、未知的键或错误的参数（`t("open", "3")`）在生产环境中是类型错误，而非空白。命名空间（`todo.lists`）区分了两个插件的 `title`。函数涵盖复数形式和词序；其中的日期和数字经过 `formatTime` 和 `formatNumber` 处理，后者使用渲染时的区域设置。

面板以 `useMessages(messages)` 渲染，终端屏幕以 `scope.messages(messages)` 渲染（见第四章和第五章）：两者均提供 `t` 绑定到其宿主语言，并在其更改时重新渲染或重绘。每个进行渲染的面片也会将其插件的目录贡献到其宿主的 `Translations` 中：

```
export default defineFacet((_host, self) => {
	self.contribute(Translations, messages.translations);
	self.contribute(Panels, { name: "groceries", Component: TodoList });
});
```

注册表是按宿主划分的，宿主从中学习其插件提供的语言：页面在选择一个之前，会将其语言与注册表中的语言进行匹配。

### 一个翻译插件

French 来自一个独立的插件，由从未接触 lists 插件的人编写。其目录由 lists 插件的消息类型化，这是一种仅在构建时擦除的类型导入：

```
import type { MessagesOf, Translation } from "@todo/app/v1";
import type { messages } from "../lists/messages.ts";

const lists: MessagesOf<typeof messages> = {
	title: "Courses",
	placeholder: "Nouvelle tâche",
	clearDone: "Retirer les tâches faites",
	open: (n) => `${n} à faire`,
	cleared: (by) => `${by} a retiré les tâches faites`,
};

export const french: readonly Translation[] = [{ namespace: "todo.lists", locale: "fr", messages: lists }];
```

并且其 Web 和终端界面贡献了该列表，每条都相同：

```
export default defineFacet((_host, self) => self.contribute(Translations, french));
```

贡献是一个列表，因此一个插件可以一次翻译整个应用和所有插件。目录可能是部分的：缺失的键回退到本地化的基础（`pt-BR` 到 `pt`），然后回退到命名空间的第一个目录，逐个键回退，每次回退只报告一次（页面的 `onError`，类型 `messages`）。一个命名空间和本地化的两个目录：第一个按插件名称获胜，另一个则被报告；用 `override: true` 贡献的翻译会替换插件自己的（更好的德语）。

### 选择语言

语言是用户保存在服务器上的偏好，`LocaleChoice`，每个身份主体一个，就像第六章的主题选择一样。直到用户选择，设备决定：浏览器在页面上的 `navigator.languages`，终端中的系统区域设置（启动器的 `locales`），与目录提供的语言进行匹配，否则为英语。步骤 8 中，所有者选择德语，而终端正在运行：

```
const ownerChooses = (locale: string) =>
	app.host.root(async (ctx) => (await ctx.get(LocaleChoice, {})).choose(locale, ctx), context, { principal: OWNER });
await ownerChooses("de");
```

```
8. the owner chooses German
   ada's pick: fr
```

```
   the terminal shows:
   Einkäufe
   [x] milk
   [x] eggs
   [ ] bread server
   [ ] butter owner
   [ ] coffee ada
```

每个所有者持有的 `LocaleChoice` 都会导致终端立即用德语重绘；没有任何东西重启。Ada 的选择（法语）属于她本人，且不会改变所有者。一个页面也会这样做，并在其根元素上设置 `lang` 和 `dir`（对于阿拉伯语或希伯来语，通过应用程序的逻辑 CSS 属性，方向为从右到左）。

### 来自宿主的文本

清除已完成的待办事项应通知所有清除它们的人。如果工作进程写入"所有者已移除已完成的待办事项"，Ada 也能读懂德语。因此，列表的状态携带了 **消息作为数据**，包含一个键及其参数，由套件的一般模式声明：

```
/** The list's latest notice, as data each viewer renders in its own language; `null` before the first. */
export const Notice = union([MessageSchema, nul()]);
```

商店使用 `messages.message` 将其写入，类型针对英文目录像 `t`（`messages.message("cleared", 42)` 无法编译），每个查看器使用 `t.render` 对其进行渲染：

```
			clear: async ({ by }, ctx) => {
				await self.tx(async (tx) => tx.deleteWhere(items, (item) => eq(item.done, true)), ctx);
				// The key and its parameters, typed against the English catalog: every viewer renders its own text.
				await notices.change((draft) => {
					draft.notice = messages.message("cleared", by);
				}, ctx);
				return null;
			},
```

```
   the notice: { namespace: 'todo.lists', key: 'cleared', params: [ 'owner' ] }
   the server renders it: owner cleared the done todos
```

```
   the terminal renders it: owner hat die erledigten Todos entfernt
```

参数是普通数据（字符串、数字、ISO 日期），因此消息跨越链接的方式与任何状态相同。在链接中，成员是通用模式，因此 `t.render` 在运行时检查命名空间和键：另一个命名空间的消息或目录不再拥有的键会显示该键并报告。从不翻译：模型的提示和回答、日志行、错误代码。

### 陷阱：普通的 `t`

该服务器的 `messages.render(…)` 打印了英文。   **纯 `t` 没有自己的语言：** 在渲染时它使用正在渲染的语言（一个消息函数调用另一个插件的 `t`， `formatTime`），而在外部则使用第一个目录。这对于服务器、测试或日志行是正确的，但对于组件是错误的：一个页面或终端使用 `useMessages(t)` 或 `scope.messages(t)` 进行渲染，绑定于其宿主语言。没有模块范围的当前语言，因为一个进程运行多个宿主，而服务器为多个人渲染。

在页面上，`screenshots.ts` 中所有者选择法语并清除正在输入一半的待办事项：

```
	await app.host.root(
		async (ctx) => {
			await (await ctx.get(LocaleChoice, {})).choose("fr", ctx);
			const todos = await ctx.get(Todos, { list: "groceries" });
			if (todos.variant === "owner") await todos.clear(ctx);
		},
		context,
		{ principal: { kind: "owner" } },
	);
```

```
wrote docs/tutorial/screens/french.png
after choosing French: lang="fr", "oat mi" still typed
the page did not reload: true
```

![法文页面](http://47.93.196.89/content/images/chord-tutorial/french.png)

图 3. 所有者选择法语后的页面：法语插件的文本列表、由其键渲染的通知、半输入的“燕麦米”保留；无重载。

IX

## 更改代码

当 Ada 连接时进行编辑：监听者标记，applyChanges 滚动，每个对象承担其命运。

Ada 输入了带空格的 " tea "，空白待办事项应像空的一样被拒绝。修复方法是 lists 插件的工作进程 facet 中的一行代码：视图在存储看到之前会修剪文本。Step 9 使编辑在运行中的应用的插件副本中生效，就像编辑器或代理一样，并应用它：

```
const BEFORE = "add: (text: string, ctx: Ctx) => store.add({ text, by }, ctx),";
const AFTER = "add: (text: string, ctx: Ctx) => store.add({ text: text.trim(), by }, ctx),";
const workerFile = join(root, "lists", "worker.ts");
await writeFile(workerFile, (await readFile(workerFile, "utf8")).replace(BEFORE, AFTER));
await until(() => app.plugins.state.value.dirty.includes("@todo/lists"));
const applied = await app.applyChanges(context);
console.log("9. lists changed, rolled out to:", applied.steps.map(({ name }) => name).join(", "));
for (const host of applied.steps.find(({ name }) => name === "list")!.hosts) {
	const fates = "report" in host ? host.report.objects.map(({ object, kind }) => `${object} ${kind}`) : [];
	console.log(`   on ${host.host}:`, [...new Set(fates)].join(", "));
}
await adas.add("  tea  ", context);
await until(() => groceries.items.value.at(-1)?.text === "tea");
console.log("   ada adds '  tea  ':", show(groceries.items.value.slice(-1)));
const refused = await adas.add(" ", context).then(
	() => "added",
	(error: ChordError) => ({ code: error.code, reason: error.reason }),
);
console.log("   ada adds ' ':", refused);
```

```
9. lists changed, rolled out to: server, list, web, tui
   on list|groceries: todo.store restarted, todo.todos(groceries) restarted
   ada adds '  tea  ': [ ] tea (ada)
   ada adds ' ': { code: 'invalid', reason: 'args' }
```

监听器在文件变更时标记 `@todo/lists` 为脏；`applyChanges` 重新构建了它（每个 facet 一次 esbuild 调用）并按宿主逐个部署。在每个宿主上，Chord 会部署新代码（它会在录制 `self` 上运行新工厂以查看其声明），并在一步中提交它，然后为每个变更插件的对象分配其 **命运**（设计 §21）：

| 对象             | 命运                                |
| -------------- | --------------------------------- |
| 直播             | **重启**：旧代码上的调用完成，新调用等待新代码，相同的文档和表 |
| 不活跃（失败、挂起、激活中） | **在新区块上重试**                       |
| 其新代码等待依赖       | **在依赖项处于活跃状态之前暂停**                |
| 其类型已被移除        | **已结束**; 其持有者等待再次为其服务             |

由于存储和视图都处于活跃状态，因此 **重新启动了**。列表工作进程上的序列：

1. 旧尝试保持开启：调用已运行并在旧代码上完成，它们所做的每一次 `ctx` (**旧工作**：视图的 `add` 运行到达存储的旧尝试，从不等待新的) 也随之完成。现在到达的调用是新工作：它们在门处等待。
2. 每个旧尝试在其调用耗尽且其在 staging 中由新工厂创建的请求就绪时会被密封（视图的新代码需要存储，而旧的尝试仍在为其服务）。这花费的时间等于正在运行的调用时间，最多为 `reloadDrainTimeout` (5 s)；此时仍在运行的调用会被切断。
3. 新的尝试运行其工厂，其实时查询再次运行（与第一次激活时的读取相同），并上线：门打开，引用按名称移至其成员，每个成员带有其最后一个值的一个帧。

Ada 的引用保持 `live`，旧代码耗尽时显示 `loading` 片刻，随后是新代码的值；她的下一个添加被修剪。长时间的工作不应属于一个方法：超过耗尽边界的调用会在重载时被切断，因此它进入一个作业。

Ada 的空待办事项在修剪后已为空，因此存储拒绝它，返回了 `invalid("args", …)`（第二章）：她的客户端收到了 `{ code: "invalid", reason: "args" }`，这是 Chord 自身的错误，该错误沿着工作进程和服务器原有的链路传递。如果编辑破坏了插件，该插件将保留其旧代码，而报告将命名该错误。

需要新 `version` 的合约会改变形状；当提供者映射其版本（`accepts`）时，旧消费者继续工作，否则它们等待 `contract_mismatch` 直到重新加载。存储文档或表改变形状会将其版本与迁移一起更新。合约版本保护调用方，存储版本保护数据。

### 在页面上

Web 侧面的编辑就地应用：页面获取更改的脚本并重新加载其宿主，无需页面重新加载。`screenshots.ts` 在面板中添加打开的待办事项计数（`t("open", n)`，使用页面的语言），当输入一半待办事项时：

```
const BEFORE = `h(
			"ul",`;
```

```
wrote docs/tutorial/screens/page.png
wrote docs/tutorial/screens/harbor.png
wrote docs/tutorial/screens/edited.png
typed "oat mi"; after the edit: "oat mi"
the page did not reload: true
```

![编辑后的页面](http://47.93.196.89/content/images/chord-tutorial/edited.png)

图 4. 在编辑 Web 面后，就地应用：出现计数，以及半输入的“燕麦米”仍在盒子中：页面在 React 挂载新组件时一直将其保留在其键下。

页面未重新加载，更改插件组件之外的所有内容保持了其状态。更改后的面板本身是新代码：其组件是一个新函数，React 会重新挂载它，因此其自身的 `useState` 会重新开始（设计 §21）。必须在一个插件更新中保持的内容（一个半类型的待办事项、滚动位置）是 **页面状态**： `usePageState(key, initial)`在页面的宿主下以键值形式保存该值，而不是保存在组件中，因此新组件可以找到它；终端的 `TuiScope`与 `scope.pageState(key, initial)`拥有相同的存储。页面状态是任何值，永远不会发送到任何地方，并随着页面的消失而消失。

X

## 移动列表

在运行期间导出工作进程的数据，在另一个地址下导入它。

列表的数据是由其工作进程创建的：其存储和文件。步骤 10 在运行其工作进程时导出杂货列表，并将其作为新列表导入：

```
const archive = join(base, "groceries.tar");
await app.exportData(List({ id: "groceries" }), archive, context);
console.log("10. exported groceries:", (await readFile(archive)).length > 0 ? "an archive" : "nothing");
await app.importData(archive, List({ id: "picnic" }), context);
const picnic = await app.host.get(Todos, { list: "picnic" }, context);
console.log("   imported as picnic:", show(picnic.items.value));
picnic.release();
await app.importData(archive, List({ id: "groceries" }), context).catch((error: Error) => {
	console.log("   importing over groceries:", error.message.replace(base, "…"));
});
```

```
10. exported groceries: an archive
   imported as picnic: [ ] bread (server), [ ] butter (owner), [ ] coffee (ada), [ ] tea (ada)
   importing over groceries: importData: list|groceries has data (its worker runs)
```

`exportData` 停止一切：运行中的工作进程在一致点（存储导出时持有其提交队列，同时读取）写入自己的存储及其文件，作为一个包含应用名称的清单的归档。没有工作进程运行时，服务器获取工作进程的处理锁并自行读取存储。`importData` 拒绝包含数据的地址；替换列表是删除然后导入（决策“应用套件命名和导出”）。如果在应用的索引中注册了导入的列表，则该列表即为应用。

XI

## 看到发生了什么

服务器上的一个遥测存储，从浏览器中的点击到工作进程上的提交的一个追踪，您自己的一个跨度，每个跨度是为谁服务的，以及为所有者提供的一个小型查看器。

Ada 说，有些晚上从手机添加待办事项感觉很慢。她的待办事项跨越三个进程：她的页面、服务器的中继和列表的工作进程。每个进程一条日志行会说明各自做了什么，但不会说明哪些行属于她的点击，或者时间花在哪里。Chord 通过 pi-telemetry 记录 **遥测**：**跨度**（带有属性的命名、计时的工作片段）、**跨度的事件**以及 **指标**（计数器和直方图）。每个宿主都有一个 **适配器**，默认由 `createHost({ telemetry })`和 NOOP 提供，它决定它们去哪里。每个调用头中的一个 **载体**命名调用方的跨度，因此由不同进程记录的同一个请求的跨度形成一个 **追踪**（设计 §29）。Mara 想在不读取 SQL 的情况下查看它们，一个小型追踪查看器会很棒。本章将遥测数据存储在一个地方，跟踪从浏览器到工作进程的点击，添加一个跨度，命名每个跨度是为谁服务的，从脚本中读取存储，并以查看器结束。

### 一个存储，在宿主上

应用程序将其遥测数据保存在 data 目录中的一个 SQLite 文件中，通过 Chord 的存储（`@earendil-works/chord/telemetry-sqlite`），且只有服务器进程会写入该文件。应用程序的基础代码为服务器打开该文件：

```
/** The telemetry store's file in the data directory `dataDir`. */
export function telemetryFile(dataDir: string): string {
	return join(dataDir, "telemetry.sqlite");
}

/** Opens the telemetry store at `file` for this process (its directory made, private, when missing). */
export async function openTelemetry(file: string): Promise<TodoTelemetry> {
	// Spans carry who called what: the data directory is private before the store's first file appears in it.
	await mkdir(dirname(file), { recursive: true, mode: 0o700 });
	const db = await openNodeSqliteDatabase(file);
	const store = await createSqliteTelemetry(db);
	return {
		store,
		close: async () => {
			await store.close();
			await db.close();
		},
	};
}
```

`createSqliteTelemetry` 返回一个 **recorder**：一个适配器，它将每个已结束的 span 保存在内存中，并每秒批量写入一次，每个 span 一行，包含其事件和链接在内联，以及以微秒为单位的开始和结束时间（一个锚定到墙钟的单调时钟，因此同一进程的 span 保持其顺序和间距）；指标按系列求和，并每 10 秒汇总一次；超过七天的行会在同一批次中删除，并且当 10,000 spans 等待时，新的 ones 会被丢弃并计数。记录永远不会运行 SQL。该对象也是存储的 **read interface**，`TelemetryStore`（见下文）。

步骤 1 在服务器（第一章）之前打开了商店，并将 `startTodo` 交给 `startServer`：

```
			// The store the server writes: its own spans and metrics, and what the workers and recording pages send.
			telemetry: telemetry.store,
```

有了存储，套件就负责其余部分。每个列表的工作进程将数据记录在内存中，并每秒通过其已有的链接将其完成的跨度指标发送到服务器；没有任何东西告诉工作进程文件在哪里，工作进程的条目也没有自己的遥测数据来设置：

```
startWorker({ api: API, kinds: KINDS, principalAttributes, onError: (error) => console.error("list:", error) });
```

一个页面记录当应用程序在其 Web 入口中要求它时：

```
	// The page records its clicks and the calls under them, and sends them to the server every few seconds.
	telemetry: true,
```

该页面将完成的 span 保留在内存中，每三秒发送一次，并在隐藏时立即发送（标签页切换后可能无法返回）。两者都发送给 kit 在服务器上提供的对象 `WorkerTelemetry` 和 `PageTelemetry`，其 `ingest(batch)` 将 span 传递给存储；其调用不记录任何 span，否则每个批次都会写入关于自身的 span。一个 worker 是服务器自己的子进程：其批次在写入时标记，并标记其 host。一个 page 是任何人的：登录和链接的限制适用于任何 call。并且服务器会检查所有内容（下一节）。

### 来自浏览器的一个追踪

The lists plugin 的面板启动了一个追踪，从 Ada 的点击开始：在其表单的 `submit`，即页面宿主的一个根 span，包含其下的调用：

```
		// The click is a trace of its own: a root span of the page, the call to the list's worker under it.
		const trace = { name: "page.add", attributes: { chars: text.length } };
		void host.root((ctx) => todos.add(text, ctx), context, { trace }).catch(() => {});
```

`host.root(fn, context, { trace })` 在宿主适配器的新根跨度中运行 `fn`，`ctx.telemetry` 该跨度，因此 `add` 的调用在其下开始其 `chord.call` 并将该跨度的载体放在调用的头部（`useHost` 给组件其页面的宿主）。没有记录适配器时它什么都不开始。`screenshots.ts` 拥有者已登录到 Chromium 中的真实页面（第四章中的应用程序页面，现在在 Harbor 和法语中），输入 honey 并提交，然后再次提交但未输入任何内容：

```
	await browser.type("honey");
	await shoot("click-before.png");
	const submit = "document.querySelector('.lists-panel form').requestSubmit()";
	await browser.evaluate(submit);
```

```
wrote docs/tutorial/screens/click-before.png
wrote docs/tutorial/screens/click-after.png
```

![输入了 honey 的页面](http://47.93.196.89/content/images/chord-tutorial/click-before.png)![点击后的页面：列表中的蜂蜜](http://47.93.196.89/content/images/chord-tutorial/click-after.png)

图 5. 点击真实页面：honey 输入（左），然后提交（右）。列表在 worker 的 store 提交它且其 live query 到达页面时显示 honey；页面已记录 `page.add` 并且 `chord.call` 在它下面，并在三秒内将它们发送到服务器。

随后，它读取存储，直到工作进程的两次点击的跨度都存在，并打印蜂蜜的带有 pi-telemetry 的 `renderTrace`（ID 和时间长度被屏蔽，每次运行都不同）的追踪，并在同一追踪上打开所有者的追踪查看器（最后部分构建了它）：

```
adding honey from the page:
   page.add chord.host="browser" chars=5 chord.principal="owner"
     chord.call chord.host="browser" type="todo.todos" member="add" object="todo.todos(groceries)" address.kind="list" outcome="ok" chord.principal="owner"
       chord.relay chord.host="server" type="todo.todos" member="#0" upstream.kind="list" outcome="ok"
         chord.method chord.host="list|groceries" type="todo.todos" member="add" object="todo.todos(groceries)" principal.kind="owner" chord.principal="owner" outcome="ok"
           chord.method chord.host="list|groceries" type="todo.store" member="add" object="todo.store" principal.kind="worker" chord.principal="worker:list|groceries" outcome="ok"
             · chord.commit documents=0 ops=0 rows=1 duration=<ms>
             todos.validate chord.host="list|groceries" chars=5
```

![带有点击追踪的追踪查看器](http://47.93.196.89/content/images/chord-tutorial/traces.png)

图 6. 所有者 trace viewer 中的相同 trace 在 `/traces`, in the Harbor 主题：树 `renderTrace` 如上所述，每个 span 及其记录它的宿主（`browser`, `server`, `list|groceries`）及其在追踪时间线上的长度作为一条条，全部由服务器编写。开始时间存储到微秒，因此即使毫秒级的小部分也能对齐。在左侧是最新的追踪：被拒绝的点击为红色，每个都带有其对应的对象；其余为应用程序自身的工作（中继、激活、HTTP 请求）。

三个进程记录了这六个跨度，服务器将它们全部写入。每个跨度说明是由哪个宿主记录的：`chord.host`是宿主地址键（`server`、`list|groceries`），`browser`是页面发送的内容。页面记录了`page.add`和`chord.call`，即调用方直到其回复的调用。服务器记录了`chord.relay`：它将调用转发给列表的工作进程，而无需知道成员名称（`#0`是方法的索引）。工作进程记录了其余部分：`chord.method`是所有者`Todos`视图的视图，然后是视图对存储器的调用，这也是一个方法跨度，由`worker`记录，因为存储器充当其宿主（第二章）。提交是跨度上运行的事务的事件，渲染器在打印跨度的事件之前打印其子事件。它计算了它更改的文档数量、它们的操作以及它写入更改的表格行：存储将待办事项保存在表格中，因此是一行且没有文档。`chord.principal`说明跨度是为谁服务的（下方）。

页面浏览器服务器存储list|购物清单工作进程页面添加，Chord 调用调用 add，作为 chord.call 的载体chord.relay调用 #0，Chord.relay 的载体chord.method ×2、todos.validate、commit答复答复ingest: 其 spans（每秒）ingest: 其 spans（每 3 s，或隐藏）

回放图 7. 一键一迹：每个进程启动时，使用自己的适配器生成其跨度；每次调用的头部名称中，载体指定了对端的父跨度；页面和工作进程将其跨度发送给服务器，服务器负责写入它们。

载体是 `call` 和 `hold` 头部的字段，最多 8 个键，每个键最多 256 个字符。接收方使用其自身的适配器读取它；缺失或格式错误的载体会生成一个新的根，绝不会报错。使用 NOOP 适配器时不会注入任何内容，也不会开始任何跨度：一个跨度站点需要一次检查（§30）。

发送的页面内容，服务器视为不可信。批次通过其模式检查：最多 100 个跨度，每个字段有界（名称 128 个字符，32 个属性，每个最多 256 个字符，16 个事件，8 个链接，一个 500 字符的错误消息），ID 为可打印 ASCII 字符且任何地方无控制字符；不符合此条件的批次将被整体拒绝。页面的跨度 ID 为 `browser:<its host's incarnation>:<n>`，服务器仅接受其 ID、追踪和父节点携带调用页面自身前缀的跨度：页面既不能写入其他发送者的追踪，也不能冒充服务器跨度，存储单元也从不替换已存储的跨度。页面每分钟最多可发送 1,200 个跨度；超过此数量则被丢弃并计入 `ingest` 的回答中。服务器为每个跨度标记 `chord.host: browser` 并用其链接来源的身份主体盖章，丢弃页面自身的 `chord.*` 属性：页面关于自身的说法毫无意义。

该追踪仅持有点击，别无他物。服务于多个请求的工作即为其自身的追踪，**链接**到首次需要它的跨度：工作进程的启动（`chord.resolve`），对象的激活（`chord.activate`）。链接是另一个追踪中跨度的引用；渲染器打印它且从不跟随它。

### 陷阱：一个追踪始于一个跨度开始的地方

面板在点击（`useHold`）之前引用了 `Todos`，在 `BACKGROUND_CONTEXT` 下，该持有在页面中没有任何痕迹：**一个没有记录跨度的上下文记录 nothing。** 宿主不给这样的作业自己的跨度，所以它的调用不开始 `chord.call` 且不发送载体，且提供者的 `chord.method` 成为其自身的追踪的根（服务器的页面持有的中继是：`chord.relay` 与 `member="hold"`）。所以如果服务器的调用在步骤 2：在每个工作进程上它都是一个根 `chord.method`。开始一个跨度，工作开始，一个点击处理程序或一个作业（`host.root(fn, context, { trace })`），并传递其 `ctx` 向下。套件中的 HTTP 组件为每个请求执行此操作：一个 `chord.http` 跨度（方法、路由模式、状态），带有路由的 `ctx` 在其下。步骤 11 从命令行添加蜂蜜，正如步骤 3 所做的那样；请求是追踪的根，服务器对工作进程的调用在其下：

```
11. adding honey from the command line:
   chord.http chord.host="server" method="POST" route="POST /api/lists/:list" status=200
     chord.hold chord.host="server" type="todo.todos" object="todo.todos(groceries)" address.kind="list" outcome="bound"
     chord.call chord.host="server" type="todo.todos" member="add" object="todo.todos(groceries)" address.kind="list" chord.principal="owner" outcome="ok"
       chord.method chord.host="list|groceries" type="todo.todos" member="add" object="todo.todos(groceries)" principal.kind="owner" chord.principal="owner" outcome="ok"
         chord.method chord.host="list|groceries" type="todo.store" member="add" object="todo.store" principal.kind="worker" chord.principal="worker:list|groceries" outcome="ok"
           · chord.commit documents=0 ops=0 rows=1 duration=<ms>
           todos.validate chord.host="list|groceries" chars=5
```

### 您自己的一个时间跨度

商店的 `add` 检查其自身范围内的文本：

```
				await ctx.trace("todos.validate", { chars: text.length }, async () => {
					// Refused as Chord refuses arguments that fail their schema: the caller gets `invalid/args`.
					if (text === "") throw invalid("args", "a todo needs text");
				});
```

`ctx.trace(name, attributes, fn)` 启动方法范围的子范围，并为 `fn` 创建一个具有相同 principal 和 `get` 的子范围 `ctx`，因此在此范围内发出的调用将隶属于它，并将其宿主命名为与 Chord 的范围相同。它是上述两条追踪的最后一行：`todos.validate chord.host="list|groceries" chars=5`。其属性是小的扁平值；todo 的文本是数据且不在遥测中，其长度足够。当 `fn` 抛出异常时，范围失败且错误按原样传递：调用方仍然获得 `invalid/args`。这样小的检查不需要拥有自己的范围；查找、模型请求或文件读取也是如此。

### 为哪个 span 而设

方法及其调用跨度记录了它们为何而存在：默认情况下为 `chord.principal`，即身份主体的键（其规�� JSON）。应用程序可能通过每个宿主采用的钩子记录其他属性（`principalAttributes`）：待办事项应用程序记录一个简短键，对于用户，则记录其登录时使用的名称：

```
export function principalAttributes(principal: TodoPrincipal): Readonly<Record<string, string>> {
	let attributes = recorded.get(principal);
	if (attributes === undefined) {
		attributes =
			principal.kind === "user"
				? { "chord.principal": `user:${principal.id}`, "user.name": principal.name }
				: { "chord.principal": principal.kind === "worker" ? `worker:${principal.list}` : principal.kind };
		recorded.set(principal, attributes);
	}
	return attributes;
}
```

Chord 为每个记录的主键调用钩子，因此它必须是同步且廉价的；任何缓存都是钩子自己的（这里是一个 `WeakMap` 按主键：一个链接在主键对象的所有调用上盖一个印章）。`startTodo` 将其交给服务器（`principalAttributes` 旁边 `telemetry`）以及工作进程的入口到 `startWorker`（上方）：每个宿主都有自己的，相同的函数。服务器也为页面的跨度盖一个印章，用于链接的主键批次传输过来：`page.add` 上方是拥有者的。只有钩子返回的内容被记录，从不记录其他应用数据，属性是供拥有者看的：存储，就像下面的查看器一样，是拥有者的。

### 阅读

该存储通过 `TelemetryStore` 读取，pi-telemetry 的读取接口模仿 Jaeger：`findTraces`（完整追踪，按最新顺序，根据第一个 span 的名称、属性、错误或时间窗口过滤，通过游标分页），`getTrace(id)` 和 `findMetrics`（一个指标的行，每个系列一行，10 秒，包含计数、总和、最小值和最大值）。`trace.ts` 是一个数据目录脚本，无论应用是否运行。它使用 Chord 的只读存储 `openTelemetryStore`（`@earendil-works/chord/telemetry-sqlite/node`），该实现仅包含读取接口并打开 SQLite 只读模式：它从不写入服务器写入的文件，从不创建或迁移该文件，并拒绝没有存储或存储版本不同的文件。

```
// The shared file, read-only: the app's processes may be writing it meanwhile.
const store = await openTelemetryStore(telemetryFile(dataDir));
try {
	if (id !== undefined) {
		const trace = await store.getTrace(id);
		console.log(trace === undefined ? `no trace ${id}` : renderTrace(trace.spans));
	} else {
		// Complete traces, newest first; a trace's first span (its root, by start time) names it, and the app's
		// `principalAttributes` hook named whom each span was for.
		const attributes = values.principal === undefined ? undefined : { "chord.principal": values.principal };
		const { traces } = await store.findTraces({ name: values.name, attributes, limit: 10 });
		for (const { traceId, spans } of traces) {
			const start = Math.min(...spans.map((span) => span.startedMs));
			const length = Math.max(...spans.map((span) => span.endedMs)) - start;
			const failed = spans.some((span) => span.status.status === "error") ? "  error" : "";
			console.log(`${traceId}  ${spans[0]!.name}  ${spans.length} spans  ${length.toFixed(2)} ms${failed}`);
		}
		// Metrics are stored per series and 10 s: a series is one set of attributes, here the direction.
		for (const direction of ["sent", "received"]) {
			const rows = await store.findMetrics({ name: "chord.link.bytes", attributes: { direction } });
			const bytes = rows.reduce((sum, row) => sum + row.sum, 0);
			const messages = rows.reduce((sum, row) => sum + row.count, 0);
			console.log(`chord.link.bytes ${direction}: ${bytes} bytes in ${messages} messages`);
		}
	}
```

`screenshots.ts` 在应用运行时，由一个独立的进程运行两次：最新的 `page.add` 为所有者生成一个 span，然后是其 id 被拒绝的点击（ids、长度和计数已屏蔽）：

```
trace.ts <data directory> --name page.add --principal owner:
   <id>  page.add  6 spans  <ms>  error
   <id>  page.add  6 spans  <ms>
   chord.link.bytes sent: <n> bytes in <n> messages
   chord.link.bytes received: <n> bytes in <n> messages
trace.ts <data directory> <the refused one's id>:
   page.add chord.host="browser" chars=0 chord.principal="owner" error trace_failed: a todo needs text
     chord.call chord.host="browser" type="todo.todos" member="add" object="todo.todos(groceries)" address.kind="list" outcome="error" chord.principal="owner" error call_failed: a todo needs text
       chord.relay chord.host="server" type="todo.todos" member="#0" upstream.kind="list" outcome="error" error relay_failed: a todo needs text
         chord.method chord.host="list|groceries" type="todo.todos" member="add" object="todo.todos(groceries)" principal.kind="owner" chord.principal="owner" outcome="error" error method_failed: a todo needs text
           chord.method chord.host="list|groceries" type="todo.store" member="add" object="todo.store" principal.kind="worker" chord.principal="worker:list|groceries" outcome="error" error method_failed: a todo needs text
             todos.validate chord.host="list|groceries" chars=0 error trace_failed: a todo needs text
```

追踪的 id 是其根跨度的 id。拒绝的添加在其路径上的每个跨度上都失败了，每个跨度都有一个固定的类别（`trace_failed`, `call_failed`, `relay_failed`, `method_failed`），消息在 500 个字符处被截断；取消的调用不是错误。它没有提交：没有写入内容。`chord.link.bytes` 计算了服务器和工作进程的链接中的编码消息数量（一个页面不发送指标），每个消息发送时计算一次，接收时再计算一次。指标不携带 id，只携带值很少的属性（`direction`），因此其系列保持很少。指标的当前 10 秒仅在超过该时长或进程关闭时发送和写入一次。

### 一个追踪查看器

两半构成了查看器：在服务器上读取存储的路由，以及显示其答案的页面。应用程序的服务器代码要么是启动脚本，要么是插件，而路由来自插件（第三章），因此路由是一个小插件，`plugins/traces/`。其服务器面服务一个宿主生命周期对象，该对象以只读方式打开存储的文件，并在其存活期间提供路由：

```
	host.serve(Traces, (_params, self) => {
		const info = self.get(AppInfo, {});
		// The server records to the same file with its own recorder; this object only reads it.
		const telemetry = self.open(
			async () => openTelemetryStore(telemetryFile(info.dataDir.value)),
			(store) => store.close(),
		);
		self.contribute(Http, {
			routes: {
				"GET /api/traces": async (request, ctx) => {
					ownerOnly(ctx);
					// `?principal=user:u1`: the traces with a span for that principal (`principalAttributes`).
					const principal = request.query.get("principal");
					const attributes = principal === null ? undefined : { "chord.principal": principal };
					const { traces } = await telemetry.value.findTraces({ limit: RECENT, attributes });
					return { kind: "json", value: traces.map(({ traceId, spans }) => summaryOf(traceId, spans)) };
				},
				"GET /api/traces/:id": async (request, ctx) => {
					ownerOnly(ctx);
					const trace = await telemetry.value.getTrace(request.params.id!);
					if (trace === undefined) return { kind: "error", status: 404 };
					return { kind: "json", value: { id: trace.traceId, spans: rowsOf(clamped(trace.spans)) } };
				},
			},
		});
		return {};
	});
```

该套件向插件提供服务器进程值作为对象：`AppInfo` 命名数据目录，API 的服务器模块通过 `telemetryFile` 和 Chord 的 `openTelemetryStore` 重新导出它。`self.open` 声明一个原生对象，仅在 `AppInfo` 存活时打开，当对象结束时关闭；插件的重新加载会重启该对象并重新打开文件。对象在其宿主（`local`，空参数；决策 R21）存活期间可贡献。Spans 命名谁调用了什么，因此只有所有者可以读取它们：

```
function ownerOnly(ctx: Ctx): void {
	if (ctx.principal?.kind !== "owner") throw forbidden("the traces are the owner's");
}
```

身份主体 `undefined` 仅在公开路由上有效，而这些路由并非如此；已登录成员获得 403。列表根据追踪的归属者（`?principal=`）进行过滤，并显示每个追踪及其用户姓名或身份主体键。`rowsOf` 按树状结构对追踪的跨度进行排序，父节点先于子节点，每行的宿主为其跨度的 `chord.host`，并将属性、事件和链接转换为文本。页面的跨度携带页面的时钟，该时钟可能与服务器的时钟相差数秒，因此页面的跨度可能显示在服务器跨度之后开始。在绘制之前，`clamped` 将追踪的页面跨度移动最小距离，使得每个服务器跨度都位于其对应的页面跨度内部（`plugins/traces/clock.ts`，带有其自身的测试）：

```
export function clamped(spans: readonly StoredSpan[]): readonly StoredSpan[] {
	const skew = skewOf(spans);
	if (skew === 0) return spans;
	return spans.map((span) =>
		hostOf(span) === "browser" ? { ...span, startedMs: span.startedMs + skew, endedMs: span.endedMs + skew } : span,
	);
}
```

该页面是应用在第二个路径上的页面：`start.ts` 在 `/` 和 `/traces` 处提供它（第一章），而网页入口则根据路径选择查看者：

```
	// The page's one other path (`start.ts` serves the page at `/` and `/traces`).
	if (location.pathname === "/traces") return h("main", { className: "wide" }, h("h1", null, "Todo"), h(TraceViewer));
```

`TraceViewer` (`src/web/traces.ts`) 获取 URL 片段中选择的列表和追踪，并在追踪的时间线上绘制每个 span：包含其宿主名称、以条形图表示的长度、属性及事件、主题中错误颜色的失败 span，以及每个引用其 id 但从未获取的链接。这些条形图通过内联样式定位，由于 React 通过 DOM 设置它们，因此页面的 CSP 允许这样做：

```
	const bar = {
		left: `${(row.offsetMs / total) * 100}%`,
		width: `${Math.max((row.durationMs / total) * 100, 0.5)}%`,
	};
```

在两次点击后，`screenshots.ts` 打开了蜂蜜的追踪查看器（图中的“来自浏览器的一个追踪”), 则将类型 `user:u1` 输入到其过滤器中：

```
	await browser.open(`${origin}/traces#${encodeURIComponent(trace.traceId)}`);
	await browser.wait("document.querySelectorAll('.traces-tree li').length === 6");
	await shoot("traces.png");
```

```
wrote docs/tutorial/screens/traces.png
the viewer shows page.add: 6 spans on 3 hosts
wrote docs/tutorial/screens/traces-ada.png
the viewer for user:u1 lists chord.method for ada
```

![按身份主体筛选的追踪查看器](http://47.93.196.89/content/images/chord-tutorial/traces-ada.png)

图 8. Ada（的`user:u1`): 咖啡 `screenshots.ts` 在开头添加她（作为服务器的根，由列表的工作进程记录），应用名为她的钩子 `ada`.

该列表主要是应用程序自身的工作：每个 HTTP 请求、中继持有和激活都是其自身的痕迹，正如上面的陷阱所说。真正的查看器也会按名称（`findTraces({ name })`）或错误（`errors: true`）进行过滤，读取接口都支持这两项；本例涉及三个文件中的330行TypeScript和20个 CSS，位于 Chord 的只读存储中。

XII

## 测试一个插件

testPlugin 像生产环境一样构建插件，并单独加载到其自身的宿主上。

上述步骤测试了整个应用程序。大多数规则在一个插件上检查得更快。`testPlugin` 构建插件时采用生产构建模式（相同的 esbuild 调用，检查 API 的导入），使用应用程序的真实 API 模块对其进行评估，并在带有内存存储的新宿主上加载它，地址位于列表处，以便它可以服务 `Todos`：

```
	let t: TestedPlugin<v1.TodoPrincipal>;
	beforeEach(async () => {
		t = await testPlugin<v1.TodoPrincipal>(
			fileURLToPath(new URL("./", import.meta.url)),
			{
				kind: "worker",
				api: { name: "@todo/app", versions: ["v1"] },
				modules: { "@todo/app/v1": v1 },
				address: v1.List({ id: "groceries" }),
				principal: { kind: "worker", list: "list|groceries" },
			},
			context,
		);
	});
	afterEach(() => t.close(context));
```

每个测试都以其选择的身份主体运行代码，`t.root`：

```
	it("lets the owner clear the done todos", async () => {
		await t.root(
			async (ctx) => {
				const todos = await ctx.get(Todos, { list: "groceries" });
				if (todos.variant !== "owner") throw new Error(`the owner got ${todos.variant}`);
				await todos.add("milk", ctx);
				await todos.add("eggs", ctx);
				await todos.tick({ id: 1, done: true }, ctx);
				await todos.clear(ctx);
				expect(todos.items.value.map(({ text }) => text)).toEqual(["eggs"]);
				expect(todos.notice.value).toEqual({ namespace: "todo.lists", key: "cleared", params: ["owner"] });
			},
			context,
			{ principal: OWNER },
		);
		expect(t.unserved()).toEqual([]);
```

该插件使用的服务以及测试服务未绑定的服务会等待未绑定；`t.unserved()` 列出了这些名称，因此测试知道要添加哪些 stub 面。它贡献给无人服务的注册表会被收集（`t.contributions(R)`）。像任何测试一样运行它：

```
node ../../node_modules/vitest/dist/cli.js --run test/examples/61-tutorial/plugins/lists/worker.test.ts
```

该测试通过包名称导入 API，并通过插件自身的 `Todos` 导入 `contract.ts`，正如 facets 所做的那样：

```
import * as v1 from "@todo/app/v1";
```

```
import { Todos } from "./contract.ts";
```

两者均解析，因为 API 是一个工作区包 npm 链接（第一章）；插件的构建捆绑了其自身的 `contract.ts` 副本，并且两者绑定，因为合约是数据（相同的名称、版本和模式）。第三个测试检查了第二章的拒绝：

```
				const refused = await todos.add("", ctx).catch((error: ChordError) => error);
				expect(refused).toMatchObject({ code: "invalid", reason: "args" });
```

XIII

## 编写 p2 插件

p2 有何不同，以及一个小型插件：一个用于代理的工具和一个为大众设计的芯片。

p2，即带有网页和终端的编码代理服务器，运行于 Chord v2 但不在 app kit 上。它比 kit 更旧，并保留自己的决策：自己的数据目录和 SQLite 文件，自己的登录、设备和共享，自己的页面和终端捆绑包。它直接使用 Chord 的组件：`createHost` 用于 p2 的身份主体，`createWorkers` 用于每个会话的一个工作进程，`createPlugins` 用于查找、构建、监听和部署其插件。插件作者看到的是相同的模型：合约、工厂、注册表和每种宿主类型的 facet。

|             | 待办事项应用（套件）                                    | p2                                                                    |
| ----------- | --------------------------------------------- | --------------------------------------------------------------------- |
| 宿主种类        | Server, List({ id })                          | Server, Session({ id }) (serverHost, sessionHost)                     |
| 身份主体        | 服务器，工作进程，所有者，用户                               | owner (带 access), guest (带 sessions 和 revision), worker, server, host |
| API         | @todo/app/v1, /server, /web, /tui             | @p2/app/v1 (浏览器安全), /node (服务器和工作进程), /server, /worker, /web, /tui    |
| 其他宿主包       | React 用于 Web                                  | pi-ai, pi-durable, pi-schema, pi-env, pi-tui, ws, yaml: 插件直接导入它们      |
| 插件          | plugins/<name>/, package.json 名为 @todo/<name> | apps/p2/plugins/<name>/, package.json 名为 <name>                       |
| 一个 Chord 存储 | 该套件按宿主：持久化文档和表                                | none: 持久化数据是 Durable 的（带有对话的文档）或 p2 自己的文件                             |
| 页面和终端       | kit 表面，Panels 和 Screens                       | p2 的包；页面的 Slots 注册表，useObject，useReadable                             |
| 应用更改        | app.applyChanges                              | 所有者的 Reload 按钮，npm run reload 在 apps/p2 中，reload\_plugins 工具          |

### 草稿板插件

示例为代理提供了 `scratch_note` 工具，并向所有可能查看会话的人展示了会话的笔记。其合约规定每个会话在其工作进程上放置一个对象：

```
export const Scratchpad = remote("p2.scratchpad", {
	version: 1,
	params: { session: param.str() },
	hostAddress: sessionHost,
	variants: { reader: { state: { notes: arr(str()) } } },
});
```

工作进程 facet 在跨工作进程存储中持有笔记（一个带有空参数的本地对象，如第二章所述）。它通过 p2 的 `Extensions` 注册表向代理提供工具，该注册表是 Durable 扩展的键值注册表，并向 `canView` 会话的任何人提供视图：

```
const Notes = defineState("p2.scratchpad.notes", {
	schema: obj({ items: arr(str()) }),
	version: 1,
	initial: () => ({ items: [] }),
});
```

```
export default defineFacet((host) => {
	host.serve(ScratchpadStore, (_params, self) => {
		const notes = self.state(Notes);
		self.contribute(Extensions, {
			scratchpad: defineExtension({
				name: "scratchpad",
				tools: [
					defineTool({
						name: "scratch_note",
						description:
							"Keep a short note for this session (a decision, an open question). Everyone viewing the session sees the notes.",
						parameters: obj({ text: str({ minLength: 1, maxLength: 500 }) }),
						replay: "unsafe",
						execute: async ({ text }, _api, context) => {
							await notes.change((draft) => {
								draft.items.push(text);
							}, context);
							return {
								output: [{ type: "text" as const, text: `Noted (${notes.value.items.length} in all).` }],
							};
						},
					}),
				],
			}),
		});
		return { notes: self.derive([notes], ([value]) => value.items) };
	});
	host.serve(Scratchpad, ({ session }, principal, self) => {
		if (!canView(principal, session)) throw forbidden("viewers only");
		const store = self.get(ScratchpadStore, {});
		return { variant: "reader", members: { notes: self.derive([store.notes], ([notes]) => notes) } };
	});
});
```

笔记是一个内存文档：p2 不打开 Chord 存储，因此 `persist: true` 不可用，笔记的持续时间与会话的工作进程一致。必须超出其生命周期的数据会存入 Durable 文档（p2 的聊天模式正是通过这种方式保持其开关）。该工具在 Durable 中运行，并带有 Durable `Context`，这是 Chord 自身的上下文类型，因此存储句柄会接收它，并且写入会被等待。页面通过 `Slots` 在作曲家的条上获得一个芯片：

```
function NotesChip() {
	const { session } = useRoute();
	const pad = useObject(Scratchpad, session === undefined ? undefined : { session });
	const notes = useReadable(pad?.notes) ?? [];
	if (notes.length === 0) return null;
	return (
		<span className="chip" title={notes.join("\n")}>
			Notes · {notes.length}
		</span>
	);
}

export default defineFacet((_host, self) => self.contribute(Slots, { "composer:scratchpad": NotesChip }));
```

### 在 p2 中测试它

p2 的测试在裸宿主上加载 facets，使用 `testHost` (`test/p2/helpers.ts`)，按需 stub，并通过带有 `fixture.client(principal)` 的内存链接作为任何 principal 到达宿主：

```
	it("keeps the agent's notes and shows them to the session's viewers", async () => {
		const extensions = await fixture.host.registry(Extensions, BACKGROUND_CONTEXT);
		const tool = extensions.value.get("scratchpad")!.tools![0]!;
		await tool.execute({ text: "ship on friday" }, undefined as never, BACKGROUND_CONTEXT);
		const pad = await fixture.client(OWNER).get(Scratchpad, { session: "s1" }, BACKGROUND_CONTEXT);
		assert.deepEqual(await until(pad.notes, (notes) => notes?.length === 1), ["ship on friday"]);
	});
```

从 `apps/p2` 运行 `node --import ./src/source.ts --test test/p2/scratchpad.test.ts`。将插件加载到运行中的 p2 中，使用所有者的 Reload 按钮或 `npm run reload -- scratchpad`；新的插件目录会被相同的 reload 拾取。第二章的陷阱适用于 p2 的名称：一个必须到达会话的工作进程的服务器方法，因为服务器运行在服务器自己的根（`serverOf(self)().root`），而不是通过其调用方的 `ctx.get`。

∴

## 陷阱与开放性问题

六次陷阱再次出现，以及在编写本文时 Chord v2 中难以解释的内容。

**陷阱。**使用前提：宿主在绑定前不绑定任何内容`load`; 一个没有需要加载内容的客户端调用 `load([])`. 引用充当创建它的任何人： `ctx.get` 作为调用方， `host.get` 和对象的本地对象 `self.get`作为宿主。等待你启动的内容：一个使用上下文启动的写入，该上下文结束即被取消并报告`transaction_cancelled`.监听器从当前值开始：读取 `handle.value`，然后 `start`；监听器仅获取变更。普通的 `t` 没有自己的语言：组件和屏幕使用 `useMessages(t)` 和 `scope.messages(t)` 进行渲染。追踪从 span 开始：在没有记录 span 的上下文中工作的内容不会记录任何内容，并且其提供者的 span 是其自身的根。

**在编写示例时发现，并在 Chord 中修复。**整数表键被类型化为 `string`（从 TypeBox 切换的遗留问题），因此 `tx.update(items, { id }, ...)` 与数字一起失败 `tsc`；存储现在使用它。重新加载的 Web 插件丢失了自己的 React 状态（现在是 `usePageState`，第九章），插件的 `contract.ts` 无法在构建外部导入（API 现在是工作区包，第一章），插件无法抛出 Chord 的 `invalid`（Chord 导出 `invalid` 和 `forbidden`，第二章），并且对表的实时窗口需要一次性的 `self.open` 读取，这导致存储在其插件每次重新加载时重启（现在是 `self.query`，Chord 运行的实时查询，第二章和第九章）。

**在撰写遥测章节时发现，并在 Chord 中已修复。** `chord.commit`仅计算文档及其操作，因此写入表行的提交报告了 `documents=0 ops=0`（现在它计算 `rows`）。跨度未指定宿主，查看器通过 SQLite 存储的 id 格式区分进程（每个 Chord 跨度现在携带 `chord.host`）。存储将跨度的开始时间保留为整毫秒，因此查看器的条形图偏移最多一（开始时间现在是微秒，存储版本 2，记录器将其文件迁移至此版本）。读取需要一个不记录任何内容的记录器（`openTelemetryStore`读取只读）。页面无法记录，因此章节的点击是 Node 中的客户端宿主，代表页面（`startPage({ telemetry: true })`现在，其跨度发送给服务器）。每个进程写入该文件本身，每个工作进程告知其路径（服务器现在是存储的唯一写入者，工作进程和页面向其发送跨度），且跨度未说明它们为谁服务（`principalAttributes`）。

A

## 《应用的文件》

| 文件                                        | 种类           | 它是什么                                                                                       |
| ----------------------------------------- | ------------ | ------------------------------------------------------------------------------------------ |
| src/principals.ts                         | 基础           | 身份主体类型，principalAttributes（该 span 所指向的对象）                                                  |
| package.json                              | 基础           | API 作为工作区包 @todo/app，其 exports                                                             |
| src/api/v1/index.ts                       | 基础，每种        | @todo/app/v1: Chord 的部分（包含 invalid、forbidden、defineMessages、Translations），defineFacet、List |
| src/api/v1/server.ts                      | 基础，服务器       | Http，临时 login，AppInfo，telemetryFile，openTelemetryStore                                     |
| src/api/v1/web.ts                         | 基础，Web       | Panels, useHold, useHost, useMessages, usePageState, useWatch                              |
| src/api/v1/tui.ts                         | 基础，tui       | Screens                                                                                    |
| start.ts, worker.ts                       | 基础           | startServer 与遥测存储，startWorker，每个都包含principalAttributes                                     |
| src/telemetry.ts                          | 基础           | 数据目录中的遥测存储文件，openTelemetry（服务器的）                                                           |
| src/web/main.ts, src/web/style.css        | 基础，Web 入口    | startPage (录制中), 面板，/traces 处的查看者                                                          |
| src/web/traces.ts                         | 基础，Web 入口    | 所有者的追踪查看器                                                                                  |
| src/tui/main.ts                           | 基础，终端入口      | startTerminalRuntime，屏幕                                                                    |
| plugins/lists/                            | 插件           | 合同 (Item, Notice, Todos), 消息 (英语，德语), 工作进程 (表，存储，视图), 服务器 (路由), 网页 (面板), tui (屏幕), 它的测试    |
| plugins/french/                           | 插件           | 用于页面和终端的列表插件的法语                                                                            |
| plugins/login.server.ts                   | 单文件插件        | 该套件的临时登录                                                                                   |
| plugins/traces/                           | 插件           | trace viewer 的路由仅经过遥测存储（服务器），所有者的；页面的时钟移动到了服务器的（clock.ts，其测试）                              |
| later/harbor/                             | 插件，在步骤 6 中添加 | 页面的 Harbor 主题和终端                                                                           |
| main.ts, screenshots.ts, tutorial.test.ts | 示例           | 步骤，截图（含页面追踪点击），检查                                                                          |
| trace.ts                                  | 示例           | 一个数据目录的最新追踪，一次追踪，一个指标                                                                      |

B

## 引用

| 调用                                                                                                                                                                                           | 章           | 它的作用                     |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- | ------------------------ |
| startServer<P>({ name, dataDir, root, listen, api, kinds, principal, workers, web, tui, onError }, ctx)                                                                                      | I           | 该应用的服务器，附带套件             |
| startWorker({ api, kinds, principalAttributes, onError })                                                                                                                                    | 我，XI        | 一个工作进程的入口                |
| app.ownerLink(), app.ownerTokenFile                                                                                                                                                          | I, III      | 所有者的登录，用于浏览器和本地客户端       |
| hostKind(name, params), remote(name, { version, params, hostAddress, variants }), local(name, { version, params, members })                                                                  | II          | 宿主种类和契约                  |
| host.serve(Type, factory); 工厂 self: table, query, derive, tx, get, contribute                                                                                                                | II          | 对象的组成部分                  |
| host.root(fn, ctx, { principal }), ctx.get(Type, params)                                                                                                                                     | II          | 代码在对象之外，作为身份主体           |
| invalid(reason, message, details?), forbidden(message, details?)                                                                                                                             | II, IX      | Chord 的错误，由插件抛出          |
| self.contribute(Http, { routes })                                                                                                                                                            | III         | 路由，默认拒绝                  |
| startPage, HostProvider, useHold, useWatch, useRegistry                                                                                                                                      | IV          | 页面                       |
| startTerminalRuntime, TuiScope, startTerminal                                                                                                                                                | V           | 终端                       |
| usePageState(key, initial), scope.pageState(key, initial)                                                                                                                                    | IV, IX      | 说明该页面在插件重载期间保持状态         |
| self.contribute(Themes, theme)                                                                                                                                                               | VI          | 一个主题                     |
| ephemeralLoginFacet<P>(), host.load(\[\], ctx), host.connect(link, { principal }, ctx)                                                                                                       | VII         | 一个第二用户的客户端               |
| defineMessages(namespace, catalogs), t.translations, self.contribute(Translations, list)                                                                                                     | IV, V, VIII | 一个插件的消息及其目录              |
| useMessages(t), scope.messages(t), t.message(key, ...args), t.render(message), MessageSchema                                                                                                 | IV, V, VIII | 按查看者的语言显示文本，将消息作为数据      |
| MessagesOf<typeof t>, LocaleChoice                                                                                                                                                           | VIII        | 一个翻译插件，用户的语言             |
| app.plugins.state, app.applyChanges(ctx)                                                                                                                                                     | VI, IX      | 标记与展开                    |
| app.exportData(address, path, ctx), app.importData(path, address, ctx)                                                                                                                       | X           | 移动工作进程的数据                |
| createSqliteTelemetry(db), startServer({ telemetry, principalAttributes }), startPage({ telemetry: true }), createHost({ telemetry, principalAttributes })                                   | XI          | 服务器上的遥测存储，页面和工作进程向其发送其跨度 |
| host.root(fn, ctx, { trace }), ctx.trace(name, attributes, fn)                                                                                                                               | XI          | 您自己的跨度                   |
| openTelemetryStore(path), store.findTraces({ name, attributes, errors, startMin, startMax, limit, cursor }), store.getTrace(id), store.findMetrics({ name, attributes }), renderTrace(spans) | XI          | 读取追踪和指标                  |
| AppInfo, self.open(open, close)                                                                                                                                                              | XI          | 数据目录，原生（查看者的存储）          |
| testPlugin(dir, { kind, api, modules, address, principal, stubs }, ctx)                                                                                                                      | XII         | 一个插件单独                   |

Together · 第一版 · 2026-11-03 · Chord v2 切片 1-13，应用套件，示例 61
