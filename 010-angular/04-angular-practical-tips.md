# Angular Practical Tips

---

## Performance

```typescript
// 1. TrackBy в ngFor — предотвращает пересоздание DOM
// ❌ Плохо
<div *ngFor="let item of items">
// ✅ Хорошо
<div *ngFor="let item of items; trackBy: trackById">
trackById(index: number, item: Item) => item.id

// 2. OnPush change detection
@Component({ changeDetection: ChangeDetectionStrategy.OnPush })

// 3. Ленивая загрузка изображений
<img [loading]="'lazy'" [src]="image.url" />

// 4. Пагинация на сервере (не фильтр 10000 items в памяти)
getUsers(page: number, size: number): Observable<Page<User>>

// 5. Использовать async pipe — меньше ручных subscribe
user$ = this.userService.getUser(id)
// Template: {{ user$ | async }}

// 6. trackBy для быстрых списков — критично при фильтрации/сортировке
```

---

## RxJS Patterns

```typescript
// 1. Auto-unsubscribe через AsyncPipe (рекомендуется)
// Никогда не subscribe вручную без cleanup

// 2. Когда нужен ручной subscribe
private destroy$ = new Subject<void>()

ngOnInit() {
  this.service.getData()
    .pipe(takeUntil(this.destroy$))
    .subscribe(data => this.data = data)
}

ngOnDestroy() {
  this.destroy$.next()
  this.destroy$.complete()
}

// 3. switchMap для поиска (отмена предыдущего запроса)
search$ = new Subject<string>()

ngOnInit() {
  this.search$.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(term => this.service.search(term))
  ).subscribe(results => this.results = results)
}

// 4. catchError + retry для API
this.http.get('/api/data').pipe(
  retry(2),
  catchError(err => {
    this.error = err.message
    return of([])
  })
)

// 5. NEVER call .subscribe inside .subscribe (subscribe hell)
```

---

## Forms

```typescript
// 1. FormBuilder для чистоты
constructor(private fb: FormBuilder) {
  this.form = this.fb.group({
    name: ['', [Validators.required, Validators.minLength(2)]],
    email: ['', [Validators.required, Validators.email]]
  })
}

// 2. Cross-field validation
const passwordMatchValidator: ValidatorFn = (control) => {
  const password = control.get('password')
  const confirm = control.get('confirmPassword')
  return password?.value === confirm?.value ? null : { mismatch: true }
}

// 3. markAllAsTouched() перед submit
onSubmit() {
  if (this.form.invalid) {
    this.form.markAllAsTouched()
    return
  }
  // save
}

// 4. ValueChanges для зависимых полей
this.form.get('country')?.valueChanges
  .pipe(switchMap(country => this.service.getCities(country)))
  .subscribe(cities => this.cities = cities)

// 5. Disable form while saving
this.form.disable()
try { await this.save() }
finally { this.form.enable() }
```

---

## Other Tips

```typescript
// 1. InjectionToken для конфигурации
export const API_CONFIG = new InjectionToken<ApiConfig>('API_CONFIG')
providers: [{ provide: API_CONFIG, useValue: { baseUrl: '/api' } }]

// 2. Pure pipes — кэшируются (не пересчитываются без изменения входа)
@Pipe({ name: 'filter', pure: true })

// 3. runOutsideAngular для частых событий
this.ngZone.runOutsideAngular(() => {
  window.addEventListener('mousemove', this.onMouseMove)
})

// 4. Состояние загрузки
loading$ = new BehaviorSubject<boolean>(false)

loadData() {
  this.loading$.next(true)
  this.service.getData().pipe(finalize(() => this.loading$.next(false)))
}

// 5. Route reuse strategy для сохранения scroll position
// Кастомная стратегия: RouteReuseStrategy
```
