# 第 03 章：TypeScript 深度指南

> 目标：从类型基础到高级类型编程，掌握在 Vue 项目中写出类型安全代码所需的全部 TypeScript 知识。

---

## 3.1 为什么使用 TypeScript

| 特性 | JavaScript | TypeScript |
|------|-----------|-----------|
| 类型检查 | 运行时 | **编译时** |
| IDE 提示 | 部分 | **完整** |
| 重构安全 | 危险 | **安全** |
| 文档价值 | 无 | **类型即文档** |
| 与 C# 类比 | — | 和 C# 类型系统高度相似 |

```typescript
// JavaScript：运行时才发现错误
function add(a, b) { return a + b; }
add(1, '2'); // 运行结果是 "12" —— 不是你想要的

// TypeScript：编译时就报错
function add(a: number, b: number): number { return a + b; }
add(1, '2'); // ❌ 编译错误：Argument of type 'string' is not assignable to 'number'
```

---

## 3.2 基础类型

```typescript
// 原始类型
let name: string = 'Alice';
let age: number = 25;
let active: boolean = true;
let data: null = null;
let undef: undefined = undefined;
let sym: symbol = Symbol('id');
let big: bigint = 9007199254740991n;

// 数组
let nums: number[] = [1, 2, 3];
let strs: Array<string> = ['a', 'b'];

// 元组（固定长度和类型的数组）
let pair: [string, number] = ['Alice', 25];
let point: [x: number, y: number] = [0, 0]; // 命名元组

// 对象
let user: { name: string; age: number; email?: string } = {
  name: 'Alice',
  age: 25,
};

// any（尽量避免）
let anything: any = 'hello';
anything = 42; // 不报错，但失去类型保护

// unknown（比 any 安全，使用前必须收窄类型）
let input: unknown = getExternalData();
if (typeof input === 'string') {
  console.log(input.toUpperCase()); // 现在可以
}

// never（不可能有值，如 throw 函数）
function throwError(msg: string): never {
  throw new Error(msg);
}

// void（函数无返回值）
function log(msg: string): void {
  console.log(msg);
}
```

---

## 3.3 接口与类型别名

```typescript
// interface（可扩展，推荐定义对象形状）
interface User {
  id: number;
  name: string;
  email?: string;        // 可选属性
  readonly createdAt: Date; // 只读属性
}

// 扩展接口（类似 C# 接口继承）
interface AdminUser extends User {
  role: 'admin' | 'superadmin';
  permissions: string[];
}

// 合并声明（同名 interface 自动合并）
interface Window {
  myGlobalVar: string; // 扩展全局类型
}

// type alias（更灵活，可以是任何类型）
type ID = string | number;
type Status = 'active' | 'inactive' | 'pending';
type Nullable<T> = T | null;

// type vs interface 选择原则：
// - 定义对象/类的形状 → 优先 interface
// - 联合类型、交叉类型、工具类型 → 用 type
```

### 与 C# 类比

```csharp
// C# 接口
public interface IUser {
    int Id { get; }
    string Name { get; set; }
    string? Email { get; set; }
}

// TypeScript 接口（几乎一样）
interface IUser {
  id: number;
  name: string;
  email?: string;
}
```

---

## 3.4 联合类型与交叉类型

```typescript
// 联合类型（OR）
type StringOrNumber = string | number;
type ApiStatus = 'success' | 'error' | 'loading';

function formatId(id: string | number): string {
  return id.toString();
}

// 交叉类型（AND，合并多个类型）
type WithTimestamp = { createdAt: Date; updatedAt: Date };
type UserWithTimestamp = User & WithTimestamp;

// 辨别联合（Discriminated Union）
// 与 C# 的 sealed class 层次结构类似
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'rectangle'; width: number; height: number }
  | { kind: 'triangle'; base: number; height: number };

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':
      return Math.PI * shape.radius ** 2;
    case 'rectangle':
      return shape.width * shape.height;
    case 'triangle':
      return (shape.base * shape.height) / 2;
  }
}
```

---

## 3.5 泛型（Generics）

