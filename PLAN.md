# Repository Tab Bar Reapply Plan

GitHub Desktop のバージョンアップ時に、下部リポジトリタブバー機能を再適用するための手順。

## 目的

リポジトリ選択ポップアップの `Recent` とは別に、画面下端へリポジトリ履歴タブを表示する。

- 下端にステータスバー風のタブを表示する
- 新しく開いたリポジトリを右端へ追加する
- タブクリックでそのリポジトリへ切り替える
- タブ右側の `x` で履歴から削除する
- 順番は変更しない
- 上限は 8 件
- 8 件を超えたら左から削除する
- ただし現在選択中のリポジトリは自動削除しない
- 既存の `recently-selected-repositories` とは別履歴を使う

## 変更対象

`desktop` ソース側で以下を変更する。バージョンが上がって仕様が変更される可能性もあるので臨機応変に対応する。

- `app/src/lib/app-state.ts`
- `app/src/lib/stores/app-store.ts`
- `app/src/ui/dispatcher/dispatcher.ts`
- `app/src/ui/app.tsx`
- `app/styles/ui/_app.scss`

ビルド後に生成物を対象バージョンへコピーする。

- `desktop/out/renderer.js` -> `3.6.xx/renderer.js`
- `desktop/out/renderer.css` -> `3.6.xx/renderer.css`

## App State

`app/src/lib/app-state.ts` の `IAppState` に、タブバー専用履歴を追加する。

```ts
/**
 * List of repository IDs shown in the bottom repository tab bar
 */
readonly repositoryTabHistory: ReadonlyArray<number>
```

追加位置は `recentRepositories` の直後が分かりやすい。

## App Store

`app/src/lib/stores/app-store.ts` に、専用 localStorage キーと上限を追加する。

```ts
const RepositoryTabHistoryKey = 'repository-tab-history'
const RepositoryTabHistoryLength = 8
```

`AppStore` のフィールドに専用履歴を追加する。

```ts
private repositoryTabHistory: ReadonlyArray<number> = getNumberArray(
  RepositoryTabHistoryKey
)
```

`getState()` の戻り値に追加する。

```ts
repositoryTabHistory: this.repositoryTabHistory,
```

`_selectRepository()` で通常の Recent 更新直後に専用履歴も更新する。

```ts
this.updateRecentRepositories(previousRepositoryId, repository.id)
this.updateRepositoryTabHistory(repository.id)
```

同じクラス内に履歴更新と削除処理を追加する。

```ts
private updateRepositoryTabHistory(currentRepositoryId: number) {
  if (this.repositoryTabHistory.includes(currentRepositoryId)) {
    return
  }

  const repositoryTabHistory = [
    ...this.repositoryTabHistory,
    currentRepositoryId,
  ]

  while (repositoryTabHistory.length > RepositoryTabHistoryLength) {
    const removableIndex = repositoryTabHistory.findIndex(
      id => id !== currentRepositoryId
    )

    if (removableIndex === -1) {
      break
    }

    repositoryTabHistory.splice(removableIndex, 1)
  }

  setNumberArray(RepositoryTabHistoryKey, repositoryTabHistory)
  this.repositoryTabHistory = repositoryTabHistory
  this.emitUpdate()
}

public _removeRepositoryFromTabHistory(repositoryId: number) {
  const repositoryTabHistory = this.repositoryTabHistory.filter(
    id => id !== repositoryId
  )

  setNumberArray(RepositoryTabHistoryKey, repositoryTabHistory)
  this.repositoryTabHistory = repositoryTabHistory
  this.emitUpdate()
}
```

## Dispatcher

`app/src/ui/dispatcher/dispatcher.ts` に削除操作を追加する。

```ts
/** Remove a repository from the bottom repository tab history. */
public removeRepositoryFromTabHistory(repositoryId: number) {
  return this.appStore._removeRepositoryFromTabHistory(repositoryId)
}
```

追加位置は `selectRepository()` の直後。

## App UI

`app/src/ui/app.tsx` の `renderApp()` で、リポジトリ画面の直後にタブバーを描画する。

```tsx
{this.renderToolbar()}
{this.renderBanner()}
{this.renderRepository()}
{this.renderRepositoryTabBar()}
{this.renderPopups()}
{this.renderDragElement()}
```

同じクラスにタブバー描画とイベント処理を追加する。

