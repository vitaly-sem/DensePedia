# Angular Basics

---

## Components

```typescript
@Component({
  selector: 'app-user-profile',
  templateUrl: './user-profile.component.html',
  styleUrls: ['./user-profile.component.scss'],
  changeDetection: ChangeDetectionStrategy.OnPush  // Default: Default
})
export class UserProfileComponent implements OnInit, OnDestroy {
  @Input() userId!: string;
  @Output() userUpdated = new EventEmitter<User>();
  @ViewChild('profileForm') form!: NgForm;
  
  user$!: Observable<User>;
  private destroy$ = new Subject<void>();
  
  ngOnInit() {
    this.user$ = this.userService.getUser(this.userId).pipe(
      takeUntil(this.destroy$)
    );
  }
  
  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Факт:** `ChangeDetectionStrategy.OnPush` — проверяет изменения только при новых @Input() ссылках или при ручном `markForCheck()`. Значительно повышает производительность.

---

## Templates & Directives

```html
<!-- Structural directives -->
<div *ngIf="user$ | async as user; else loading">
  <h1>{{ user.name }}</h1>
  
  <ul>
    <li *ngFor="let role of user.roles; trackBy: trackByRole; let i = index">
      {{ i }}: {{ role }}
    </li>
  </ul>
  
  <div [ngSwitch]="user.status">
    <p *ngSwitchCase="'active'">Active</p>
    <p *ngSwitchDefault>Unknown</p>
  </div>
</div>

<ng-template #loading>
  <spinner-component></spinner-component>
</ng-template>

<!-- Attribute directives -->
<input [(ngModel)]="user.name" />
<button [disabled]="!form.valid" (click)="save()">Save</button>
```

**Факт:** `trackBy` в `*ngFor` — предотвращает пересоздание DOM при изменении списка (ключ по идентификатору).

---

## Dependency Injection

```typescript
// Service with providedIn
@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(private http: HttpClient) { }
  
  getUser(id: string): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`);
  }
}

// Component-level provider
@Component({
  providers: [UserService]  // New instance per component
})

// InjectionToken
export const API_URL = new InjectionToken<string>('API_URL');
providers: [{ provide: API_URL, useValue: 'https://api.example.com' }]
```

**Факт:** `providedIn: 'root'` — singleton (tree-shakeable). `providedIn: 'any'` — lazy module singleton.

---

## Routing

```typescript
const routes: Routes = [
  { path: '', redirectTo: '/dashboard', pathMatch: 'full' },
  { path: 'dashboard', component: DashboardComponent },
  { path: 'users/:id', component: UserDetailComponent },
  { 
    path: 'admin', 
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    canLoad: [AuthGuard],
    canActivate: [AuthGuard],
    data: { roles: ['admin'] }
  },
  { path: '**', component: NotFoundComponent }
];

// Route guard
@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate, CanLoad {
  constructor(private auth: AuthService, private router: Router) {}
  
  canActivate(route: ActivatedRouteSnapshot): boolean {
    if (!this.auth.isLoggedIn) {
      this.router.navigate(['/login']);
      return false;
    }
    return true;
  }
}
```

---

## Reactive Forms

```typescript
export class ProfileComponent implements OnInit {
  profileForm = new FormGroup({
    name: new FormControl('', [Validators.required, Validators.minLength(2)]),
    email: new FormControl('', [Validators.required, Validators.email]),
    address: new FormGroup({
      street: new FormControl(''),
      city: new FormControl('')
    })
  });
  
  get email() => this.profileForm.get('email')!;
  
  onSubmit() {
    if (this.profileForm.valid) {
      this.userService.update(this.profileForm.value).subscribe();
    }
  }
}
```

---

## Pipes

```typescript
// Built-in: date, uppercase, currency, json, async, slice, decimal

// Custom pipe
@Pipe({ name: 'truncate', pure: true })
export class TruncatePipe implements PipeTransform {
  transform(value: string, maxLength: number = 100, suffix: string = '...'): string {
    return value.length > maxLength 
      ? value.substring(0, maxLength) + suffix 
      : value;
  }
}
// {{ description | truncate:50 }}
```

---

## Чек-лист

- [ ] Components: @Input/@Output, lifecycle hooks (OnInit, OnDestroy, OnChanges)
- [ ] Templates: *ngIf, *ngFor (trackBy), *ngSwitch, pipes (async pipe for observables)
- [ ] DI: providedIn, InjectionToken, factory providers
- [ ] Routing: lazy loading, guards (CanActivate, CanLoad, CanDeactivate)
- [ ] Forms: Reactive Forms (FormGroup, FormControl, Validators)
- [ ] Services: HttpClient, RxJS (Observable, pipe, operators)
- [ ] ChangeDetection: Default vs OnPush
