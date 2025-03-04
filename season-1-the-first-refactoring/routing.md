# ルーティングと状態管理

このサンプルの最後に、ルーティングでユーザー詳細のページへ移動できるようにします。アプリケーションは最終的に次のようになります。ユーザーリストから選択されたユーザーの詳細情報が表示され、URLは `/users/<ユーザーのID>` に変化します。ページの遷移中には、個別のユーザー情報を取得中であることを示す Fetching... の表示がおこなわれます。

![](../.gitbook/assets/large-gif-604x664.gif)

### User型の拡張

ユーザー詳細ページを作るにあたって、まずは現在の `User` 型を拡張します。名前に加えて、メールアドレスと電話番号、会社名をもつようになります。

{% code-tabs %}
{% code-tabs-item title="user.ts" %}
```typescript
export interface User {
  id: string;
  name: string;
  email: string;
  phone: string;
  company: {
    name: string;
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### Stateの拡張

ユーザー詳細にかかわるアプリケーションの状態の定義を、 `State` 型に追加し、初期値を設定します。詳細を表示するユーザーのインスタンスを `userDetail.user` に保持すると、次のようになります。（変更部分だけを表示しています）

{% code-tabs %}
{% code-tabs-item title="state.ts" %}
```typescript
import { User } from './user';

export interface State {
  // ...
  userDetail: {
    user: User | null;
  }
}

export const initialState = {
  // ...
  userDetail: {
    user: null,
  }
};
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### UserApiServiceの作成

さて、これまでユーザーの配列をHTTPリクエスト経由で取得するのは、 `UserListUsecase` の責任でした。しかし今回のアップデートでは、個別のユーザー情報をID指定で取得するAPIを呼び出す必要が生まれたので、User APIへのHTTPリクエストを責務とする `UserApiService` を新たに作成します。そして `UserListUsecase` は直接HTTPリクエストを送るのではなく、 サービス経由でレスポンスを受け取るようにリファクタリングします。

`UserApiService` は次のようなクラスです。すべてのユーザーを取得するAPI呼び出しと、個別のユーザーをID指定で取得するAPI呼び出しの両方をサポートします。

{% code-tabs %}
{% code-tabs-item title="user-api.service.ts" %}
```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { User } from '../user';

const apiHost = 'https://jsonplaceholder.typicode.com';

@Injectable({ providedIn: 'root' })
export class UserApiService {

  constructor(private http: HttpClient) { }

  async getAllUsers() {
    return await this.http.get<User[]>(`${apiHost}/users`).toPromise();
  }

  async getUserById(id: string) {
    return await this.http.get<User>(`${apiHost}/users/${id}`).toPromise();
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

そして `UserListUsecase` は次のようにリファクタリングします。 `HttpClient` に依存していましたが、代わりに `UserApiService` に依存するようになり、 `fetchUsers` メソッドの中で利用するようになりました。（変更部分だけを表示しています）

{% code-tabs %}
{% code-tabs-item title="user-list.usecase.ts" %}
```typescript
import { Injectable } from '@angular/core';
import { filter } from 'rxjs/operators';
import { Store } from '../service/store.service';
import { UserApiService } from '../service/user-api.service';
import { User } from '../user';

@Injectable({ providedIn: 'root' })
export class UserListUsecase {

  constructor(private userApi: UserApiService, private store: Store) { }

