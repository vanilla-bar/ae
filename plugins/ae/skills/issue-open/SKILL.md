---
name: issue-open
description: GitHub Issue（単一 or 親＋子複数）を作成する。プロジェクト設定に GitHub Project があれば Priority / Estimate / Iteration / 任意の単一選択フィールドも設定する。ユーザーが「issueに出して」「issueに上げて」「issue立てて」等と言った時にも使用する。
argument-hint: '[Issueの説明]'
---

# /issue-open - GitHub Issue 作成 + Project 設定

やることを Issue として積み、プロジェクト設定に GitHub Project があればそのフィールド（Priority / Estimate / Iteration / 任意の単一選択フィールド）を設定する。
関連する複数項目を一度に立てたい場合は、自動で親 issue + 複数の子 issue（sub-issue）として作成する。

**引数:** `/issue-open <Issueの説明>`（省略時は聞く）

## プロジェクト設定の読み込み

リポジトリ直下の `.claude/ae.config.md` があれば読み込む（書式はプラグインの README「プロジェクト設定」を参照）。 使う項目:

- `## GitHub Project`: `{OWNER}` / `{PROJECT_NUMBER}` / `{PROJECT_ID}`、フィールド ID（Priority のオプション ID、Estimate、Iteration、任意の単一選択フィールドとそのオプション ID）
- `## Issue`: `labels`（ラベル候補）、`assignees`（アサイン候補の login）、`template`（Issue テンプレートのパス）

**設定ファイルが無い場合**: ラベルは下記の既定リスト、アサイン候補は `自分(@me)` / `未アサイン` のみ、手順 3 の Project フィールド質問と手順 6 はスキップして Issue 作成だけ行う。

ID が一致しない場合は `gh project field-list {PROJECT_NUMBER} --owner {OWNER} --format json` で再取得する。

## 手順

### 1. 入力の受け取り

$ARGUMENTS またはユーザーの発言をそのまま使う。

入力がない場合のみ聞く：

```
Issue の内容を教えてください。
例: 「一覧画面でソートが効かない」「A と B と C を追加したい（関連内容）」
```

入力が曖昧で「何をやるか」が読み取れない場合は、それだけを聞く。設計や実装方針には踏み込まない。

### 2. issue 構成の判断とたたき台生成

**分割の単位は「1 つのユーザー価値」。** テンプレの「完了したら誰が何ができる / どうなるか」が 1 行で書ける単位を 1 issue とする。

- それぞれ独立した「誰が何をできる」で書ける → **価値が複数 = 分割する**（兄弟 or 親子）
- 全部が 1 つの「誰が何をできる」を実現するための**手段の列挙**（backend / frontend / migration、手順 1・2・3 など）→ **分割しない**。実装レイヤーや手順で子 issue を切らない。その分解は実装着手時に詰める（plan として持つ）
- **迷ったら割らない。** 後から価値で割るのは安く（着手前にいつでも割れる）、切りすぎた issue を統合する方が面倒だから

分割の例:

- 「起票時にアサインも優先度も選べるように」→「担当者を選べる」「優先度を選べる」は別の価値 → **分割**
- 「アサイン機能を追加（API 改修とフロントの選択 UI が必要）」→ どちらも 1 つの「担当者を選べる」を実現する手段 → **分割しない**（1 issue。内訳は実装時に詰める）

入力から以下を判断する:

- 単一トピック → 1 個の issue として作成
- 複数の関連項目（「A と B と C」「○○と××を追加」など）→ さらに「統合的な成果物」の有無で分ける:
  - **統合的な成果物がある**（子が全部終わった後に最後に実装・マージされる、まとめ役の成果物）→ **その成果物を親 issue、残りを子 issue（sub-issue）** にする。親も実装着手時に PR を持ち `Closes #親` で自動クローズされる前提とする
  - **統合的な成果物がない**（互いに独立した項目）→ **親を作らず、複数の兄弟 issue として並べる**（sub-issue 関係なし）

