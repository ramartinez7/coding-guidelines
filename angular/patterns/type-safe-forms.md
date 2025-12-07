# Type-Safe Forms

> Use typed forms (Angular 14+) for compile-time form validation and type safety.

## Problem

Traditional Angular forms use untyped `FormControl`, `FormGroup`, and `FormArray`, requiring runtime string-based field access and providing no compile-time type checking.

## Example

### ❌ Before (Untyped Forms)

```typescript
@Component({
  selector: 'app-user-form',
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <input formControlName="name">
      <input formControlName="email">
      <input formControlName="age">
      <button type="submit">Submit</button>
    </form>
  `
})
export class UserFormComponent {
  userForm = this.fb.group({
    name: [''],
    email: [''],
    age: [0]
  });

  constructor(private fb: FormBuilder) {}

  onSubmit() {
    // No type safety! String-based access
    const name = this.userForm.get('name')?.value; // type: any
    const email = this.userForm.get('emal')?.value; // Typo! No error
    const age = this.userForm.get('age')?.value; // type: any
    
    // What if we add a new field? No compiler help
    console.log(name, email, age);
  }
}
```

### ✅ After (Typed Forms - Angular 14+)

```typescript
interface UserForm {
  name: FormControl<string>;
  email: FormControl<string>;
  age: FormControl<number>;
}