```tsx
private renderRepositoryTabBar() {
  const tabs = this.state.repositoryTabHistory
    .map(id => this.state.repositories.find(r => r.id === id))
    .filter((r): r is Repository | CloningRepository => r !== undefined)

  if (tabs.length === 0) {
    return null
  }

  const selectedRepository = this.state.selectedState?.repository

  return (
    <div id="repository-tab-bar">
      {tabs.map(repository => {
        const isSelected = selectedRepository?.id === repository.id
        const title =
          repository instanceof Repository
            ? repository.alias ?? repository.name
            : repository.name

        return (
          <button
            key={repository.id}
            className={classNames('repository-tab', { selected: isSelected })}
            title={repository.path}
            onClick={() => this.onRepositoryTabClick(repository)}
          >
            <span className="repository-tab-title">{title}</span>
            <span
              className="repository-tab-close"
              role="button"
              aria-label={`Remove ${title} from repository tabs`}
              onClick={event =>
                this.onRepositoryTabCloseClick(event, repository.id)
              }
            >
              x
            </span>
          </button>
        )
      })}
    </div>
  )
}

private onRepositoryTabClick = (repository: Repository | CloningRepository) => {
  this.props.dispatcher.selectRepository(repository)
}

private onRepositoryTabCloseClick = (
  event: React.MouseEvent<HTMLSpanElement>,
  repositoryId: number
) => {
  event.stopPropagation()
  this.props.dispatcher.removeRepositoryFromTabHistory(repositoryId)
}
```

## Styling

`app/styles/ui/_app.scss` に追加する。場所は `.sr-only` より前。

```scss
#repository-tab-bar {
  display: flex;
  flex: 0 0 auto;
  min-height: 31px;
  overflow: hidden;
  padding: 0 8px 8px;
  background: #f8f8f8;
  border-top: 1px solid var(--box-border-color);
}

.repository-tab {
  display: flex;
  align-items: center;
  max-width: 180px;
  min-width: 0;
  margin: -1px 4px 0 0;
  padding: 0 8px 0 10px;
  color: var(--text-color);
  background: var(--box-background-color);
  border: 1px solid color-mix(in srgb, var(--box-border-color) 55%, transparent);
  border-top: 0;
  border-radius: 0 0 5px 5px;
  font-size: 12px;
  line-height: 24px;

  &.selected {
    color: var(--box-selected-active-text-color);
    background: color-mix(
      in srgb,
      var(--box-selected-active-background-color) 82%,
      white
    );
    border-color: color-mix(
      in srgb,
      var(--box-selected-active-background-color) 82%,
      white
    );
    font-weight: var(--font-weight-semibold);

    &:hover {
      color: var(--box-selected-active-text-color);
      background: var(--box-selected-active-background-color);
      border-color: var(--box-selected-active-background-color);
    }
  }

  &:hover {
    background: var(--box-alt-background-color);
  }
}

.repository-tab-title {
  font-family:var(--font-family-sans-serif);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.repository-tab-close {
  flex: 0 0 auto;
  margin-left: 14px;
  color: var(--text-secondary-color);

  &:hover {
    color: var(--text-color);
  }
}
```

デザイン意図:

- タブは下端に置くが、`border-top: 0` と下側の角丸で上下反転した見た目にする
- 上のコンテンツから下へ生えているように見せるため、タブの `margin-top` は `-1px`
- `x` はリポジトリ名から `14px` 離す
- 選択中は通常時を少し明るい青、hover 時を標準の選択青にする
- 選択中 hover でも文字色は `var(--box-selected-active-text-color)` を維持する


## 追加のスタイル変更
- `--font-family-monospace` の先頭に `HackGen35` を追加
- `.side-by-side-diff` の `font-size`を `13px` にする


## Build And Copy

`desktop` で production build する。

```powershell
cd desktop
yarn compile:prod
```

成功後、対象バージョンへ生成物をコピーする。例は `3.5.12`。フォルダが存在しない場合もあるので作成してから実行する。
バージョンの数字はユーザーから指示されない場合 `app/package.json` の `version` を見て決定する

```powershell
New-Item -Path "3.5.12" -ItemType Directory -Force
Copy-Item -LiteralPath "desktop\out\renderer.js" -Destination "3.5.12\renderer.js" -Force
Copy-Item -LiteralPath "desktop\out\renderer.css" -Destination "3.5.12\renderer.css" -Force
```