**進捗管理専用の親 issue（コード変更を伴わず PR に紐づかない親）は作らない。** PR に紐づかない issue は自動クローズの経路がなく手動クローズが残るため。親を立てるのは、それ自身が最後にマージされる成果物であるときに限る。

各 issue について以下を生成する:

- **タイトル**: 簡潔な要約（70 文字以内）
- **ラベル**: プロジェクト設定の `labels` があればそこから、無ければ以下から 1 つ選ぶ
  - `feature`: 新規機能 / 機能拡張
  - `bug`: 不具合の調査〜修正
  - `support`: 先方への質問回答・サポート対応
  - `docs`: ドキュメント作成
  - `refactor`: 振る舞いを変えずコード改善
  - `research`: 技術検証・スパイク
  - `chore`: 依存更新・CI・スクリプトなど雑務
- **本文**: プロジェクト設定の `template`（例: `.github/ISSUE_TEMPLATE/task.md`）があればその構造を正本とし、無くても次の 2 ブロックで書く:
  - **なぜ（発端）**: なぜやるか・誰のどの要望か。時間が経つと失われる耐久性のある文脈を 1〜2 行
  - **完了したら誰が何をできる / どうなるか**: 完了状態を観測可能な形で 1 行（「誰」はエンドユーザーに限らず受益者でよい）
  - How（実装方法）は書かない。実装着手時に詰める

AskUserQuestion で構成を確認:

```
■ 親 issue（最後にマージする統合的な成果物）
  タイトル: {parent_title}
  ラベル: {parent_label}
  本文: {parent_body}

■ 子 1
  タイトル: {child1_title}
  ラベル: {child1_label}
  本文: {child1_body}

■ 子 2
  ...

この構成で進めますか？
```

単一構成なら 1 つだけ、兄弟構成（親なし）なら親/子の区別なく並列に提示する。修正要望があれば反映して再確認。

### 3. フィールド設定を質問

**単一選択フィールド / Iteration / アサインは全 issue 共通**、**Priority / Estimate は issue ごと**に設定する。AI で勝手に判断しない（いずれも推奨値を出さない）。プロジェクト設定に GitHub Project が無ければ、この手順はアサインの質問だけ行う。

AskUserQuestion は 1 回あたり最大 4 問。

**共通質問（全 issue に適用）** をまとめて聞く:

1. **単一選択フィールド**（設定に定義があれば。例: Business）: 設定に列挙されたオプションから選ぶ
2. **今 Iteration に含める?**: `はい` / `いいえ`（「いいえ」なら未設定）
3. **アサイン**: `自分(@me)` / プロジェクト設定の `assignees` に列挙された login（設定順） / `未アサイン`（Other で任意の login を入力可）。選択肢は常にこの順で並べる

**Priority + Estimate（issue ごと）** を聞く:

各 issue（複数構成なら親も含む）について、Priority と Estimate を統合した 1 問を聞く:

- 選択肢は `未定`（Priority・Estimate とも設定しない）と **自由入力**（Other で `Priority, Estimate` を入力。例: `P0, 3` / `P1, 未定`）の 2 択
- カンマ区切りで解釈する: `P0`〜`P4` を Priority、数値を Estimate に対応。片方だけの指定も可（欠けた側は未設定）

単一 issue なら共通質問 ＋ その issue の Priority+Estimate の計 4 問を 1 回でまとめてよい。複数構成で合計が 4 問を超える場合は 4 問ずつ複数回に分けて聞く。

### 4. Issue 作成

親がある構成では親 issue → 各子 issue の順、兄弟構成では各 issue を順に作成する。アサインは共通の選択に応じて `--assignee` を付ける（`自分`→`@me`、設定のメンバー / Other→その login、`未アサイン`→フラグ自体を付けない）:

```bash
gh issue create --title "{TITLE}" --body "$(cat <<'EOF'
{BODY}
EOF
)" --label "{LABEL}" {ASSIGNEE_FLAG}
```

返却された URL から各 Issue 番号を抽出する。

### 5. sub-issue 登録（親＋子構成の場合のみ）

