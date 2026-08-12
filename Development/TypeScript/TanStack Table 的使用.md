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

一个表 