---
name: neo-angular
description: >
  Angular expertise for Neo. Activate when writing Angular components, services,
  directives, pipes, guards, resolvers, Angular routing, Angular forms (reactive
  or template-driven), Angular Signals, Standalone Components, Angular 19 patterns,
  RxJS operators, NgRx state management, or any .component.ts / .service.ts /
  .module.ts / angular.json file.
---

# Neo Angular Expertise

## Naming Conventions (Company Standard)

```typescript
// Interfaces and Types — PascalCase, NO "I" prefix
interface UserData { }
type ResponseStatus = 'success' | 'error';

// Functions and methods — camelCase
function getUserById(id: number): User { }

// Variables — camelCase
const userName = 'Mario';
let totalCount = 0;

// Global constants — UPPER_SNAKE_CASE
const MAX_RETRY_COUNT = 3;

// Enum — PascalCase for name and values
enum UserStatus {
  Active = 'ACTIVE',
  Inactive = 'INACTIVE'
}

// Files: kebab-case
// user-profile.component.ts
// auth.service.ts
// admin.guard.ts
```

## Angular 19 — Standalone Components (Default)

```typescript
// ✅ Always use standalone: true
@Component({
  selector: 'app-user-profile',
  standalone: true,
  imports: [CommonModule, RouterModule, ReactiveFormsModule],
  templateUrl: './user-profile.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserProfileComponent {
  // Angular 19: use signal-based inputs/outputs instead of decorators
  userId = input.required<string>();
  userUpdated = output<UserData>();

  // Computed signals
  private userService = inject(UserService);
  user = signal<UserData | null>(null);
  isLoading = signal(false);
  
  displayName = computed(() => 
    this.user()?.displayName ?? this.user()?.email ?? 'Anonymous'
  );

  ngOnInit(): void {
    effect(() => {
      this.loadUser(this.userId());
    });
  }

  private async loadUser(id: string): Promise<void> {
    this.isLoading.set(true);
    try {
      const user = await firstValueFrom(this.userService.getUser(id));
      this.user.set(user);
    } finally {
      this.isLoading.set(false);
    }
  }
}
```

## Angular 19 Signal APIs (use instead of decorators)

```typescript
// ❌ Old decorator style
@Input() userId: string;
@Output() userUpdated = new EventEmitter<UserData>();
@ViewChild('form') formRef: ElementRef;
@ContentChild(HeaderComponent) header: HeaderComponent;

// ✅ New signal style (Angular 19+)
userId = input.required<string>();
userName = input('Anonymous');                     // with default
userUpdated = output<UserData>();
formRef = viewChild.required<ElementRef>('form');
header = contentChild(HeaderComponent);
```

## Reactive Forms Pattern

```typescript
@Component({ ... })
export class CreateOrderFormComponent {
  private fb = inject(FormBuilder);

  form = this.fb.group({
    customerId: ['', [Validators.required]],
    items: this.fb.array([this.createItemGroup()]),
    notes: ['']
  });

  get items(): FormArray {
    return this.form.get('items') as FormArray;
  }

  createItemGroup(): FormGroup {
    return this.fb.group({
      productId: ['', Validators.required],
      quantity: [1, [Validators.required, Validators.min(1)]]
    });
  }

  addItem(): void {
    this.items.push(this.createItemGroup());
  }

  removeItem(index: number): void {
    this.items.removeAt(index);
  }

  onSubmit(): void {
    if (this.form.invalid) return;
    // emit or call service
  }
}
```

## HTTP Service Pattern

```typescript
@Injectable({ providedIn: 'root' })
export class OrderService {
  private http = inject(HttpClient);
  private apiUrl = inject(API_URL);  // use InjectionToken, never hardcode

  getOrders(): Observable<Order[]> {
    return this.http.get<Order[]>(`${this.apiUrl}/orders`).pipe(
      catchError(this.handleError)
    );
  }

  createOrder(command: CreateOrderCommand): Observable<string> {
    return this.http.post<string>(`${this.apiUrl}/orders`, command).pipe(
      catchError(this.handleError)
    );
  }

  private handleError(error: HttpErrorResponse): Observable<never> {
    const message = error.error?.message ?? error.message;
    console.error('API Error:', message);
    return throwError(() => new Error(message));
  }
}
```

## Smart vs Presentational Components

```typescript
// ✅ Smart (container) — handles data, passes to presentational
@Component({
  selector: 'app-orders-page',
  standalone: true,
  template: `
    <app-orders-list 
      [orders]="orders()" 
      [isLoading]="isLoading()"
      (orderSelected)="onOrderSelected($event)" />
  `
})
export class OrdersPageComponent {
  private orderService = inject(OrderService);
  orders = signal<Order[]>([]);
  isLoading = signal(false);

  ngOnInit(): void {
    this.loadOrders();
  }

  private loadOrders(): void {
    this.isLoading.set(true);
    this.orderService.getOrders()
      .pipe(finalize(() => this.isLoading.set(false)))
      .subscribe(orders => this.orders.set(orders));
  }

  onOrderSelected(order: Order): void {
    // handle navigation or detail
  }
}

// ✅ Presentational — pure, no service injection, only inputs/outputs
@Component({
  selector: 'app-orders-list',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `...`
})
export class OrdersListComponent {
  orders = input.required<Order[]>();
  isLoading = input(false);
  orderSelected = output<Order>();
}
```

## RxJS Operators (Common Patterns)

```typescript
// Search with debounce
searchControl = new FormControl('');

filteredResults$ = this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.service.search(term ?? '')),
  catchError(() => of([]))
);

// Load with retry
data$ = this.service.getData().pipe(
  retry({ count: 3, delay: 1000 }),
  shareReplay(1)
);
```

## Clean Code Rules (TypeScript Specific)

```typescript
// ✅ const by default, let only when needed, never var
const userId = 'abc';

// ✅ optional chaining and nullish coalescing
const city = user?.address?.city ?? 'Unknown';

// ✅ No any, use unknown if unsure
function parseData(input: unknown): UserData {
  if (typeof input !== 'object' || input === null) {
    throw new Error('Invalid input');
  }
  return input as UserData;
}

// ✅ Early return in methods
getUserDisplayName(user: User | null): string {
  if (!user) return 'Anonymous';
  if (!user.profile) return user.email;
  return user.profile.displayName ?? user.email;
}
```

## Angular Routing Pattern

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 'orders',
    loadComponent: () => import('./orders/orders-page.component')
      .then(m => m.OrdersPageComponent),
    canActivate: [authGuard],
    children: [
      {
        path: ':id',
        loadComponent: () => import('./orders/order-detail.component')
          .then(m => m.OrderDetailComponent),
        resolve: { order: orderResolver }
      }
    ]
  },
  { path: '', redirectTo: 'orders', pathMatch: 'full' },
  { path: '**', loadComponent: () => import('./not-found.component').then(m => m.NotFoundComponent) }
];
```

## References
- See [component-patterns.md](./references/component-patterns.md) for advanced component examples