兄弟構成（親なし）ではスキップする。各子 issue を親に紐付ける。GitHub の sub-issue API は **数値の database ID** を必要とする（issue 番号ではない）。`sub_issue_id` は integer 型のため、文字列として送る `-f` ではなく typed field の `-F` で送る（`-f` だと 422 Invalid request になる）:

```bash
OWNER_REPO=$(gh repo view --json owner,name --jq '"\(.owner.login)/\(.name)"')

for CHILD_NUMBER in {子issue番号たち}; do
  CHILD_DB_ID=$(gh api "repos/$OWNER_REPO/issues/$CHILD_NUMBER" --jq .id)
  gh api -X POST "repos/$OWNER_REPO/issues/{親issue番号}/sub_issues" \
    -F sub_issue_id="$CHILD_DB_ID"
done
```

### 6. Project Item の取得とフィールド設定

プロジェクト設定に GitHub Project が無ければこの手順はスキップする。全 issue について Auto-add 反映後にフィールドを設定する。単一選択フィールド / Iteration は共通値、**Priority / Estimate は手順 3 で issue ごとに聞いた値**を使う。各 issue ごとに最大 5 回（1 秒間隔）リトライしてアイテムを見つける:

```bash
ITEM_ID=""
for i in 1 2 3 4 5; do
  ITEM_ID=$(gh project item-list {PROJECT_NUMBER} --owner {OWNER} --format json --limit 200 \
    --jq ".items[] | select(.content.number == $ISSUE_NUMBER) | .id")
  if [ -n "$ITEM_ID" ]; then break; fi
  sleep 1
done

if [ -z "$ITEM_ID" ]; then
  echo "⚠ Project Item が見つかりませんでした（#$ISSUE_NUMBER）。フィールドは手動で設定してください。"
else
  # 単一選択フィールド設定（全 issue 共通。設定に定義がある場合のみ）
  gh project item-edit --id "$ITEM_ID" \
    --field-id {SINGLE_SELECT_FIELD_ID} \
    --project-id {PROJECT_ID} \
    --single-select-option-id {選ばれた OPTION_ID}

  # Priority 設定（この issue の値。未定/未指定ならこのブロックはスキップ）
  gh project item-edit --id "$ITEM_ID" \
    --field-id {PRIORITY_FIELD_ID} \
    --project-id {PROJECT_ID} \
    --single-select-option-id {この issue の PRIORITY_OPTION_ID}

  # Estimate 設定（この issue の値。未定/未指定でなければ）
  if [ "$ESTIMATE" != "未定" ] && [ -n "$ESTIMATE" ]; then
    gh project item-edit --id "$ITEM_ID" \
      --field-id {ESTIMATE_FIELD_ID} \
      --project-id {PROJECT_ID} \
      --number "$ESTIMATE"
  fi

  # Iteration 設定（「はい」が選ばれた場合のみ）
  if [ "$ADD_TO_CURRENT" = "yes" ]; then
    TODAY=$(date +%Y-%m-%d)
    # field-list の Iteration field configuration から、TODAY が含まれる Iteration の ID を取得
    # （startDate <= TODAY < startDate + duration を満たすもの）
    CURRENT_ITERATION_ID=$(gh project field-list {PROJECT_NUMBER} --owner {OWNER} --format json --limit 50 \
      --jq '<Iteration field の configuration.iterations から TODAY を含むものの id を抽出する jq 式>')

    if [ -n "$CURRENT_ITERATION_ID" ]; then
      gh project item-edit --id "$ITEM_ID" \
        --field-id {ITERATION_FIELD_ID} \
        --project-id {PROJECT_ID} \
        --iteration-id "$CURRENT_ITERATION_ID"
    fi
  fi
fi
```

エラー時は警告のみ表示して続行する。

### 7. 完了

作成した Issue の URL 一覧と、設定した内容（単一選択フィールド / Iteration / アサイン＝全 issue 共通、Priority / Estimate＝issue ごと、親子関係）を表示する。
