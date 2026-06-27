# Angular Advanced

---

## Change Detection — Zone.js

**Факты:**
- Angular использует **Zone.js** для monkey-patching асинхронных API (setTimeout, addEventListener, XHR)
- После каждого async event → Change Detection проверяет все компоненты
- **OnPush** — проверяет только при @Input() изменении (new reference)
- **Detach** — `ChangeDetectorRef.detach()` — ручное управление

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserListComponent {
  constructor(private cdr: ChangeDetectorRef) {}
  
  // После async операции — вручную сообщаем
  async loadUsers() {
    this.users = await this.userService.getUsers().toPromise();
    this.cdr.markForCheck();  // Signal → root check
  }
  
  // Zone.js free — NgZone.runOutsideAngular
  constructor(private zone: NgZone) {
    this.zone.runOutsideAngular(() => {
      setInterval(() => this.pollServer(), 1000);
    });
  }
}
```

**Факт:** `markForCheck()` идёт ВВЕРХ до root (ApplicationRef), а `detectChanges()` — ВНИЗ от компонента.

**Факт:** Signal-based change detection (Angular 16+) — отказ от Zone.js, улучшение производительности.

---

## NgRx — State Management

```typescript
// Actions
export const loadUsers = createAction('[User Page] Load Users');
export const loadUsersSuccess = createAction(
  '[User API] Load Users Success',
  props<{ users: User[] }>()
);

// Reducer
export const userReducer = createReducer(
  initialState,
  on(loadUsersSuccess, (state, { users }) => ({
    ...state,
    users,
    loaded: true
  }))
);

// Effect
@Injectable()
export class UserEffects {
  loadUsers$ = createEffect(() =>
    this.actions$.pipe(
      ofType(loadUsers),
      switchMap(() =>
        this.userService.getUsers().pipe(
          map(users => loadUsersSuccess({ users })),
          catchError(error => of(loadUsersFailure({ error })))
        )
      )
    )
  );
}

// Selector (memoized)
export const selectUsers = createSelector(
  selectUserState,
  state => state.users
);

// Component
users$ = this.store.select(selectUsers);
```

**Факт:** NgRx использует **memoized selectors** — пересчёт только при изменении входа.

---

## Lazy Loading — стратегии

```typescript
// Preloading strategy
{ provide: PreloadingStrategy, useClass: PreloadAllModules }

// Custom preloading
export class CustomPreloader implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    return route.data?.['preload'] ? load() : of(null);
  }
}

// Route with preload metadata
{
  path: 'admin',
  loadChildren: () => import('./admin/admin.module'),
  data: { preload: true, roles: ['admin'] }
}
```

---

## Server-Side Rendering (Angular Universal)

```typescript
// app.server.module.ts
@NgModule({
  imports: [AppModule, ServerModule, ServerTransferStateModule]
})
export class AppServerModule {}

// TransferState — передача данных сервер→клиент
@Injectable()
export class DataService {
  constructor(private transferState: TransferState) {}
  
  getData(id: string): Observable<Data> {
    const key = makeStateKey<Data>(`data-${id}`);
    
    if (this.transferState.hasKey(key)) {
      const data = this.transferState.get(key, null!);
      this.transferState.remove(key);
      return of(data);  // Из SSR
    }
    
    return this.http.get<Data>(`/api/data/${id}`).pipe(
      tap(data => this.transferState.set(key, data))
    );
  }
}
```

---

## Custom Directives

```typescript
// Structural directive — *appHasPermission
@Directive({ selector: '[appHasPermission]' })
export class HasPermissionDirective implements OnInit {
  private hasView = false;
  
  @Input() appHasPermission!: string;
  
  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private auth: AuthService
  ) {}
  
  ngOnInit() {
    this.auth.hasPermission(this.appHasPermission).subscribe(can => {
      if (can && !this.hasView) {
        this.viewContainer.createEmbeddedView(this.templateRef);
        this.hasView = true;
      } else if (!can && this.hasView) {
        this.viewContainer.clear();
        this.hasView = false;
      }
    });
  }
}
// Usage: <div *appHasPermission="'reports:read'">Secret content</div>
```

---

## Angular Elements (Custom Elements)

```typescript
// Angular Component → Web Component
@NgModule({
  declarations: [UserCardComponent],
  imports: [BrowserModule],
  entryComponents: [UserCardComponent]
})
export class AppModule {
  constructor(private injector: Injector) {
    const el = createCustomElement(UserCardComponent, { injector });
    customElements.define('user-card', el);
  }
  ngDoBootstrap() {}
}
// Usage in any HTML: <user-card user-id="123"></user-card>
```

---

## Чек-лист

- [ ] Zone.js: monkey-patching, runOutsideAngular
- [ ] ChangeDetection: OnPush, markForCheck, detach, detectChanges
- [ ] Signals: Signal-based change detection (Angular 16+)
- [ ] NgRx: Store, Actions, Reducers, Effects, Selectors (memoized)
- [ ] Lazy Loading: PreloadAllModules, custom preloader
- [ ] SSR: Angular Universal, TransferState
- [ ] Custom Directives: structural (*appHasPermission)
- [ ] Angular Elements: Web Components from Angular
