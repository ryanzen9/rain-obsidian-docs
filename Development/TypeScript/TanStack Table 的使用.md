## 无头样式库（Headless UI Library）是什么

“无头”指的是**只提供逻辑与行为，不提供任何预设的 UI 外观（CSS）和固定的 DOM 结构**。通俗来说，它给你一套“大脑”和“骨架”，但“长相”完全由你决定。这种设计让开发者可以自由地将组件接入任意样式体系，而不必受制于框架自带的模板。目前比较流行的组件库如 Tailwind CSS、shadcn 等，都倾向于与传统的 Ant Design 等“重 UI”组件库不同，将组件的行为与样式彻底分离。这也意味着更低的耦合度、更高的自定义能力，同时也更适合与 AI 协同编程——AI 可以更精准地控制逻辑，而样式交给用户自定义，极大提升了开发效率与灵活性。

## TanStack Table

TanStack Table 是一款典型的**无头（Headless）表格库**，延续了“只提供逻辑与行为，不预设 UI 外观”的设计理念。

它提供了完整的表格状态管理、排序、筛选、分页、行选择、列固定、虚拟滚动等复杂功能，但**不输出任何 HTML 结构或 CSS**。开发者可以基于它的 API 和 Hooks，使用任何 UI 框架（React、Vue、Solid、Svelte 等）或纯 HTML 来构建自己的表格外观。

### 核心特点

- **框架无关**：官方支持 React、Vue、Solid、Svelte、Lit 等，核心逻辑与视图层解耦。
- **受控与非受控模式**：可以完全掌控表格状态，也可以交给库内部管理。
- **强大的功能组合**：排序、过滤、分组、分页、列排序、行展开、虚拟化等均可组合使用。
- **TypeScript 友好**：提供完整类型推导，提升开发体验。
- **与 Tailwind CSS、shadcn 等无头/样式无关生态配合良好**，适合需要深度自定义表格样式的项目。

由于它不限制 JSX 结构和 CSS，开发者可以轻松地将它接入现有的设计系统或 Tailwind CSS，也能更方便地与 AI 协同——AI 负责逻辑与状态管理，开发者专注于视觉呈现。

> 下文以 TanStack Table V9 为例（V7, v8 改动较大）。

```tsx
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  getFilteredRowModel,
  getPaginationRowModel,
  flexRender,
  type ColumnDef,
  type SortingState,
} from '@tanstack/react-table';

import { useState } from 'react';

type Person = {
  id: number;
  name: string;
  age: number;
  email: string;
};

const columns: ColumnDef<Person>[] = [
  {
    accessorKey: 'name',
    header: '姓名',
    cell: (info) => info.getValue(),
  },
  {
    accessorKey: 'age',
    header: '年龄',
    cell: (info) => info.getValue(),
  },
  {
    accessorKey: 'email',
    header: '邮箱',
    cell: (info) => info.getValue(),
  },
];

const data: Person[] = [
  { id: 1, name: '张三', age: 28, email: 'zhangsan@example.com' },
  { id: 2, name: '李四', age: 34, email: 'lisi@example.com' },
  { id: 3, name: '王五', age: 22, email: 'wangwu@example.com' },
];

export function TanStackTableExample() {
  const [tableData] = useState(data);
  const [sorting, setSorting] = useState<SortingState>([]);

  const table = useReactTable({
    data: tableData,
    columns,
    state: {
      sorting,
    },
    onSortingChange: setSorting,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
  });

  return (
    <div>
      <table>
        <thead>
          {table.getHeaderGroups().map((headerGroup) => (
            <tr key={headerGroup.id}>
              {headerGroup.headers.map((header) => (
                <th key={header.id}>
                  {header.isPlaceholder ? null : (
                    <button
                      onClick={header.column.getToggleSortingHandler()}
                      className={header.column.getCanSort() ? 'cursor-pointer select-none' : ''}
                    >
                      {flexRender(header.column.columnDef.header, header.getContext())}
                      {{
                        asc: ' 🔼',
                        desc: ' 🔽',
                      }[header.column.getIsSorted() as string] ?? null}
                    </button>
                  )}
                </th>
              ))}
            </tr>
          ))}
        </thead>
        <tbody>
          {table.getRowModel().rows.map((row) => (
            <tr key={row.id}>
              {row.getVisibleCells().map((cell) => (
                <td key={cell.id}>
                  {flexRender(cell.column.columnDef.cell, cell.getContext())}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>

      <div className="pagination">
        <button onClick={() => table.previousPage()} disabled={!table.getCanPreviousPage()}>
          上一页
        </button>
        <button onClick={() => table.nextPage()} disabled={!table.getCanNextPage()}>
          下一页
        </button>
        <span>
          第 {table.getState().pagination.pageIndex + 1} / {table.getPageCount()} 页
        </span>
      </div>
    </div>
  );
}
```

