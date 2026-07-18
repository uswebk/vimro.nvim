---
name: add-problem
description: vimro の問題 JSON を作成する。カテゴリ・狙う操作を聞き取り、スキーマどおりの1問1ファイルを作成し、解法キーを headless Neovim で実再生して検証する。「問題を追加して」「◯◯の問題を作って」で使用。
---

# vimro 問題作成スキル

`problems/<category>/NNN-<slug>.json` に 1 問 1 ファイルで問題を追加する。

## 手順

1. **要件の確認**: カテゴリ（`problems/` のサブディレクトリ。新カテゴリも可 — エンジンが自動検出する）、狙う vim 操作、難易度を引数や文脈から決める。不明なら聞く。
2. **連番の決定**: 対象カテゴリ内の既存ファイルを見て次の `NNN` を決める。`id` は `<category>-NNN`。
3. **JSON の作成**: 下記スキーマとガイドラインに従って書く。
4. **検証**: 下記の検証を必ず実行し、通るまで修正する。

## スキーマ

```json
{
  "id": "python-011",
  "category": "python",
  "difficulty": 2,
  "start":  ["行の配列（初期状態）"],
  "goal":   ["行の配列（この状態でクリア）"],
  "cursor": [1, 1],
  "solutions": [
    { "keys": "dw", "optimal": true },
    { "keys": "daw" }
  ],
  "tags": ["dw", "delete"],
  "i18n": {
    "ja": { "title": "...", "description": "...", "hints": ["..."], "notes": ["...", "..."] },
    "en": { "title": "...", "description": "...", "hints": ["..."], "notes": ["...", "..."] }
  }
}
```

制約:

- `cursor` は 1 始まりの `[行, 列]`。`start` の範囲内であること
- `solutions[].keys` は vim 表記（`<Esc>` `<CR>` など）。言語共通・翻訳しない
- `optimal: true` はちょうど 1 つ
- `i18n.ja` と `i18n.en` の両方必須。`notes[i]` は `solutions[i]` の説明で、要素数は solutions と一致させる
- `id` はカテゴリ内で一意

## 良い問題のガイドライン

- **1問につき狙う操作は1つ**。余計な移動や別コマンドが混ざらない最小構成
- **`description` と `goal` を必ず一致させる**（頻出ミス）
- 難易度の目安: 1=単純な削除・挿入（x, dd, dw, A, I）、2=変更系（cw, r, D）、3=テキストオブジェクト・複合（ci", di(, yyp, J）
- カテゴリの題材に合わせる（python なら Python コードらしい編集シチュエーション）
- 判定は「行末空白・最終空行は無視、行内容と行数は厳密」。空白が答えの一部になる問題はこれを踏まえて設計する

## 検証（必須）

スクラッチパッドに一時スクリプトを作り、headless で実行する:

```lua
-- verify.lua（スクラッチパッドに置く。REPO と FILE を差し替える）
local root = "REPO"  -- リポジトリ絶対パス
vim.opt.rtp:prepend(root)
local p = vim.json.decode(table.concat(vim.fn.readfile(root .. "/problems/FILE"), "\n"))
for i, s in ipairs(p.solutions) do
  local buf = vim.api.nvim_create_buf(false, true)
  vim.api.nvim_set_current_buf(buf)
  vim.api.nvim_buf_set_lines(buf, 0, -1, false, p.start)
  vim.api.nvim_win_set_cursor(0, { p.cursor[1], p.cursor[2] - 1 })
  vim.api.nvim_feedkeys(vim.api.nvim_replace_termcodes(s.keys, true, false, true), "x", false)
  vim.api.nvim_feedkeys(vim.api.nvim_replace_termcodes("<Esc>", true, false, true), "x", false)
  local got = vim.api.nvim_buf_get_lines(buf, 0, -1, false)
  assert(require("vimro.engine").is_cleared(got, p.goal),
    ("solution %d (%s) failed: %s"):format(i, s.keys, vim.inspect(got)))
end
print("OK")
```

```sh
nvim --headless -u NONE -l verify.lua
```

これで **全 solutions の keys が start を goal に本当に変形するか**を機械確認する。失敗したら `start` / `goal` / `cursor` / `keys` を見直す。

あわせて次も目視確認する:

- `cursor` が `start` の範囲内か
- `notes` の要素数が `solutions` と一致しているか（ja / en 両方）
- `id` の重複がないか（`grep -r '"id"' problems/<category>/`）
- `description` と `goal` が矛盾していないか
