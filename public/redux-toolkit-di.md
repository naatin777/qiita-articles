---
title: Redux Toolkitで型安全にDIする
tags:
  - DI
  - redux-toolkit
private: false
updated_at: '2026-09-16T01:28:10+09:00'
id: e594c079b68820940794
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

# はじめに

Redux Toolkitで`createAsyncThunk`を使って非同期処理を書くとき、Thunkの中で直接`axios`や`fetch`を呼び出していませんか？

Thunk内に通信処理を直接書くと、テスト時に実際の通信が走ってしまったり、`jest.mock`でモジュール全体をモックする必要があったりして、テストコードが複雑になりがちです。

この記事では、Redux Toolkitの`thunk.extraArgument`と`createAsyncThunk.withTypes`を使って、型安全に依存を注入する方法を紹介します。

:::note info
この記事は、Redux Toolkit v2.0以上を対象にしています。  
`createAsyncThunk.withTypes`の使い方を紹介したい記事なので、詳しく知りたい方は公式ドキュメントの「Defining a Pre-Typed createAsyncThunk」も確認してみてください。
:::

:::note info
通常のAPI通信では、MSWやRTK Queryを使う方が適している場面も多いです。  
一方で、FirebaseやStripeのような外部SDKを使う場合や、React NativeとWebでロジックを共有したい場合は、今回のように依存を注入する設計が扱いやすいことがあります。
:::

:::note info
`jest.mock`を使う方が本番コードをシンプルに保てる場合もあります。  
ただし、モック対象が増えると型キャストが多くなりがちなので、型安全にテストを書きたい場合は今回の方法も選択肢になります。
:::

# サンプルコード

## APIのインターフェースを定義する

まずは、API通信を行うサービスのインターフェースを定義します。

Thunkが具体的な実装ではなくインターフェースに依存するようにしておくことで、本番用の実装とテスト用のモックを差し替えやすくなります。

```ts:src/services/api.ts
export interface User {
  id: string;
  name: string;
}

export interface ApiService {
  fetchUser(id: string): Promise<User>;
}

export const apiService: ApiService = {
  fetchUser: async (id) => {
    return new Promise((resolve) => {
      // 実行するたびに1秒待つような処理は、テストでは避けたい
      setTimeout(() => {
        resolve({ id, name: `User ${id}` });
      }, 1000);
    });
  },
};
```

## Storeに依存を注入する

`configureStore`を呼び出すときに、`getDefaultMiddleware`経由で`thunk.extraArgument`を設定します。

ここで渡した値は、`createAsyncThunk`のpayload creator内で`extra`として参照できます。

```ts:src/app/store.ts
import { configureStore } from "@reduxjs/toolkit";
import { apiService } from "../services/api";
import userReducer from "../features/user/userSlice";
import type { ApiService } from "../services/api";

export interface AppExtraArgument {
  api: ApiService;
}

const extraArgument: AppExtraArgument = {
  api: apiService,
};

export const setupStore = (extra: AppExtraArgument = extraArgument) => {
  return configureStore({
    reducer: {
      user: userReducer,
    },
    middleware: (getDefaultMiddleware) =>
      getDefaultMiddleware({
        thunk: {
          extraArgument: extra,
        },
      }),
  });
};

export const store = setupStore();

export type AppStore = ReturnType<typeof setupStore>;
export type AppDispatch = AppStore["dispatch"];
export type RootState = ReturnType<AppStore["getState"]>;
```

## 型安全なカスタムThunkを作成する

そのまま`createAsyncThunk`を使うと、`extra`の型は自動では分かりません。

そこで、`createAsyncThunk.withTypes`を使って、アプリ用に型付け済みの`createAsyncThunk`を作成します。

```ts:src/app/asyncThunk.ts
import { createAsyncThunk } from "@reduxjs/toolkit";
import type { RootState, AppDispatch, AppExtraArgument } from "./store";

export const createAppAsyncThunk = createAsyncThunk.withTypes<{
  state: RootState;
  dispatch: AppDispatch;
  extra: AppExtraArgument;
  rejectValue: string;
}>();
```

React Reduxのhooksも型付けしておきます。

```ts:src/app/hooks.ts
import { useDispatch, useSelector } from "react-redux";
import type { RootState, AppDispatch } from "./store";

export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
```

## スライスを実装する

先ほど作成した`createAppAsyncThunk`を使って非同期処理を書きます。

`extra`が`AppExtraArgument`型として扱われるため、`extra.api.fetchUser`の補完も効きます。

```ts:src/features/user/userSlice.ts
import { createSlice } from "@reduxjs/toolkit";
import { createAppAsyncThunk } from "../../app/asyncThunk";
import type { User } from "../../services/api";

interface UserState {
  data: User | null;
  loading: "idle" | "pending" | "succeeded" | "failed";
  error: string | null;
}

const initialState: UserState = {
  data: null,
  loading: "idle",
  error: null,
};

export const fetchUserById = createAppAsyncThunk(
  "user/fetchById",
  async (userId: string, { extra, rejectWithValue }) => {
    try {
      const user = await extra.api.fetchUser(userId);
      return user;
    } catch {
      return rejectWithValue("Failed to fetch user");
    }
  },
);

const userSlice = createSlice({
  name: "user",
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchUserById.pending, (state) => {
        state.loading = "pending";
        state.error = null;
      })
      .addCase(fetchUserById.fulfilled, (state, action) => {
        state.loading = "succeeded";
        state.data = action.payload;
      })
      .addCase(fetchUserById.rejected, (state, action) => {
        state.loading = "failed";
        state.error = action.payload ?? "unknown error";
      });
  },
});

export default userSlice.reducer;
```

## テストを書く

DIのメリットが特に分かりやすいのはテストです。

`jest.mock`でモジュール全体をモックしなくても、モック化したAPIオブジェクトを`extraArgument`に差し込むだけでテストできます。

```ts:src/features/user/userSlice.test.ts
import { setupStore } from "../../app/store";
import { fetchUserById } from "./userSlice";
import type { ApiService, User } from "../../services/api";

describe("userSlice with DI", () => {
  it("injected api serviceを使ってユーザーを取得できる", async () => {
    const mockUser: User = {
      id: "test-id",
      name: "Mock User",
    };

    const mockApi: ApiService = {
      fetchUser: jest.fn().mockResolvedValue(mockUser),
    };

    const store = setupStore({ api: mockApi });

    await store.dispatch(fetchUserById("test-id"));

    const state = store.getState().user;

    expect(state.loading).toBe("succeeded");
    expect(state.data).toEqual(mockUser);
    expect(mockApi.fetchUser).toHaveBeenCalledWith("test-id");
  });

  it("APIエラーを扱える", async () => {
    const mockApi: ApiService = {
      fetchUser: jest.fn().mockRejectedValue(new Error("API Error")),
    };

    const store = setupStore({ api: mockApi });

    await store.dispatch(fetchUserById("error-id"));

    const state = store.getState().user;

    expect(state.loading).toBe("failed");
    expect(state.error).toBe("Failed to fetch user");
  });
});
```

# おわりに

`createAsyncThunk`の中で直接APIクライアントを呼び出すと、実装は簡単ですが、テスト時にモックしづらくなることがあります。

`thunk.extraArgument`で依存を外から渡し、`createAsyncThunk.withTypes`で`extra`に型を付けておくと、Thunkの中でも補完が効き、テストでも本番用の実装とモックを簡単に差し替えられます。

`jest.mock`で十分な場合もありますが、型キャストが増えてつらくなってきたら、今回のような設計も選択肢として試してみてください。
