---
title: react_router路由传参
comments: false
date: 2022-06-03 22:39:21
categories:
  - 前端
tags:
  - react
---

对于`react`来说，在界面的跳转过程中，组件之间的传参是必须的，他们分别是`params传参`,`query/search传参`,`state传参`。

在下面的示例中，传参的方向都是由组件`Message`向组件`Detail`传递。

<!--more-->

## Params 传参

在跳转路由(Link、NavLink)路径后面，以路径的格式传递 params 参数。

在路由展示组件(Route)path 属性中通过`/:xxx`的形式申明要接收的 params 参收。

在路由对应的组件内可通过 `props.match.params`接受传递的参数。

- `Message` 组件

```js
export default class Message extends Component {
	state = {
		messageArr: [
			{ id: "01", title: "消息1" },
			{ id: "02", title: "消息2" },
			{ id: "03", title: "消息3" },
		],
	};
	render() {
		const { messageArr } = this.state;
		return (
			<div>
				<ul>
					{messageArr.map((msgObj) => {
						return (
							<li key={msgObj.id}>
								{/* 向路由组件传递params参数 */}
								<Link to={`/home/message/detail/${msgObj.id}/${msgObj.title}`}>
									{msgObj.title}
								</Link>
							</li>
						);
					})}
				</ul>
				<hr />
				{/* 声明接收params参数 */}
				<Route path="/home/message/detail/:id/:title" component={Detail} />
			</div>
		);
	}
}
```

- `Detail` 组件

```js
const DetailData = [
	{ id: "01", content: "你好，以前" },
	{ id: "02", content: "你好，现在" },
	{ id: "03", content: "你好，未来" },
];

export default class Detail extends Component {
	render() {
		console.log(this.props);
		// 接收params参数
		const { id, title } = this.props.match.params;
		const findResult = DetailData.find((detailObj) => {
			return detailObj.id === id;
		});
		return (
			<ul>
				<li>ID:{id}</li>
				<li>TITLE:{title}</li>
				<li>CONTENT:{findResult.content}</li>
			</ul>
		);
	}
}
```

## query/search 传参

通过在 Link 或者 NavLink 组件中以 urlencode 的方式传递参数，在路由对应的组件内可通过`props.location.search`接收传递的路由参数。

需要注意的是获取到的 search 参数是未解析的，需要用`react` 已经下好的 qs 模块来解析 urlencode 编码字符串。

- `Message` 组件

```js
export default class Message extends Component {
	state = {
		messageArr: [
			{ id: "01", title: "消息1" },
			{ id: "02", title: "消息2" },
			{ id: "03", title: "消息3" },
		],
	};
	render() {
		const { messageArr } = this.state;
		return (
			<div>
				<ul>
					{messageArr.map((msgObj) => {
						return (
							<li key={msgObj.id}>
								{/* 向路由组件传递search参数 */}
								<Link
									to={`/home/message/detail?id=${msgObj.id}&title=${msgObj.title}`}
								>
									{msgObj.title}
								</Link>
							</li>
						);
					})}
				</ul>
				<hr />
				{/* search参数无需声明接收，正常注册路由即可 */}
				<Route path="/home/message/detail" component={Detail} />
			</div>
		);
	}
}
```

- `Detail` 组件

```js
import qs from "querystring";

const DetailData = [
	{ id: "01", content: "你好，以前" },
	{ id: "02", content: "你好，现在" },
	{ id: "03", content: "你好，未来" },
];
export default class Detail extends Component {
	render() {
		console.log(this.props);
		// 接收search参数
		const { search } = this.props.location;
		const { id, title } = qs.parse(search.slice(1));
		const findResult =
			DetailData.find((detailObj) => {
				return detailObj.id === id;
			}) || {};
		return (
			<ul>
				<li>ID:{id}</li>
				<li>TITLE:{title}</li>
				<li>CONTENT:{findResult.content}</li>
			</ul>
		);
	}
}
```

## state 参数

通过在 Link 或者 NavLink 组件中，`to` 传递一个对象，`pathname` 属性为跳转路径，`state` 为传递的路由参数。
通过 `props.location.state` 接收传递的参数。

- `Message` 组件

```js
export default class Message extends Component {
	state = {
		messageArr: [
			{ id: "01", title: "消息1" },
			{ id: "02", title: "消息2" },
			{ id: "03", title: "消息3" },
		],
	};
	render() {
		const { messageArr } = this.state;
		return (
			<div>
				<ul>
					{messageArr.map((msgObj) => {
						return (
							<li key={msgObj.id}>
								{/* 向路由组件传递state参数 */}
								<Link
									to={{
										pathname: "/home/message/detail",
										state: { id: msgObj.id, title: msgObj.title },
									}}
								>
									{msgObj.title}
								</Link>
							</li>
						);
					})}
				</ul>
				<hr />
				{/* state参数无需声明接收，正常注册路由即可 */}
				<Route path="/home/message/detail" component={Detail} />
			</div>
		);
	}
}
```

- `Detail` 组件

```js
const DetailData = [
	{ id: "01", content: "你好，以前" },
	{ id: "02", content: "你好，现在" },
	{ id: "03", content: "你好，未来" },
];
export default class Detail extends Component {
	render() {
		console.log(this.props);
		// 接收state参数
		const { id, title } = this.props.location.state || {};
		const findResult =
			DetailData.find((detailObj) => {
				return detailObj.id === id;
			}) || {};
		return (
			<ul>
				<li>ID:{id}</li>
				<li>TITLE:{title}</li>
				<li>CONTENT:{findResult.content}</li>
			</ul>
		);
	}
}
```