  async fetchUsers() {
    const users = await this.userApi.getAllUsers();

    this.store.update(state => ({
      ...state,
      userList: {
        ...state.userList,
        items: users,
      }
    }));
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### UserDetailPageComponentの作成

これで準備が整ったので、ルーティングによって表示されるユーザー詳細のコンポーネントを `UserDetailPageComponent` として作成しましょう。 はじめは単純なコンポーネントで、 `RouterModule` のセットアップを行います。Angularのルーティングについての詳細は[公式ドキュメント](https://angular.jp/guide/router)を参考にしてください。

ルーティングに利用されるコンポーネントは、 `ActivatedRoute` サービスを利用することで、ルーティングに関わる情報にアクセスできます。たとえば、 `/users/:userId` でルーティングされるコンポーネントから `userId` を取得するために、 `ActivatedRoute.params` を利用します。ここではサービスをインジェクトするだけにとどめ、あとで利用します。

{% code-tabs %}
{% code-tabs-item title="app-routing.module.ts" %}
```typescript
import { NgModule } from '@angular/core';
import { RouterModule } from '@angular/router';
import { UserDetailPageComponent } from './view/user-detail-page/user-detail-page.component';

@NgModule({
  imports: [
    RouterModule.forRoot([
      {
        path: 'users/:userId',
        component: UserDetailPageComponent
      }
    ])
  ],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```
{% endcode-tabs-item %}

{% code-tabs-item title="user-detail-page.component.ts" %}
```typescript
import { Component } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

@Component({
  templateUrl: './user-detail-page.component.html',
  styleUrls: ['./user-detail-page.component.css']
})
export class UserDetailPageComponent {
  constructor(private route: ActivatedRoute) {
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### URLパラメータの監視と状態の接続

今回のアップデートで要求されるのは以下のステップです。

1. ルーティングにより変化するURL中の `userId` を監視する
2. `userId` が変化したら、そのIDをもとにAPIを呼び出す
3. APIのレスポンスをアプリケーションの状態に反映する
4. 変化したアプリケーションの状態をコンポーネントで描画する

まずは最初のステップを実装しましょう。 `userId` は `ActivatedRoute.params` のObservableを購読し、渡されるオブジェクトから `params['userId']` のように取得できます。`userId` が変更したときにだけコールバック関数が呼び出されるように注意して実装すると次のようになります。 `onDestroy$` はコンポーネントが破棄されたタイミングで完了するObservableです。このObservableとRxJSの `takeUntil` オペレーターを使った自動的な購読停止のパターンは、ルーティングに限らずコンポーネントが明示的にObservableを購読しなければならない場合にとても有用です。

{% code-tabs %}
{% code-tabs-item title="user-detail-page.component.ts" %}
```typescript
import { Component, OnDestroy, EventEmitter } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { Observable } from 'rxjs'
import { map, takeUntil, distinctUntilChanged } from 'rxjs/operators'
import { UserDetailUsecase } from '../../usecase/user-detail.usecase';
import { User } from '../../user';

@Component({
  templateUrl: './user-detail-page.component.html',
  styleUrls: ['./user-detail-page.component.css']
})
export class UserDetailPageComponent implements OnDestroy {
  private onDestroy$ = new EventEmitter();

  constructor(private route: ActivatedRoute, private userDetailUsecase: UserDetailUsecase) {
    this.route.params.pipe(
      // コンポーネントの破棄と同時に停止する
      takeUntil(this.onDestroy$),
      // paramsからuserIdを取り出す
      map(params => params['userId']),
      // userIdが変わったときだけ値を流す
      distinctUntilChanged(),
    ).subscribe(userId => {
      // ユーザーIDを使った処理を記述する
    });
  }

  ngOnDestroy() {
    this.onDestroy$.complete();
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

続いて、ユーザーIDを使ってAPIを呼び出すステップを実装しなければならないですが、ここまでの内容を読んでいればわかるように、明らかにこのコンポーネントに記述すべきではありませんね。さらに言えば、 `ActivatedRoute` をどのように監視してユーザーIDを取り出すかについても、ビューを担当するコンポーネントの責務ではありません。

このようなときの解決策は、サービスを利用することです。ユーザーリストに対して `UserListUsecase` サービスを作ったように、ユーザー詳細についても `UserDetailUsecase` を作成し、責務を分割しましょう。

### UserDetailUsecase の作成

`UserDetailUsecase` は`UserListUsecase` と同じように、コンポーネントが持つべきでない責務を引き受け、柔軟に要求をこなす便利屋です。まずは、ユーザーIDの変化を監視する役目を、 `subscribeRouteChange` メソッドとして実装します。ここでのポイントは、 `ActivatedRoute` のインジェクトはコンポーネントがおこなうことです。`ActivatedRoute` はRouterの設定に対応した階層構造を持っており、ルーターによりアクティベートされたコンポーネント以外でインジェクトすると、うまく意図通りのイベントを購読できないことがあります。 `/users/:userId` のパラメーターを取得したい場合には、そのパスと対応した `UserDetailPageComponent` でインジェクトします。

{% code-tabs %}
{% code-tabs-item title="user-detai.usecase.ts" %}
```typescript
import { Injectable } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { Observable } from 'rxjs';
import { takeUntil, map, distinctUntilChanged } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class UserDetailUsecase {

  subscribeRouteChanges(route: ActivatedRoute, untilObservable: Observable<any>) {
    route.params.pipe(
      takeUntil(untilObservable),
      map(params => params['userId']),
      distinctUntilChanged(),
    ).subscribe(userId => this.onUserIdChanged(userId));
  }

  private async onUserIdChanged(userId: string) {
  }
}
```
{% endcode-tabs-item %}

{% code-tabs-item title="user-detail-page.component.ts" %}
```typescript
import { Component, OnDestroy, EventEmitter } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { UserDetailUsecase } from '../../usecase/user-detail.usecase';

@Component({
  templateUrl: './user-detail-page.component.html',
  styleUrls: ['./user-detail-page.component.css']
})
export class UserDetailPageComponent implements OnDestroy {
  private onDestroy$ = new EventEmitter();

  constructor(
    private route: ActivatedRoute, 
    private userDetailUsecase: UserDetailUsecase
  ) {
    this.userDetailUsecase.subscribeRouteChanges(this.route, this.onDestroy$);
  }

  ngOnDestroy() {
    this.onDestroy$.complete();
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### API呼び出しと状態の更新

`onUserIdChanged` メソッドの中で、APIを呼び出してレスポンスをもとに状態を更新します。さきほど作成した `UserApiService` を利用して、ユーザーIDからユーザーを取得します。（変更部分だけを表示しています）また、`UserListUsecase` と同じように、`user$` ゲッターは  `Store` からユーザー詳細の描画に必要な状態を選択して公開しています。

{% code-tabs %}
{% code-tabs-item title="user-detail.usecase.ts" %}
```typescript
import { Injectable } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { Observable } from 'rxjs';
import { takeUntil, map, distinctUntilChanged } from 'rxjs/operators';
import { Store } from '../service/store.service';
import { UserApiService } from '../service/user-api.service';
import { User } from '../user';

@Injectable({ providedIn: 'root' })
export class UserDetailUsecase {

  get user$() {
    return this.store.select(state => state.userDetail.user);
  }

  constructor(private userApi: UserApiService, private store: Store) { }

  private async onUserIdChanged(userId: string) {
    // リクエスト前に現在の状態をリセットする
    this.store.update(state => ({
      ...state,
      userDetail: {
        ...state.userDetail,
        user: null,
      }
    }));

    // APIを呼び出す
    const user = await this.userApi.getUserById(id);

    // 状態を更新する
    this.store.update(state => ({
      ...state,
      userDetail: {
        ...state.userDetail,
        user,
      }
    }));
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

最後に `UserDetailPageComponent` を次のように変更し、ユーザー情報を表示します。 `ngIf` と `async` パイプを使い、ユーザーが存在する時には情報を表示し、`null` のときには読み込み中である表示をおこないます。

{% code-tabs %}
{% code-tabs-item title="user-detail-page.component.ts" %}
```typescript
import { Component, OnDestroy, EventEmitter } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { UserDetailUsecase } from '../../usecase/user-detail.usecase';

@Component({
  templateUrl: './user-detail-page.component.html',
  styleUrls: ['./user-detail-page.component.css']
})
export class UserDetailPageComponent implements OnDestroy {
  user$ = this.userDetailUsecase.user$;

  private onDestroy$ = new EventEmitter();

  constructor(private route: ActivatedRoute, private userDetailUsecase: UserDetailUsecase) {
    this.userDetailUsecase.subscribeRouteChanges(this.route, this.onDestroy$);
  }

  ngOnDestroy() {
    this.onDestroy$.complete();
  }
}
```
{% endcode-tabs-item %}

{% code-tabs-item title="user-detail-page.component.html" %}
```markup
<ng-container *ngIf="user$ | async as user; else userFetching">
  <h1>{{user.name}}</h1>

  <dl>
    <dt>Email</dt>
    <dd>{{ user.email }}</dd>
    <dt>Phone</dt>
    <dd>{{ user.phone }}</dd>
     <dt>Company</dt>
    <dd>{{ user.company.name }}</dd>
  </dl>

</ng-container>

<ng-template #userFetching>
  <div>Fetching...</div>
</ng-template>
```
{% endcode-tabs-item %}
{% endcode-tabs %}

これですべての実装がおわりました。完成したアプリケーションは以下のサンプルコードから実行できます。

{% embed url="https://stackblitz.com/edit/angular-yfsmkj" %}

## さらに良くするには？

さて、このサンプルではHTTPリクエストでAPIを呼び出し、ユーザーリストを表示し、ユーザー詳細へルーティングするという、一般的なデータ駆動のアプリケーションを作成しました。アプリケーションとしては小規模ですが、一般的なAngularアプリケーションの開発に役立ついくつかの示唆を得ることができました。

1. コンポーネントは単一責任の原則にしたがって親子関係を分割する
2. コンポーネントがもつべきでないビューと関係のない処理はサービスに移動する
3. コマンド・クエリ分離原則にしたがって、副作用の発生し得る箇所を限定する
4. ビューとの結合性が高く、単独の責務として定義しづらいビジネスロジックをUsecaseに集約し、API呼び出しや状態管理などのサービスの独立性を維持する

サンプルアプリケーションにはまだリファクタリングできる場所がいくつもあります。たとえば、ユーザー詳細は `UserDetailPageComponent` が `UserDetailUsecase` を利用しているのに対して、ユーザーリストは `AppComponent` が `UserListUsecase` を利用していて、対照性に問題があります。また、ユーザー詳細で取得しようとしたユーザーが存在せず404エラーが返されたときの考慮はできていません。

これらの問題の解決も含め、はじめに書いたように設計に答えはありません。大事なことは、一貫した原則や方針に基づいて設計をすることと、はじめから完璧な設計を目指さないことです。このサンプルで示したように、最初はAppComponentに直接あらゆる処理を書いてもかまいません。プログラムはまずは意図通りに実行できることが第一です。機能的な要求を満たすことが確認できたら、それを壊さないように設計を見直し、リファクタリングをおこなうという、一連の流れを習慣づけるようにしましょう。