```typescript
// 基础泛型（与 C# 泛型语法几乎相同）
function identity<T>(value: T): T {
  return value;
}

// C#: T Identity<T>(T value) => value;
// TS:  function identity<T>(value: T): T { return value; }

// 泛型接口
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
  timestamp: string;
}

// 使用
type UserResponse = ApiResponse<User>;
type ListResponse = ApiResponse<{ items: User[]; total: number }>;

// 泛型约束
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const name = getProperty(user, 'name'); // ✅ string
const xxx = getProperty(user, 'xxx');   // ❌ 编译错误

// 泛型类
class Repository<T extends { id: number }> {
  private items: T[] = [];

  add(item: T): void {
    this.items.push(item);
  }

  findById(id: number): T | undefined {
    return this.items.find(item => item.id === id);
  }

  getAll(): T[] {
    return [...this.items];
  }
}

const userRepo = new Repository<User>();
userRepo.add({ id: 1, name: 'Alice' });
```

---

## 3.6 工具类型（Utility Types）

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// Partial<T>：所有属性变可选
type UpdateUserDto = Partial<User>;
// { id?: number; name?: string; email?: string; ... }

// Required<T>：所有属性变必需
type RequiredUser = Required<Partial<User>>;

// Pick<T, K>：选取部分属性
type UserProfile = Pick<User, 'id' | 'name' | 'email'>;
// { id: number; name: string; email: string }

// Omit<T, K>：排除部分属性
type PublicUser = Omit<User, 'password'>;
// { id: number; name: string; email: string; createdAt: Date }

// Record<K, V>：键值映射
type UserMap = Record<number, User>;
// { [key: number]: User }

type RoleConfig = Record<'admin' | 'user' | 'guest', string[]>;

// Readonly<T>：所有属性变只读
type ImmutableUser = Readonly<User>;

// ReturnType<T>：获取函数返回类型
function createUser() { return { id: 1, name: 'Alice' }; }
type CreatedUser = ReturnType<typeof createUser>; // { id: number; name: string }

// Parameters<T>：获取函数参数类型
type CreateParams = Parameters<typeof createUser>; // []

// Awaited<T>：解包 Promise 类型
type ResolvedUser = Awaited<Promise<User>>; // User

// NonNullable<T>：排除 null 和 undefined
type DefiniteString = NonNullable<string | null | undefined>; // string

// Extract<T, U> / Exclude<T, U>
type OnlyStrings = Extract<string | number | boolean, string>; // string
type NoStrings = Exclude<string | number | boolean, string>;   // number | boolean
```

---

## 3.7 类型守卫（Type Guards）

```typescript
// typeof 守卫
function process(value: string | number) {
  if (typeof value === 'string') {
    return value.toUpperCase();
  }
  return value.toFixed(2);
}

// instanceof 守卫
function handleError(error: unknown) {
  if (error instanceof Error) {
    console.log(error.message);
  } else if (typeof error === 'string') {
    console.log(error);
  }
}

// in 运算符守卫
interface Cat { meow(): void }
interface Dog { bark(): void }

function makeSound(animal: Cat | Dog) {
  if ('meow' in animal) {
    animal.meow();
  } else {
    animal.bark();
  }
}

// 自定义类型谓词（User-defined Type Guard）
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value
  );
}

// 断言函数
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== 'string') {
    throw new TypeError('Expected string');
  }
}
```

---

## 3.8 类（Classes）

```typescript
// TypeScript 类与 C# 类高度相似
class Animal {
  // 访问修饰符
  public name: string;
  protected age: number;
  private _secret: string;
  readonly id: number;

  // 构造函数简写（自动创建并赋值属性）
  constructor(
    public species: string,
    private weight: number,
  ) {
    this.name = '';
    this.age = 0;
    this._secret = '';
    this.id = Math.random();
  }

  // Getter / Setter（与 C# 属性类似）
  get secret(): string {
    return '***';
  }

  set secret(value: string) {
    if (value.length < 6) throw new Error('Too short');
    this._secret = value;
  }

  // 静态方法
  static create(species: string): Animal {
    return new Animal(species, 0);
  }

  // 抽象方法（抽象类中）
}

// 继承
class Dog extends Animal {
  constructor(weight: number) {
    super('Canis lupus familiaris', weight);
  }

  bark(): void {
    console.log(`${this.name} says: Woof!`);
  }
}

// 实现接口
interface Serializable {
  serialize(): string;
  deserialize(data: string): void;
}

class UserModel implements Serializable {
  serialize(): string {
    return JSON.stringify(this);
  }
  deserialize(data: string): void {
    Object.assign(this, JSON.parse(data));
  }
}
```

---

## 3.9 装饰器（Decorators）

```typescript
// 启用：tsconfig.json 中 "experimentalDecorators": true