@Component({
  selector: 'app-user-form',
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <input formControlName="name">
      <input formControlName="email">
      <input formControlName="age" type="number">
      <button type="submit">Submit</button>
    </form>
  `
})
export class UserFormComponent {
  userForm = this.fb.group<UserForm>({
    name: this.fb.control('', { nonNullable: true }),
    email: this.fb.control('', { nonNullable: true }),
    age: this.fb.control(0, { nonNullable: true })
  });

  constructor(private fb: FormBuilder) {}

  onSubmit() {
    // Type-safe access!
    const name: string = this.userForm.controls.name.value;
    const email: string = this.userForm.controls.email.value;
    const age: number = this.userForm.controls.age.value;
    
    // Typo causes compile error
    // const x = this.userForm.controls.emal.value; // ❌ Error!
    
    console.log({ name, email, age });
  }
}
```

## Why It's a Problem

1. **No compile-time safety**: Typos in field names only fail at runtime.

2. **Any types everywhere**: Form values default to `any`, losing all type information.

3. **Refactoring risks**: Renaming fields requires finding all string-based references manually.

4. **Poor IntelliSense**: No autocomplete for form fields or their values.

5. **Hidden bugs**: Type mismatches not caught until runtime.

## Symptoms

- Using `get('fieldName')` with string literals throughout code
- Type assertions (`as string`, `as number`) when accessing form values
- Runtime errors from typos in field names
- No autocomplete when accessing form controls
- Defensive null checks on every form access

## Benefits

- **Compile-time errors**: Typos and type mismatches caught immediately
- **Better IntelliSense**: Full autocomplete for fields and values
- **Safer refactoring**: Renaming fields updates all references automatically
- **Self-documenting**: Form structure clear from type definitions
- **Fewer runtime checks**: Type system guarantees correctness

## Typed FormGroup Pattern

```typescript
// Define form structure
interface AddressForm {
  street: FormControl<string>;
  city: FormControl<string>;
  zipCode: FormControl<string>;
}

interface UserProfileForm {
  name: FormControl<string>;
  email: FormControl<string>;
  address: FormGroup<AddressForm>;
}

@Component({...})
export class UserProfileComponent {
  profileForm = this.fb.group<UserProfileForm>({
    name: this.fb.control('', { nonNullable: true }),
    email: this.fb.control('', { nonNullable: true }),
    address: this.fb.group<AddressForm>({
      street: this.fb.control('', { nonNullable: true }),
      city: this.fb.control('', { nonNullable: true }),
      zipCode: this.fb.control('', { nonNullable: true })
    })
  });

  onSubmit() {
    // Fully typed access
    const street = this.profileForm.controls.address.controls.street.value;
    console.log(street); // type: string
  }
}
```

## Typed FormArray Pattern

```typescript
interface PhoneNumberForm {
  type: FormControl<'mobile' | 'home' | 'work'>;
  number: FormControl<string>;
}

interface ContactForm {
  name: FormControl<string>;
  phones: FormArray<FormGroup<PhoneNumberForm>>;
}

@Component({...})
export class ContactFormComponent {
  contactForm = this.fb.group<ContactForm>({
    name: this.fb.control('', { nonNullable: true }),
    phones: this.fb.array<FormGroup<PhoneNumberForm>>([])
  });

  addPhone() {
    const phoneGroup = this.fb.group<PhoneNumberForm>({
      type: this.fb.control<'mobile' | 'home' | 'work'>('mobile', { nonNullable: true }),
      number: this.fb.control('', { nonNullable: true })
    });
    
    this.contactForm.controls.phones.push(phoneGroup);
  }

  onSubmit() {
    const phones = this.contactForm.controls.phones.value;
    // type: Array<{ type: 'mobile' | 'home' | 'work'; number: string; }>
    console.log(phones);
  }
}
```

## NonNullable Option

By default, form controls can have `null` values. Use `nonNullable` to prevent this:

```typescript
// Without nonNullable
const control = this.fb.control('initial');
control.value; // type: string | null

// With nonNullable
const control = this.fb.control('initial', { nonNullable: true });
control.value; // type: string

// For the entire form
const form = this.fb.group({
  name: [''],
  email: ['']
}, { nonNullable: true }); // All controls are non-nullable
```

## Validation with Typed Forms

```typescript
interface LoginForm {
  username: FormControl<string>;
  password: FormControl<string>;
}

@Component({...})
export class LoginComponent {
  loginForm = this.fb.group<LoginForm>({
    username: this.fb.control('', {
      nonNullable: true,
      validators: [Validators.required, Validators.minLength(3)]
    }),
    password: this.fb.control('', {
      nonNullable: true,
      validators: [Validators.required, Validators.minLength(8)]
    })
  });

  onSubmit() {
    if (this.loginForm.valid) {
      // Type-safe access to valid values
      const { username, password } = this.loginForm.getRawValue();
      console.log(username, password); // Both type: string
    }
  }
}
```

## Extracting Types from Forms

```typescript
// Define the data model
interface User {
  name: string;
  email: string;
  age: number;
}

// Create form type from data model
type UserFormType = {
  [K in keyof User]: FormControl<User[K]>;
};

@Component({...})
export class UserFormComponent {
  userForm = this.fb.group<UserFormType>({
    name: this.fb.control('', { nonNullable: true }),
    email: this.fb.control('', { nonNullable: true }),
    age: this.fb.control(0, { nonNullable: true })
  });

  onSubmit() {
    const user: User = this.userForm.getRawValue();
    this.userService.save(user);
  }
}
```

## Migration from Untyped Forms

1. **Add form interface**:
   ```typescript
   interface MyForm {
     field1: FormControl<string>;
     field2: FormControl<number>;
   }
   ```

2. **Type the form group**:
   ```typescript
   formGroup = this.fb.group<MyForm>({...});
   ```

3. **Replace `.get()` with `.controls`**:
   ```typescript
   // Before
   this.form.get('field1')?.value
   
   // After
   this.form.controls.field1.value
   ```

4. **Add validators to control definitions**:
   ```typescript
   field1: this.fb.control('', {
     nonNullable: true,
     validators: [Validators.required]
   })
   ```

## Common Patterns

### Optional Fields

```typescript
interface ProfileForm {
  name: FormControl<string>;
  bio: FormControl<string | null>; // Optional field
}

const form = this.fb.group<ProfileForm>({
  name: this.fb.control('', { nonNullable: true }),
  bio: this.fb.control<string | null>(null) // Can be null
});
```

### Disabled Controls

```typescript
interface EditForm {
  id: FormControl<number>;
  name: FormControl<string>;
}

const form = this.fb.group<EditForm>({
  id: this.fb.control({ value: 1, disabled: true }, { nonNullable: true }),
  name: this.fb.control('', { nonNullable: true })
});

// getRawValue() includes disabled controls
const allValues = form.getRawValue(); // { id: number, name: string }

// value excludes disabled controls
const enabledOnly = form.value; // { name: string }
```

## See Also

- [TypeScript Strict Mode](./typescript-strict-mode.md)
- [Reactive Forms Over Template Forms](./reactive-forms-over-template.md)
- [Form Validation Patterns](./form-validation-patterns.md)
- [Custom Form Controls](./custom-form-controls.md)
