React 和 Typescript（上）
======

本文大部分内容翻译至[React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/docs/basic/setup)，并根据个人感受补充、阉割了部分内容，想更详细了解可查看[原文](https://react-typescript-cheatsheet.netlify.app/docs/basic/setup)。《React 和 Typescript（下）》准备整理常见问题及解决方案。

## 开始

### 安装依赖

`devDependencies`：
- `typescript`
- `@types/react`： React 类型定义
- `@types/react-dom`：React DOM 类型定义

`dependencies`：
- `react`
- `react-dom`
- `tslib`：TypeScript 辅助函数库。TypeScript 通过一些辅助函数实现降级（转换为旧版本的 JavaScript），这些辅助函数可能导致大量重复代码，在 `tsconfig.json` 开启 `importHelpers` 选项后辅助函数将从 [tslib](https://www.npmjs.com/package/tslib) 导入。

### tsconfig.json

```
{
  "compilerOptions": {
    /* react 相关 */
    "jsx": "react",                       // 控制 JSX 文件输出方式：将 `JSX` 改为 `React.createElement` 调用并生成 `.js` 文件。
    "lib": ["dom", "es2015"],             // 库文件（声明文件）：至少支持dom 和 es2015特性
    
    /* 其他可选项 */
    "allowSyntheticDefaultImports": true, // 允许合成默认导入：模块没有显式指定默认导出时，允许默认导入（即支持 `import React from "react"`，而不需要 `import * as React from "react"`）。
    "esModuleInterop": true,              // TypeScript 处理 CommonJS/AMD/UMD 模块时将存在一些缺陷，开启 esModuleInterop 可修复这些缺陷
    "experimentalDecorators": true        // 支持装饰器
  }
}
```
更多配置项可查看[tsconfig.json 详解](https://github.com/wjcj/blog/issues/24)。

## Prop Types

``` ts
type AppProps = {
  message: string;
  count: number;
  disabled: boolean;
  names: string[];
  status: "waiting" | "success";
  obj: object; // 任何对象，只要你不使用其属性（不常见但用作占位符）
  obj1: Object; // 除 null、undefined 以外的任意值，即使它不是一个对象
  obj2: {}; // 与 `Object` 完全相同
  obj3: {
    id: string;
    title: string;
  };
  objArr: {
    id: string;
    title: string;
  }[];
  dict1: {
    [key: string]: MyTypeHere;
  };
  dict2: Record<string, MyTypeHere>; // 相当于 dict1
  onSomething: Function; // 任何函数，只要你不调用它（不推荐）
  onClick: () => void;
  onChange: (id: number) => void;
  onClick(event: React.MouseEvent<HTMLButtonElement>): void;
  optional?: OptionalType; // 可选 prop
};

export declare interface AppProps {
  // children1: JSX.Element; // 不支持 array children
  // children2: JSX.Element | JSX.Element[]; // 不支持 string
  // children3: React.ReactChildren; // 错误，不是类型，是一个工具，例如 React.Children.map(children, function[(thisArg)])
  // children4: React.ReactChild[]; // 接受 array children
  children: React.ReactNode; // 最好, 支持所有类型（注意下面👇极端情况）
  functionChildren: (name: string) => React.ReactNode; // 使用函数渲染child
  style?: React.CSSProperties; // style props
  onChange?: React.FormEventHandler<HTMLInputElement>; // form 事件，泛型参数是 event.target 的类型
  //  more info: https://react-typescript-cheatsheet.netlify.app/docs/advanced/patterns_by_usecase/#wrappingmirroring
  props: Props & React.ComponentPropsWithoutRef<"button">; // 模拟 button 所有 props，并明确不转发 ref
  props2: Props & React.ComponentPropsWithRef<MyButtonWithForwardRef>; // 模拟 MyButtonForwardedRef 的所有 props，并明确转发 ref
}
```

**`React.ReactNode` 极端情况**

``` tsx
type Props = {
  children: React.ReactNode;
};
function Comp({ children }: Props) {
  return <div>{children}</div>;
}
function App() {
  return <Comp>{{}}</Comp>; // Runtime Error: Objects not valid as React Child!
}
```
这是因为 `ReactNode` 包含 `ReactFragment`（它允许 `{}`），修复这个会破坏很多库，所以你只需要注意 `ReactNode` 并不是绝对安全。

**`React.ReactNode` vs `JSX.Element`**

有效的 React node 与 `React.createElement` 返回的内容不同。无论组件最终渲染什么，`React.createElement` 始终返回一个对象，即`JSX.Element` 接口，但 `React.ReactNode` 是组件所有可能返回值的集合。即：

- `JSX.Element` -> `React.createElement` 的返回值
- `React.ReactNode` -> 组件的返回值

**`type` or `interface`？**

在创建库或第三方的类型定义时，始终使用 `interface` 作为公共 API 的定义，因为这允许使用者在缺少某些定义时通过声明合并来扩展。
而你的 React 组件 `Props` 和 `State` 考虑使用 `Type` 以保持一致性，因为它受到更多限制。

`type` 对于联合类型（例如`type MyType = TypeA | TypeB`）很有用，而 `interface` 更适合声明数据模型然后可以实现（`implements`）或扩展（`extends`）。

## Components

### Function Components

一个普通函数，函数接受一个 `props` 参数并返回一个 `JSX.Element`。

``` ts
type AppProps = {
  message: string;
}; /* 如果类型定义需要导出，请使用 `interface`，以便用户可以扩展（`extends`） */

// 最简方式：自动推断返回类型
const App = ({ message }: AppProps) => <div>{message}</div>;

// 推荐方式：标注返回类型，以免返回意外的类型而引发错误
const App = ({ message }: AppProps): JSX.Element => <div>{message}</div>;

// 不推荐使用 React.FC，具体原因见👇
const App: React.FC<AppProps> = ({ message}) => <div>{message}</div>;

// props类型声明内联
const App = ({ message }: { message: string }) => <div>{message}</div>;
```

不推荐使用 `React.FC` ?

`React.FC` 即 `React.FunctionComponent` 的简写，它没有明显好处却存在几个主要缺点：
- 提供隐式定义 `props.children`，意味着所有组件都可接受 `children`，但实际可能并不需要；
- 在 `component as namespace pattern`（使用组件作为相关组件（常为子组件）的命名空间）中（如 `<Select.Item />`），使用 `React.FC` 没有意义；
- 不支持泛型；
- 使用 `defaultProps` 无法正常工作。
具体可查看[Remove React.FC from Typescript template](https://github.com/facebook/create-react-app/pull/8177)。

### Class Components

``` tsx
// Props 类型声明
type MyProps = { // 需要导入则推荐使用 `interface`
  message: string;
  // readonly message: string; // readonly 是多余的
};
// State 类型声明
type MyState = {
  count: number;
  // readonly count: number; // readonly 是多余的
};
class App extends React.Component<MyProps, MyState> {
  // 第二个泛型参数 MyState（State 类型声明）可选
  state: MyState = {
    count: 0,
  };
  // 类属性：声明但不要赋值
  pointer: number;
  componentDidMount() {
    this.pointer = 3;
  }
  render() {
    return (
      <div onClick={() => this.increment(1)}>
        {this.props.message} {this.state.count}
      </div>
    );
  }
  // 类方法
  increment = (amt: number) => {
    this.setState((state) => ({
      count: state.count + amt,
    }));
  };
}
```
- `State` 类型声明不是必须的；
- `Props` 和 `State` 中的 `readonly` 修饰符是不多余的，因为 `React.Component<P,S>` 已经将它们标记为不可变的。
- `Class Properties`（类属性）：如果需要声明类属性供以后使用，只需像 `state` 一样声明它，但不要赋值。

**Typing getDerivedStateFromProps**

通过 `getDerivedStateFromProps` 可以从 `props` 获取派生状态，下面是一些定义 `getDerivedStateFromProps` 的方法：

1. 显式定义派生状态并确保 `getDerivedStateFromProps` 返回值符合它

``` ts
class Comp extends React.Component<Props, State> {
  static getDerivedStateFromProps(
    props: Props,
    state: State
  ): Partial<State> | null { // Partial<T>：部分映射类型
    //
  }
}
```

2. 想从 `getDerivedStateFromProps` 的返回值确定 `State` 类型

``` tsx
class Comp extends React.Component<
  Props,
  ReturnType<typeof Comp["getDerivedStateFromProps"]> // ReturnType<T>：获取函数返回值类型。
> {
  static getDerivedStateFromProps(props: Props) {}
}
```

3. 当你的派生状态想要具有其他状态字段和 [memoization](https://zh-hans.reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html#what-about-memoization) 时

``` tsx
type CustomValue = any;
interface Props {
  propA: CustomValue;
}
interface DefinedState {
  otherStateField: string;
}

type State = DefinedState & ReturnType<typeof transformPropsToState>; // 👈看这里
function transformPropsToState(props: Props) {
  return {
    savedPropA: props.propA, // save for memoization
    derivedState: props.propA,
  };
}
class Comp extends React.PureComponent<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = {
      otherStateField: "123",
      ...transformPropsToState(props),
    };
  }
  static getDerivedStateFromProps(props: Props, state: State) {
    if (isEqual(props.propA, state.savedPropA)) return null;
    return transformPropsToState(props);
  }
}
```

## DefaultProps

**你可能不需要 `defaultProps`**。因为根据这条[推文](https://twitter.com/dan_abramov/status/1133878326358171650) `defaultProps` 最终将被弃用，共识是使用对象默认值。

**Function Components**
``` ts
type GreetProps = { age?: number };
const Greet = ({ age = 21 }: GreetProps) => // ...
```

**Class components**
``` tsx
type GreetProps = {
  age?: number;
};

class Greet extends React.Component<GreetProps> {
  render() {
    const { age = 21 } = this.props;
    /*...*/
  }
}

let el = <Greet age={3} />;
```
### `defaultProps` 类型

`defaultProps` 类型推断在 [TypeScript 3.0+](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-0.html) 中得到了很大改进，尽管[一些极端情况仍然存在问题](https://github.com/typescript-cheatsheets/react-typescript-cheatsheet/issues/61)。

**Function Components**

``` ts
// 使用 typeof 更加方便，请注意它会变量提升
type GreetProps = { age: number } & typeof defaultProps;

const defaultProps = {
  age: 21,
};

const Greet = (props: GreetProps) => {
  // ...
};
Greet.defaultProps = defaultProps;
```

**明确定义 `DefaultProps` 类型**

在某些情况下，我们可能希望 `defaultProps` 有多种类型。这种方式可以灵活地分配两种类型。此外它还会检查 `defaultProp` 的类型验证，[例如👇](https://github.com/typescript-cheatsheets/react/issues/415#issuecomment-841223219)

``` tsx
const GreetComponent = ({ name, age, status }: RequiredProps & DefaultProps) => (
  <div>{`Hello, my name is ${name}, ${age}, ${typeof status === string ? status : status[0]}`}</div>
);

const defaultProps = {
  age: 25,
  status: ""
} as DefaultProps;

GreetComponent.defaultProps = defaultProps;

type RequiredProps = {
  name: string;
}

type DefaultProps = {
 age: number,
 status: string | string[]
}
```

**Class components**

``` ts
type GreetProps = typeof Greet.defaultProps & { age: number };

class Greet extends React.Component<GreetProps> {
  static defaultProps = {
    age: 21,
  };
  // ...
}

let el = <Greet age={3} />;
```
**JSX.LibraryManagedAttributes**

上面的实现对于应用程序来说是很好的方式，但是有时你希望能够 `export GreetProps` 以便其他人可以使用。这里的问题在于 `GreetProps` 的定义方式，`age` 是一个必需的属性，而不是 `defaultProps`。

你可以为了 `export` 专门创建单独的类型，也可以使用 `JSX.LibraryManagedAttributes`（提取必需和可选的 Props）：

``` tsx
// 内部约定，不应该导出
type GreetProps = {
  age: number;
};

class Greet extends Component<GreetProps> {
  static defaultProps = { age: 21 };
};

// 对外约定
export type ApparentGreetProps = JSX.LibraryManagedAttributes<
  typeof Greet,
  GreetProps
>;
```

### 使用 `defaultProps` 消费组件的 Props

问题：
``` tsx
interface IProps {
  name: string;
}
const defaultProps = {
  age: 25,
};
const GreetComponent = ({ name, age }: IProps & typeof defaultProps) => (
  <div>{`Hello, my name is ${name}, ${age}`}</div>
);
GreetComponent.defaultProps = defaultProps;

// React.ComponentProps<T>：获取组件 T 的 props 类型
const TestComponent = (props: React.ComponentProps<typeof GreetComponent>) => {
  return <h1 />;
};

// Error: Property 'age' is missing in type '{ name: string; }' but required in type '{ age: number; }'
const el = <TestComponent name="foo" />;
```

解决方案：定义一个应用了 `LibraryManagedAttributes` 的工具👇

``` tsx
// type ComponentType<P = {}> = ComponentClass<P> | FunctionComponent<P>;
// infer：表示在 extends 条件语句中待推断的类型变量
type ComponentProps<T> = T extends
  | React.ComponentType<infer P>
  | React.Component<infer P>
  ? JSX.LibraryManagedAttributes<T, P>
  : never;

const TestComponent = (props: ComponentProps<typeof GreetComponent>) => {
  return <h1 />;
};

// No error
const el = <TestComponent name="foo" />;
```

## Events

``` tsx
type State = {
  text: string;
};
class App extends React.Component<Props, State> {
  state = {
    text: "",
  }
  // 方式1：推断类型
  onChange = (e: React.FormEvent<HTMLInputElement>): void => {
    this.setState({ text: e.currentTarget.value });
  }
  onClick(e: React.MouseEvent<HTMLButtonElement | HTMLAnchorElement>) {
    event.preventDefault();
    console.log(event.currentTarget.tagName);
  }
  // 方式2：强制使用 @types/react 提供的委托类型
  onChange: React.ChangeEventHandler<HTMLInputElement> = (e) => {
    this.setState({text: e.currentTarget.value})
  }

  render() {
    return (
      <div>
        <button onClick={this.onClick}>Click me</button>
        <input type="text" value={this.state.text} onChange={this.onChange} />
      </div>
    );
  }
}
```

- 如果你不关心事件的类型，可以使用 `React.SyntheticEvent`；
- 如果你不在意事件的元素，可以省略泛型参数，比如：`onClick(e: React.MouseEvent) { ... }`。

### 事件类型列表

- `AnimationEvent`: CSS 动画事件。
- `ChangeEvent`: 当用户更改 `<input>`， `<select>` 和 `<textarea>` 元素的值并提交这个更改时触发的事件。
- `ClipboardEvent`: 使用复制、粘贴和剪切事件。
- `CompositionEvent`: 由于用户间接输入文本而发生的事件（例如用户使用拼音输入法开始输入汉字时，这个事件就会被触发）。
- `DragEvent`: 与指针设备（例如鼠标）的拖放交互。
- `FocusEvent`:	当元素获得或失去焦点时发生的事件。
- `FormEvent`: 每当表单或表单元素获得/失去焦点、表单元素值更改或表单提交时发生的事件。
- `InvalidEvent`: 当输入的有效性限制失败时触发的事件（例如 `<input type="number" max="10">`，但有人输入数字 20）。
- `KeyboardEvent`: 键盘事件。
- `MouseEvent`:	用户与指针设备(如鼠标)交互时发生的事件。
- `PointerEvent`: 用于在定点输入设备（鼠标、触控笔和单点或多点的手指触摸）上交互所触发的事件。
- `TouchEvent`: 用户与触摸设备交互而发生的事件。
- `TransitionEvent`: CSS 过渡事件。
- `WheelEvent`: 用户滚动鼠标滚轮或类似输入设备时触发的事件。
- `SyntheticEvent`: 以上所有事件的基础事件，当不确定事件类型时应该使用。

**关于 `InputEvent`**

Typescript 不支持 `InputEvent`，因为这是一个实验中的功能，并不是所有浏览器都完全支持，并且在不同的浏览器中可能表现不同。你可以改用 `KeyboardEvent`。

## Hooks

### useState

``` ts
// 1. 使用类型推断，对简单的值来说非常有效
const [val, toggle] = React.useState(false);
// 使用依赖推断复杂类型: https://react-typescript-cheatsheet.netlify.app/docs/basic/troubleshooting/types/#using-inferred-types

// 2. 用空值进行默认值初始化，显式声明类型 + 联合类型
const [user, setUser] = React.useState<IUser | null>(null);
// later...
setUser(newUser);

// 3. 如果状态在设置后不久初始化并且始终具有该值，则还可以使用类型断言：
const [user, setUser] = React.useState<IUser>({} as IUser);
// later...
setUser(newUser);
```

### useReducer

``` tsx
const initialState = { count: 0 };

type ACTIONTYPE =
  | { type: "increment"; payload: number }
  | { type: "decrement"; payload: string };

function reducer(state: typeof initialState, action: ACTIONTYPE) {
  switch (action.type) {
    case "increment":
      return { count: state.count + action.payload };
    case "decrement":
      return { count: state.count - Number(action.payload) };
    default:
      throw new Error();
  }
}

function Counter() {
  const [state, dispatch] = React.useReducer(reducer, initialState);
  return (
    <>
      Count: {state.count}
      <button onClick={() => dispatch({ type: "decrement", payload: "5" })}>
        -
      </button>
      <button onClick={() => dispatch({ type: "increment", payload: 5 })}>
        +
      </button>
    </>
  );
}
```

### useEffect

确保返回函数或 `undefined`。

``` tsx
// 确保返回函数或 `undefined`。
useEffect(() => {
  const subscription = props.source.subscribe();
  return () => {
    subscription.unsubscribe();
  };
});
```
*`useLayoutEffect` 会在所有的 DOM 变更之后同步调用 effect，使用方式与 `useEffect` 基本一致。*

### useRef

**访问 DOM**

``` jsx
function Foo() {
  // 如果可能的话请尽可能具体（例如具体性：HTMLDivElement > HTMLElement > Element），将返回 React.RefObject<HTMLDivElement>
  const divRef = useRef<HTMLDivElement>(null);
  // const divRef = useRef<HTMLDivElement>(null!); // 如果你确定 divRef.current 永远不会为空，那么也可以使用非空断言运算符(`!`)

  useEffect(() => {
    // ref.current 可能为空。因为绑定 ref 的元素渲染时有条件地，也可能你忘记绑定 ref
    if (!divRef.current) throw Error("divRef is not assigned");

    doSomethingWith(divRef.current);
  });

  return <div ref={divRef}>...</div>;
}
```
**保存可变值**

``` tsx
function Foo() {
  // 返回 React.MutableRefObject<number | null>
  const intervalRef = useRef<number | null>(null);

  useEffect(() => {
    intervalRef.current = setInterval(...);
    return () => clearInterval(intervalRef.current);
  }, []);

  return <button onClick={/* clearInterval the ref */}>Cancel timer</button>;
}
```

### useImperativeHandle

``` tsx
type ListProps<ItemType> = {
  items: ItemType[];
  innerRef?: React.Ref<{ scrollToItem(item: ItemType): void }>;
};

function List<ItemType>(props: ListProps<ItemType>) {
  useImperativeHandle(props.innerRef, () => ({
    scrollToItem() {},
  }));
  return null;
}
```

### useMemo & useCallback

``` tsx
function getHistogram(image: ImageData): number[] {
  // ...
  return histogram;
}
function Histogram() {
  // ...
  const histogram = useMemo(() => getHistogram(imageData), [imageData]);
}
```

### useCallback

``` tsx
const memoCallback = useCallback((a: number) => {
  // doSomething
}, [a]);
```

## Context

``` tsx
interface AppContextInterface {
  name: string;
  author: string;
  url: string;
}

const AppCtx = React.createContext<AppContextInterface | null>(null);

const sampleAppContext: AppContextInterface = {
  name: "Using React Context in a Typescript App",
  author: "thehappybug",
  url: "http://www.example.com",
};

export const App = () => (
  <AppCtx.Provider value={sampleAppContext}>...</AppCtx.Provider>
);

// 使用
export const PostInfo = () => {
  const appContext = React.useContext(AppCtx);
  return (
    <div>
      // Error 👇: Object is possibly 'null'
      Name: {appContext.name}, Author: {appContext.author}, Url:{" "}
      {appContext.url}
    </div>
  );
};
```

``` tsx
interface ContextState {
  name: string | null;
}
// 使用空对象（`{}`）作为 `React.createContext` 的默认值
const Context = React.createContext({} as ContextState);
```

**创建一个没有 `defaultValue` 且不需要检查 `undefined` 的 `createCtx`**

``` tsx
// 需要显示声明类型参数，因为没有默认值可以推断：      👇👇👇👇👇👇👇👇👇👇
const currentUserContext = React.createContext<string | undefined>(undefined);

function EnthusasticGreeting() {
  const currentUser = React.useContext(currentUserContext);
  return <div>HELLO {currentUser!.toUpperCase()}!</div>;
  // 告知ts currentUser 一定有值：👆（非空断言）
}

function App() {
  return (
    <currentUserContext.Provider value="Anders">
      <EnthusasticGreeting />
    </currentUserContext.Provider>
  );
}
```

上面的写法略显多余，因为随后就在 `Provider` 设置了有效的 `content`。有几个解决方案：

1. 使用非空断言
``` tsx
//                                                              👇
const currentUserContext = React.createContext<string>(undefined!);
```
简单但不太安全，你有可能忘记向 `Provider` 提供值。

2. 编写一个名为 `createCtx` 的辅助函数，以防止访问未提供值的 `Context`。这样不需要提供 `defaultValue`，也不需要检查 `undefined`

``` tsx
function createCtx<A extends {} | null>() {
  const ctx = React.createContext<A | undefined>(undefined);
  function useCtx() {
    const c = React.useContext(ctx);
    if (c === undefined)
      throw new Error("useCtx must be inside a Provider with a value");
    return c;
  }
  return [useCtx, ctx.Provider] as const; // 'as const' 使 TypeScript 推断出一个元组
}

// 使用:
export const [useCurrentUserName, CurrentUserProvider] = createCtx<string>();
function EnthusasticGreeting() {
  const currentUser = useCurrentUserName();
  return <div>HELLO {currentUser.toUpperCase()}!</div>;
}
function App() {
  return (
    <CurrentUserProvider value="Anders">
      <EnthusasticGreeting />
    </CurrentUserProvider>
  );
}
```

## forwardRef/createRef

### createRef
``` ts
class CssThemeProvider extends React.PureComponent<Props> {
  private rootRef = React.createRef<HTMLDivElement>();
  render() {
    return <div ref={this.rootRef}>{this.props.children}</div>;
  }
}
```

``` tsx
type Props = { children: React.ReactNode; type: "submit" | "button" };
export type Ref = HTMLButtonElement;
export const FancyButton = React.forwardRef<Ref, Props>((props, ref) => (
  <button ref={ref} className="MyClassName" type={props.type}>
    {props.children}
  </button>
));
```

从 `forwardRef` 得到的 `ref` 是可变的，所以你可以根据需要指定它。如果你希望使其不可变（即确保没人可以重新指定它），请指定 `React.Ref`：

``` tsx
type Props = { children: React.ReactNode; type: "submit" | "button" };
export type Ref = HTMLButtonElement;
export const FancyButton = React.forwardRef((
  props: Props,
  ref: React.Ref<Ref> // <-- 这里!
) => (
  <button ref={ref} className="MyClassName" type={props.type}>
    {props.children}
  </button>
));
```

如果你想要获取 `forward` 中 `ref` 组件的 props，请使用 [ComponentPropsWithRef<E>](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/a05cc538a42243c632f054e42eab483ebf1560ab/types/react/index.d.ts#L770)。

## 参考
- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/docs/basic/setup)
- [TypeScript 和 React](https://fettblog.eu/typescript-react/getting-started/)
- [10++ TypeScript Pro tips/patterns with (or without) React](https://medium.com/@martin_hotell/10-typescript-pro-tips-patterns-with-or-without-react-5799488d6680)