// 类装饰器（与 C# [Attribute] 类似）
function Injectable(target: Function) {
  Reflect.defineMetadata('injectable', true, target);
}

// 属性装饰器
function Validate(min: number, max: number) {
  return function (target: any, propertyKey: string) {
    let value: number;
    Object.defineProperty(target, propertyKey, {
      get: () => value,
      set: (v: number) => {
        if (v < min || v > max) throw new RangeError(`${propertyKey} 超出范围`);
        value = v;
      },
    });
  };
}

// 方法装饰器（常用于 AOP 日志、权限检查）
function Log(target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function (...args: any[]) {
    console.log(`调用 ${key}，参数:`, args);
    const result = original.apply(this, args);
    console.log(`${key} 返回:`, result);
    return result;
  };
  return descriptor;
}

class UserService {
  @Log
  getUser(id: number): User {
    return { id, name: 'Alice', email: '' };
  }
}
```

---

## 3.10 高级类型技巧

```typescript
// 条件类型
type IsString<T> = T extends string ? 'yes' : 'no';
type A = IsString<string>; // 'yes'
type B = IsString<number>; // 'no'

// 映射类型
type Optional<T> = {
  [K in keyof T]?: T[K];
};

// 模板字面量类型
type EventName = 'click' | 'focus' | 'blur';
type EventHandler = `on${Capitalize<EventName>}`;
// 'onClick' | 'onFocus' | 'onBlur'

// 递归类型
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

// infer：类型推断
type UnpackArray<T> = T extends (infer Item)[] ? Item : T;
type A2 = UnpackArray<string[]>; // string
type B2 = UnpackArray<number>;   // number

// 实用：提取函数返回类型（手动实现 ReturnType）
type MyReturnType<T extends (...args: any) => any> =
  T extends (...args: any) => infer R ? R : never;
```

---

## 3.11 Vue 3 中的 TypeScript 实践

```typescript
// 组件 Props 类型
interface Props {
  title: string;
  count?: number;
  items: string[];
  onUpdate: (value: string) => void;
}

// defineProps with TypeScript
const props = defineProps<Props>();

// withDefaults
const props2 = withDefaults(defineProps<Props>(), {
  count: 0,
  items: () => [],
});

// defineEmits
const emit = defineEmits<{
  update: [value: string];
  delete: [id: number];
  change: [oldVal: string, newVal: string];
}>();

// ref 类型
const count = ref<number>(0);
const user = ref<User | null>(null);

// reactive 类型
interface State {
  users: User[];
  loading: boolean;
  error: string | null;
}
const state = reactive<State>({
  users: [],
  loading: false,
  error: null,
});

// computed 类型推断
const activeUsers = computed(() =>
  state.users.filter(u => u.active)
);
// 自动推断为 ComputedRef<User[]>
```

---

## 3.12 常见错误与排错

```typescript
// ❌ 错误 1：不必要的 any
function processData(data: any) { ... }
// ✅ 使用 unknown 或泛型
function processData<T>(data: T) { ... }
function processData(data: unknown) { ... }

// ❌ 错误 2：非空断言滥用
const el = document.getElementById('app')!; // 确信不为 null 才用 !
// ✅ 先判断
const el = document.getElementById('app');
if (el) { el.textContent = '...'; }

// ❌ 错误 3：类型断言过于武断
const user = data as User; // 可能不安全
// ✅ 配合类型守卫
if (isUser(data)) { const user = data; }

// ❌ 错误 4：忽视 strictNullChecks
// ✅ tsconfig.json 中始终保持 "strict": true
```

---

## 3.13 章节小结

| 概念 | 优先级 | C# 类比 |
|------|--------|---------|
| 基础类型 + 接口 | ⭐⭐⭐⭐⭐ | 类似 C# 类型 + interface |
| 联合类型 + 辨别联合 | ⭐⭐⭐⭐⭐ | 类似 C# 模式匹配 |
| 泛型 | ⭐⭐⭐⭐⭐ | 与 C# 泛型几乎相同 |
| 工具类型 | ⭐⭐⭐⭐ | 类似 C# LINQ + 反射 |
| 类型守卫 | ⭐⭐⭐⭐ | 类似 C# is/as + 模式匹配 |
| 高级类型 | ⭐⭐⭐ | 超过 C# 类型系统 |

> **下一步**：阅读 [第 04 章 - Vue 3 核心](04_vue3_core.md)
