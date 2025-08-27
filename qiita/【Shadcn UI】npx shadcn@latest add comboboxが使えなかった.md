## はじめに：Shadcn UIとは何か？
[**shadcn/ui**](https://ui.shadcn.com) は、[Radix UI](https://www.radix-ui.com/) のアクセシブルなコンポーネントを [Tailwind CSS](https://tailwindcss.com/) で構成し、React/Next.js プロジェクトに導入できる UI ライブラリです。
特徴は以下の通りです：

- CLI (`npx shadcn add <component>`) を使って必要なコンポーネントを簡単に追加できる
- 各コンポーネントは「ソースコードのコピー」として導入されるため自由にカスタマイズ可能
- 公式が提供するのは **完成品のライブラリではなく、コピー元の設計リソース**

---

## 直面した問題：`npx shadcn add combobox` が動かない
公式サイトには [Combobox のドキュメント](https://ui.shadcn.com/docs/components/combobox) が存在しますが、実際に以下を実行するとエラーになります。

```bash
npx shadcn@latest add combobox
```

```
It may not exist at the registry. Please make sure it is a valid component.
```

これは **CLI に Combobox が登録されていない** ためです。
GitHub Issue などでも同様の報告(投稿はsvelteですが...)が見られます
→[shadcn-svelte issue #1520](https://github.com/huntabyte/shadcn-svelte/issues/1520)

## なぜ動かないのか？
[公式ドキュメント](https://ui.shadcn.com/docs/components/combobox) にもある通り、Combobox は独立したコンポーネントではなく、以下を組み合わせて実装する「Composite Component（複合コンポーネント）」です。

- [`Popover`](https://ui.shadcn.com/docs/components/popover)
- [`Command`](https://ui.shadcn.com/docs/components/command)

そのため CLI では直接インストールできず、自分でファイルを作成してコードを貼り付ける必要があります。

---

## 解決方法：Comboboxを導入する手順

### 1. 初期化
まず Shadcn UI を初期化します。※すでに導入済の場合は不要

```bash
npx shadcn@latest init
```

詳細: [CLI ドキュメント](https://ui.shadcn.com/docs/cli)

---

### 2. 必要なコンポーネントを追加
Combobox 構築に必要な部品を導入します。

```bash
npx shadcn@latest add popover command
```

これで `components/ui/` 配下に `popover.tsx` と `command.tsx` が追加されます。

### 3. 実用例（props対応）
`components/ui/` 配下に以下のファイルを新規作成する。
汎用的に利用するため、元記事のソースコードを少し編集(props対応)している。

```ComboBox.tsx
"use client"

import * as React from "react"
import { Check, ChevronsUpDown } from "lucide-react"
import { Button } from "@/components/ui/button"
import {
  Command,
  CommandEmpty,
  CommandGroup,
  CommandInput,
  CommandItem,
  CommandList,
} from "@/components/ui/command"
import {
  Popover,
  PopoverContent,
  PopoverTrigger,
} from "@/components/ui/popover"

type Option = {
  value: string
  label: string
}

type ComboboxProps = {
  options: Option[]
  placeholder?: string
  emptyText?: string
  onChange?: (value: string) => void
}

export function Combobox({
  options,
  placeholder = "Select option...",
  emptyText = "No results found.",
  onChange,
}: ComboboxProps) {
  const [open, setOpen] = React.useState(false)
  const [value, setValue] = React.useState("")

  return (
    <Popover open={open} onOpenChange={setOpen}>
      <PopoverTrigger asChild>
        <Button variant="outline" role="combobox" aria-expanded={open}>
          {value
            ? options.find((o) => o.value === value)?.label
            : placeholder}
          <ChevronsUpDown className="ml-2 h-4 w-4 opacity-50" />
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-[200px] p-0">
        <Command>
          <CommandInput placeholder="Search..." />
          <CommandEmpty>{emptyText}</CommandEmpty>
          <CommandList>
            <CommandGroup>
              {options.map((o) => (
                <CommandItem
                  key={o.value}
                  value={o.value}
                  onSelect={(val) => {
                    setValue(val)
                    setOpen(false)
                    onChange?.(val)
                  }}
                >
                  {o.label}
                  {value === o.value && <Check className="ml-auto" />}
                </CommandItem>
              ))}
            </CommandGroup>
          </CommandList>
        </Command>
      </PopoverContent>
    </Popover>
  )
}
```

### 4. 呼び出し例

```tsx
<Combobox
  options={[
    { value: "next.js", label: "Next.js" },
    { value: "sveltekit", label: "SvelteKit" },
    { value: "nuxt.js", label: "Nuxt.js" },
  ]}
  placeholder="フレームワークを選択してください"
  onChange={(val) => console.log("選択:", val)}
/>
```

## まとめ
- `npx shadcn add combobox` は執筆時点(2025/08/27)で利用不可
- Combobox は Popover + Command を組み合わせた複合コンポーネント
- 解決策：`npx shadcn add popover command` 実行後→ `Combobox.tsx` ファイルを自分で準備する必要がある
