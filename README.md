<!-- SPDX-License-Identifier: AGPL-3.0-only -->

# Miroot

最小構成のDockerイメージをビルドするためのBash補助関数群。

## 概要

`miroot`は、バイナリとその実行時依存（共有ライブラリ、シンボリックリンク連鎖、Debianパッケージの著作権情報）を`/out`ディレクトリにコピーするシェル関数を提供します。`/out`は最終的なDockerイメージのルートファイルシステム`/`として使用することを前提としています。

## 依存関係

- `bash`
- `lddtree`（`pax-utils`パッケージ）
- `dpkg`
- GNU coreutils（`realpath`、`readlink`、`cp`、`mkdir`、`chmod`、`chown`、`touch`、`stat`）

## 使い方

```bash
. miroot

# バイナリとその全依存ライブラリを/outへコピー
copy_binary_dependencies /usr/bin/myapp

# /outをルートとした最小Dockerイメージをビルド
# （例: docker build 内でCOPY /out /）
```

### 提供関数

<!-- markdownlint-disable MD013 -->

| 関数                                       | 説明                                                                       |
| ------------------------------------------ | -------------------------------------------------------------------------- |
| `copy_path_with_symlink_chain <path>`      | シンボリックリンク連鎖を解決しながらファイルやディレクトリを`/out`へコピー |
| `copy_dir_with_symlink_chain <path>`       | シンボリックリンク連鎖を解決しながらディレクトリを`/out`へ作成する         |
| `copy_binary_dependencies <path>`          | バイナリ本体とその全依存ライブラリ、ライセンス情報を`/out`へコピー         |
| `copy_package_copyright_for_path <path>`   | パスが属するDebianパッケージのcopyrightを`/out`へコピー                    |
| `ensure_parent_dirs_copied <path>`         | 親ディレクトリが`/out`配下に存在するよう確保                               |
| `resolve_symlink_chain_target <path>`      | シンボリックリンク連鎖を解決し最終的な実体パスを出力                       |
| `create_dir_with_existing_ancestor <path>` | 既存の祖先を探して`/out`配下にディレクトリを作成                           |

<!-- markdownlint-enable MD013 -->

## 設計方針

- 意図しない状態では即座に失敗し、呼び出し元にエラーを伝播させる
- merged-usr（`/lib` → `/usr/lib`）のシンボリックリンク構造に対応
- Debianパッケージのcopyright取得は`dpkg -S`の終了ステータスが成功した場合のみ
  所有者行として扱う。所有者行が取得できない場合（diversionや
  no path foundを含む）は実体パスに1回フォールバックし、
  それでも失敗すればビルドを止める
- 各コマンドの失敗は明示的に`|| return 1`で伝播し、`set -e`が無効化されるコンテキストでも安全
