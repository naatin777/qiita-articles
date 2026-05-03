---
title: Redux Toolkitで型安全にDIする
tags:
  - DI
  - redux-toolkit
private: false
updated_at: '2026-05-03T18:29:46+09:00'
id: e594c079b68820940794
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

Redux Toolkitで`createAsyncThunk`を使って非同期処理を書く際にThunk内で直接axiosやfetchを用いてハードコードしていませんか?
ハードコードすると、テストする際に実際の通信が走って時間がかかってしまったり、`jest.mock`でモジュール全体をモックする必要があり、コードが複雑になってしまいます。
この記事では、`extraArgument`と`createAsyncThunk.withTypes`を用いて、型安全なDIを実現する方法について紹介します。

:::note info
この先はほぼサンプルコードを貼り付けてるだけなので忙しい人は[公式ドキュメント](https://redux-toolkit.js.org/usage/usage-with-typescript#defining-a-pre-typed-createasyncthunk)を見てください。`createAsyncThunk.withTypes`について紹介したかっただけです。Redux Toolkit v2.0以上を対象としています。
:::

:::note info
よほどの理由がない限り[MSW](https://mswjs.io/)と[RTK Query](https://redux-toolkit.js.org/rtk-query/overview)を用いることをお勧めします。FirebaseやStripeを用いている場合はRTK Query周りがめんどくさいので今回の方法が良いと思います。また、React NativeとWebでロジックを共有する場合にも良いと思います。
:::

:::note info
`jest.mock`の方が本番コードはシンプルになるというメリットがあるので自分に合ってる方を選択することをお勧めします(私はキャストだらけになるのが嫌いなので今回の方法がいいと思ってます)。
:::

# サンプルコード

## APIのインターフェースを定義

DIではインターフェースに依存させることが重要なのでAPI通信をするサービスの実体とそのインターフェースを定義します。

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
      // 実行するたびに1秒なんて待てない
      setTimeout(() => {
        resolve({ id, name: `User ${id}` });
      }, 1000);
    });
  },
};
```

## Storeに依存を注入する

`configureStore`を呼び出す際に`getDefaultMiddleware`を通じて`thunk.extraArgument`に先ほどの`apiService`を渡します。これがDIコンテナとしての役割を果たします。

```ts:src/app/store.ts
import { configureStore } from "@reduxjs/toolkit";
import { apiService, ApiService } from "../services/api";
import userReducer from "../features/user/userSlice";

export interface AppExtraArgument {
  api: ApiService;
}

const extraArgument: AppExtraArgument = {
  api: apiService,
};

export const setupStore = (extra: AppExtraArgument) => {
  return configureStore({
    reducer: {
      user: userReducer,
    },
    middleware: (getDefaultMiddleware) => 
      getDefaultMiddleware({
        thunk: {
          // ここにオリジナルを差し込む
          extraArgument: extra,
        },
      }),
  });
};

export const store = setupStore(extraArgument);
export type AppDispatch = typeof store.dispatch;
export type RootState = ReturnType<typeof store.getState>;
```

## 型安全なカスタムThunkの作成

ここが重要なポイントでこのまま何もせずに`createAsyncThunk`を用いて`extra`経由で`api`を呼び出そうとしても、標準の`createAsyncThunk`は`extra`の型推論のデフォルトが`unknown`になっているため、めんどくさいことになります。
そのため、ここで`createAsyncThunk.withTypes`を用いて`extra`の型が定義されたオレオレThunk関数を作成します。

```ts:src/app/hooks.ts
import { createAsyncThunk } from "@reduxjs/toolkit";
import { useDispatch, useSelector } from "react-redux";
import { RootState, AppDispatch, AppExtraArgument } from "./store";

export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
// ここで型安全なcreateAsyncThunkを作る!!!
export const createAppAsyncThunk = createAsyncThunk.withTypes<{
  state: RootState;
  dispatch: AppDispatch;
  extra: AppExtraArgument;
  rejectValue: string;
}>();
```

## スライスの実装

ここで先ほど作成した`createAppAsyncThunk`を用いて非同期処理を書きます。
ここで`extra`が`AppExtraArgument`型と認識してくれるので型エラーもなく、補完機能もしっかり効きます!

```ts:src/features/user/userSlice.ts
import { createSlice, PayloadAction } from "@reduxjs/toolkit";
import { User } from "../../services/api";
import { createAppAsyncThunk } from "../../app/hooks";

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
      // extraがAppExtraArgument型になってる!!!
      const response = await extra.api.fetchUser(userId);
      return response;
    } catch (e) {
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
      })
      .addCase(
        fetchUserById.fulfilled,
        (state, action: PayloadAction<User>) => {
          state.loading = "succeeded";
          state.data = action.payload;
        },
      )
      .addCase(fetchUserById.rejected, (state, action) => {
        state.loading = "failed";
        state.error = action.payload || "unknown error";
      });
  },
});

export default userSlice.reducer;
```

## テスト

DIで最もその真価を発揮するのはテストです。
`jest.mock`を用いる必要はなく、モック化したAPIオブジェクトを`extraArgument`に差し込むだけなのでテストコードが読みやすくなります。

```ts:src/features/user/userSlice.test.ts
import { setupStore } from "../../app/store";
import userReducer, { fetchUserById } from "./userSlice";
import { ApiService, User } from "../../services/api";

describe("userSlice with DI", () => {
  it("should fetch user using the injected api service", async () => {
    // ここでapiServiceの代わりとなるモックを作成する
    const mockUser: User = { id: "test-id", name: "Mock User" };
    const mockApi: ApiService = {
      fetchUser: jest.fn().mockResolvedValue(mockUser),
    };

    const store = setupStore({ api: mockApi })
    
    await store.dispatch(fetchUserById("test-id"));

    const state = store.getState().user;
    expect(state.loading).toBe("succeeded");
    expect(state.data).toEqual(mockUser);
    expect(mockApi.fetchUser).toHaveBeenCalledWith("test-id");
  });

  it("should handle API errors", async () => {
    const mockApi: ApiService = {
      fetchUser: jest.fn().mockRejectedValue(new Error("API Error")),
    };

    const store = setupStore({ api: mockApi })

    await store.dispatch(fetchUserById("error-id"));

    const state = store.getState().user;
    expect(state.loading).toBe("failed");
    expect(state.error).toBe("Failed to fetch user");
  });
});
```

# おわりに

jestでテストを書いていた時にキャスト地獄でJavaScript化していたので気持ち悪さを感じました。
この設計パターンを取り入れることで保守性が高まると思うので試してみてください!!!