### Data, Features, Columns

一个最简单的表:

```tsx
import {
  useTable,
  stockFeatures,
} from "@tanstack/react-table"

const table = useTable({
  features: stockFeatures,
  data,
  columns,
})

```

一个表由三部分核心配置驱动：**Data、Features、Columns**。这三者各司其职，组合在一起才构成一个完整的表格实例——`data` 提供原始数据，`columns` 描述列的映射与渲染方式，`features` 则声明需要启用的交互能力。它们之间没有任何样式绑定，因此你可以完全控制最终渲染出的 `<table>` 结构。

**Data**

`data` 就是表格要展示的数据数组，可以是任意类型的对象数组。它的结构不需要与列一一对应，列可以通过 `accessorKey` 或 `accessorFn` 从数据对象中提取值。比如前面例子中的 `Person`，`accessorKey: 'name'` 表示从 `{ name: '张三' }` 中取出 `name` 字段。TanStack Table 本身不修改数据，也不要求数据必须扁平——你可以传入深层嵌套对象，再通过 `accessorFn` 自定义取值逻辑。

**Features**

`features` 是 TanStack Table 的“功能开关”。在 V9 中，`stockFeatures` 包含所有内置功能（排序、分页、筛选、分组、列调整、行选择等），但你也可以按需组合，避免打包体积臃肿。比如只想要排序和分页，可以显式传入 `[sorting, pagination]`。每个 feature 都对应一套独立的状态和 row/column 模型扩展，例如 `getSortedRowModel()` 和 `getPaginationRowModel()` 就是这些功能在 React 适配层中的具体实现。

**Columns**

`columns` 描述了表格的列结构，常见的类型有：
- **`accessorKey`**：基于数据字段的简单映射，适合大多数场景。
- **`accessorFn`**：自定义函数，接收整行数据并返回单元格值，适合复杂计算。
- **`header`**：列头显示内容，可以是字符串，也可以是一个函数返回动态 JSX。
- **`cell`**：单元格渲染逻辑，默认会取 `accessor` 的值，但你完全可以根据 `info.row.original` 做自定义渲染，比如格式化日期、插入按钮或状态徽章。
- **`enableSorting` / `enableFiltering`**：列级别的功能开关，覆盖全局 features 的设置。
`columns` 还支持**分组**（grouping）、**固定列**（pinned）、**聚合**（aggregation）等高级配置，这些都由 features 提供能力支撑。

当你把 `data`、`columns`、`features` 传递给 `useTable` 后，它会返回一个 `table` 实例，封装了所有状态和操作方法，比如我们之前用到的 `table.getHeaderGroups()`、`table.getRowModel()`、`table.nextPage()`。接下来要做的，就是把这些方法返回的模型对象（如 header groups、rows、cells）手动映射到你的 JSX 结构中——这正是“无头”的体现：渲染结构由你掌控，但状态和逻辑已经由 Table 内部管理好了。

### 示例

下面以一套 **User 表格**为例，演示从数据模型到列定义的完整写法。为了贯彻"逻辑与 UI 分离"的思路，示例将定义拆分为两个文件：

- `users/_components/user-table/data.ts` —— 数据模型与示例数据
- `users/_components/user-table/column.ts` —— 列定义
- `users/_components/user-table/table.tsx` —— 负责利用 `useTable` 组装数据和列，并渲染出最终的表格 UI。
#### 1. 数据模型（data.ts）

