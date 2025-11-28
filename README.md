# AI SDK v5 Data Parts Demo

Vercel AI SDK v5 の **Data Parts** 機能を使って、検索進捗をリアルタイムで可視化するデモです。

## 参考記事

- [Vercel AI SDK の Data Parts をカスタムして検索進捗を可視化する](https://zenn.dev/tsuboi/articles/26e3fe8fb6dc98)
- [AI SDK 5 - Vercel Blog](https://vercel.com/blog/ai-sdk-5)

## 技術スタック

- Next.js 16
- React 19
- AI SDK v5
- Zod 4
- Tailwind CSS 4
- TypeScript

## セットアップ

```bash
# 依存関係のインストール
pnpm install

# 環境変数の設定
cp .env.example .env.local
# .env.local に OpenAI API Key を設定

# 開発サーバー起動
pnpm dev
```

## ファイル構成

```
src/
├── app/
│   ├── api/
│   │   └── chat/
│   │       └── route.ts    # API Route - createUIMessageStream
│   ├── layout.tsx
│   ├── page.tsx            # Chat UI - useChat
│   └── globals.css
└── types/
    └── chat.ts             # カスタム UIMessage 型定義
```

## 主な機能

### Data Parts の型定義

```typescript
export type MyDataParts = {
  searchProgress: {
    stage: SearchStage;
    current: number;
    total: number;
    message: string;
  };
  status: {
    message: string;
    level: "info" | "warning" | "error";
  };
};

export type MyUIMessage = UIMessage<MyMetadata, MyDataParts>;
```

### サーバー側での Data Parts 送信

```typescript
const stream = createUIMessageStream<MyUIMessage>({
  execute: async ({ writer }) => {
    // 同じIDで書き込むと更新（Reconciliation）
    writer.write({
      type: "data-searchProgress",
      id: "search-1",
      data: { stage: "searching", current: 25, total: 100, message: "検索中..." },
    });

    // Transient（一時的）なデータ
    writer.write({
      type: "data-status",
      data: { message: "処理中...", level: "info" },
      transient: true,
    });
  },
});
```

### クライアント側での Data Parts 受信

```typescript
const { messages } = useChat<MyUIMessage>({
  onData: (dataPart) => {
    // Transient な部分はここでのみ受け取れる
    if (dataPart.type === "data-status") {
      showToast(dataPart.data.message);
    }
  },
});

// message.parts から永続化された Data Parts を取得
{message.parts.map((part) => {
  if (part.type === "data-searchProgress") {
    return <ProgressBar data={part.data} />;
  }
})}
```

## ポイント

1. **型安全性**: `UIMessage<Metadata, DataParts>` でサーバー〜クライアント間の型を統一
2. **Reconciliation**: 同じ `id` で書き込むと既存のパーツを更新
3. **Transient Parts**: `transient: true` で一時的なデータを送信（履歴に残らない）
4. **onData コールバック**: Transient なデータはここでのみ受け取れる
