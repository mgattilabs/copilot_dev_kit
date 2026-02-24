# Angular Advanced Component Patterns

## Custom Validator

```typescript
export function emailDomainValidator(allowedDomain: string): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const email = control.value as string;
    if (!email) return null;
    
    const isValid = email.endsWith(`@${allowedDomain}`);
    return isValid ? null : { emailDomain: { required: allowedDomain, actual: email.split('@')[1] } };
  };
}

// Usage
email: ['', [Validators.required, emailDomainValidator('company.com')]]
```

## Guard with Redirect

```typescript
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url }
  });
};
```

## Custom Component

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-my-component',
  templateUrl: './my-component.component.html',
  styleUrls: ['./my-component.component.scss']
})
export class MyComponent {
  title = 'Esempio';
  // sposta qui la logica TS già presente
}
```

<div class="my-component">
  <h1>{{ title }}</h1>
  <!-- incolla qui il template HTML inline -->
</div>

.my-component {
  h1 {
    color: #2c3e50;
  }
  /* incolla qui gli stili inline */
}