```typescript
export interface User {
  id: string
  name: string
  age: number
  email: string
  status: "active" | "disabled"
  createdAt: string
}

export const users: User[] = [
  {
    id: "1",
    name: "张三",
    age: 28,
    email: "zhangsan@example.com",
    status: "active",
    createdAt: "2026-08-01",
  },
  // ...
]
```

`User` 是表格要展示的数据对象，`users` 数组即 `useTable` 的 `data` 入参。

#### 2. 列定义（column.ts）

TanStack Table 提供三种列类型，分别适用于不同场景：

| 类型 | 定义方式 | 说明 |
| --- | --- | --- |
| **Accessor 列** | `accessorKey` / `accessorFn` | 从数据对象中提取字段值，用于展示原始数据（如姓名、年龄） |
| **Display 列** | `display` | 不绑定任何字段，仅用于自定义渲染（如操作按钮、勾选框） |
| **Group 列** | `group` | 将多个子列聚合到同一表头下（如多级表头） |

> **Tip**：尽量让 accessor 保持**原始、有语义**的数据值（如时间戳、状态码），再通过 `cell` 转换成 Badge、货币、日期等 UI 表现。这样排序、过滤以及服务端查询的逻辑都更稳定，UI 变化也不会污染数据层。

> 列定义详见：<https://tanstack.com/table/latest/docs/guide/column-defs>；`features` 在 V9 中指通过 `tableFeatures` 引入的能力对象。

最简单的写法是直接声明 `ColumnDef` 数组：

```tsx
import type { ColumnDef } from "@tanstack/react-table"

const columns: ColumnDef<typeof features, User>[] = [
  // Accessor 列：直接映射字段
  {
    accessorKey: "name",
    header: "姓名",
  },
  {
    accessorKey: "age",
    header: "年龄",
  },
  // Display 列：渲染操作按钮
  {
    id: "actions",
    header: "操作",
    cell: ({ row }) => {
      const user = row.original

      return (
        <Button onClick={() => editUser(user.id)}>
          编辑
        </Button>
      )
    },
  },
]
```

> 示例中的 `Button`、`editUser` 为项目自定义组件与函数，可按实际 UI 替换。

#### 3. 使用 createColumnHelper（推荐）

直接手写对象的方式类型推导不够精确。V9 中推荐使用 `createColumnHelper` 来定义列，它用 `accessor` / `display` / `group` 三个方法分别对应上述三种列类型，并带来完整的 TypeScript 类型推导：

```tsx
import { createColumnHelper } from "@tanstack/react-table"

const columnHelper = createColumnHelper<typeof features, User>()

const columns = columnHelper.columns([
  // Accessor 列
  columnHelper.accessor("name", {
    header: "姓名",
    cell: (info) => info.getValue(),
  }),

  columnHelper.accessor("age", {
    header: "年龄",
    cell: (info) => `${info.getValue()} 岁`,
  }),

  // Display 列
  columnHelper.display({
    id: "actions",
    header: "操作",
    cell: ({ row }) => (
      <Button>
        编辑 {row.original.name}
      </Button>
    ),
  }),

  // Group 列：多级表头
  columnHelper.group({
    header: "用户信息",
    columns: [
      columnHelper.accessor("name", {
        header: "姓名",
      }),
      columnHelper.accessor("age", {
        header: "年龄",
      }),
    ],
  }),
])
```

`columnHelper.accessor("name", ...)` 中的 `"name"` 会被推断为 `User` 的字段名，拼错字段或传错类型时 TypeScript 会直接报错；`cell` 回调中的 `info.getValue()` 也会自动推导出对应字段的类型（如 `name` 为 `string`、`age` 为 `number`），无需手动标注。

#### 4. 使用 useTable

最后在页面组件中把前面定义好的三部分组装起来，调用 `useTable` 生成 `table` 实例，再传给负责渲染的 UI 组件：

```tsx
"use client"

import { useTable, stockFeatures } from "@tanstack/react-table"
import { users, User } from "./data"
import { columns } from "./column"

export function Users() {
  const table = useTable({
    features: stockFeatures,
    data: users,
    columns,
  })

  return (
    <div className="overflow-x-auto">
      <table className="min-w-full divide-y divide-gray-200" table={table}>
        {/* 表头、表体、分页渲染逻辑与前文一致，此处省略 */}
      </table>
    </div>
  )
}
```
