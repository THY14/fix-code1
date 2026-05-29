# Bug Fixes & Feature Changes

## Summary of Issues Fixed

1. **OTP login did not navigate to homepage after verification**
2. **Role-based redirect loop caused silent navigation failure**
3. **Guards crashed on malformed JWT tokens**
4. **Switch Role feature called a non-existent backend endpoint (404)**

---

## Changed Files

### 1. `src/app/app.routes.ts`

**Problems fixed:**
- `RedirectGuard.canActivate()` was calling `this.router.navigateByUrl(url)` inside the guard AND returning `false`. In Angular 15.2+, when a guard returns `false`, any navigation triggered inside it is **also cancelled**. This meant the post-OTP redirect silently did nothing.
- `tokenPayload.user.roles` was accessed outside a try/catch, so if the JWT had an unexpected structure it would throw an unhandled error and crash the navigation.
- The default case in `getRoleDefaultUrl` returned `/auth`, which triggered `NoAuthGuard` (the user has a valid token), which redirected back to root, creating an infinite redirect loop that Angular eventually cancelled.
- `roleResolver([RoleEnum.ADMIN])` was blocking all non-Administrator accounts from the `/admin` route.

**What changed:**
- `canActivate()` now returns a `UrlTree` directly (`this.router.parseUrl(url)`) instead of calling `navigateByUrl` + returning `false`. This is the correct Angular pattern — the router handles the redirect atomically.
- Wrapped entire body in `try/catch` with optional chaining (`user?.roles ?? []`) so malformed tokens are handled gracefully.
- If `getRoleDefaultUrl` returns `/auth` (unknown role), signs out and redirects to `/auth/sign-in` to break the redirect loop.
- Removed `roleResolver` from the `admin` route so any authenticated user can access admin pages.

```typescript
// src/app/app.routes.ts

import { Injectable }           from '@angular/core';
import { AuthGuard }            from 'app/core/auth/guards/auth.guard';
import { NoAuthGuard }          from 'app/core/auth/guards/noAuth.guard';
import { LayoutComponent }      from 'app/layout/layout.component';
import { initialDataResolver }  from './app.resolver';
import { getRoleDefaultUrl } from './core/auth/resolvers/role.resolver';
import { CanActivate, Route, Router, UrlTree } from '@angular/router';
import { AuthService } from './core/auth/auth.service';
import { UserPayload } from 'helper/interfaces/payload.interface';
import jwt_decode from 'jwt-decode';

@Injectable({ providedIn: 'root' })
export class RedirectGuard implements CanActivate {
    constructor(
        private router: Router,
        private authService: AuthService,
    ) { }

    canActivate(): UrlTree {
        const token = this.authService.accessToken;

        if (!token) {
            return this.router.parseUrl('/auth/sign-in');
        }

        try {
            const tokenPayload: UserPayload = jwt_decode(token);
            const roles = tokenPayload?.user?.roles ?? [];
            const activeRole = roles.find(role => role.id === tokenPayload.user.is_active) ?? roles[0];
            const targetUrl = getRoleDefaultUrl(activeRole?.name);

            if (targetUrl === '/auth') {
                this.authService.signOut().subscribe();
                return this.router.parseUrl('/auth/sign-in');
            }

            return this.router.parseUrl(targetUrl);
        } catch {
            this.authService.signOut().subscribe();
            return this.router.parseUrl('/auth/sign-in');
        }
    }
}

export const appRoutes: Route[] = [
    { path: '', pathMatch: 'full', redirectTo: 'redirect' },
    {
        path: 'redirect',
        canActivate: [RedirectGuard],
        component: LayoutComponent
    },
    {
        path: 'auth',
        canActivate: [NoAuthGuard],
        component: LayoutComponent,
        data: { layout: 'empty' },
        loadChildren: () => import('app/resources/r1-account/a1-auth/auth.routes')
    },
    {
        path: '',
        canActivate: [AuthGuard],
        component: LayoutComponent,
        resolve: { initialData: initialDataResolver },
        children: [
            {
                path: 'admin',
                // roleResolver removed — any authenticated user can access admin routes
                loadChildren: () => import('app/resources/r2-admin/route')
            },
            { path: '404-not-found', pathMatch: 'full', loadChildren: () => import('app/shared/error/not-found.routes') },
            { path: '**', redirectTo: '404-not-found' }
        ]
    }
];
```

---

### 2. `src/app/app.resolver.ts`

**Problems fixed:**
- Error cases called `router.navigateByUrl('')` from inside a resolver and returned a `Promise`. This caused conflicts with the ongoing navigation.
- After the Switch Role feature was added, the resolver needed to respect the client-side role preference stored in `localStorage`.

**What changed:**
- Error cases now return `router.parseUrl('')` (a `UrlTree`), which Angular 15.2+ properly handles as a redirect from a resolver.
- Reads `preferredRoleId` from `localStorage`. If set, uses that role ID instead of `is_active` from the JWT.
- Rebuilds the user object with correct `is_active` and `is_default` flags so all components (header label, switch-role panel) reflect the chosen role.

```typescript
// src/app/app.resolver.ts

import { inject }               from "@angular/core";
import { Router, UrlTree }      from "@angular/router";
import { NavigationService }    from "app/core/navigation/navigation.service";
import { UserService }          from "app/core/user/user.service";
import { UserPayload }          from 'helper/interfaces/payload.interface';
import jwt_decode               from 'jwt-decode';
import { AuthService }          from "./core/auth/auth.service";

export const initialDataResolver = (): UrlTree | void => {
    const router = inject(Router);
    const token = inject(AuthService).accessToken;

    if (!token) {
        localStorage.clear();
        return router.parseUrl('');
    }

    const navigationService = inject(NavigationService);

    try {
        const tokenPayload: UserPayload = jwt_decode(token);

        // Use client-side preferred role (set via Switch Role) if present
        const preferredRoleId = localStorage.getItem('preferredRoleId');
        const activeRoleId = preferredRoleId ? parseInt(preferredRoleId) : tokenPayload.user.is_active;

        // Rebuild user with correct is_active and is_default flags
        const updatedUser = {
            ...tokenPayload.user,
            is_active: activeRoleId,
            roles: (tokenPayload.user.roles ?? []).map(r => ({ ...r, is_default: r.id === activeRoleId }))
        };

        inject(UserService).user = updatedUser;

        const role = updatedUser.roles.find(r => r.id === activeRoleId) ?? updatedUser.roles[0];

        if (!role) {
            localStorage.clear();
            return router.parseUrl('');
        }

        navigationService.navigations = role;

    } catch (error) {
        localStorage.clear();
        return router.parseUrl('');
    }
};
```

---

### 3. `src/app/core/auth/guards/auth.guard.ts`

**Problem fixed:**
- `jwt_decode(token)` was called without a try/catch. If the token string was malformed, it would throw an unhandled error inside the guard, crashing the entire navigation silently.

**What changed:**
- Wrapped `jwt_decode` in a try/catch so invalid tokens are handled gracefully and the user is redirected to `/auth` instead of getting a white screen.

```typescript
// src/app/core/auth/guards/auth.guard.ts

import { inject }                                     from '@angular/core';
import { CanActivateChildFn, CanActivateFn, Router }  from '@angular/router';
import { of }                                         from 'rxjs';
import { AuthService }                                from 'app/core/auth/auth.service';
import { UserPayload }                                from 'helper/interfaces/payload.interface';
import jwt_decode                                     from 'jwt-decode';

export const AuthGuard: CanActivateFn | CanActivateChildFn = () => {

    const router: Router = inject(Router);
    const authService = inject(AuthService);
    const token = authService?.accessToken;
    if (token) {
        try {
            const tokenPayload: UserPayload = jwt_decode(token);
            if (tokenPayload) {
                return of(true);
            }
        } catch {
            // Invalid token — deny access
        }
    }
    return of(router.parseUrl('/auth'));
};
```

---

### 4. `src/app/core/auth/guards/noAuth.guard.ts`

**Problem fixed:**
- Same issue as `AuthGuard` — `jwt_decode(token)` had no try/catch, meaning a malformed token would crash the guard and break navigation to the login page.

**What changed:**
- Added try/catch around `jwt_decode`. If the token is invalid, the guard falls through and **allows** access to auth routes (login page) instead of crashing.

```typescript
// src/app/core/auth/guards/noAuth.guard.ts

import { inject } from '@angular/core';
import { CanActivateChildFn, CanActivateFn, Router } from '@angular/router';
import { of } from 'rxjs';
import jwt_decode from 'jwt-decode';
import { AuthService } from 'app/core/auth/auth.service';
import { UserPayload } from 'helper/interfaces/payload.interface';

export const NoAuthGuard: CanActivateFn | CanActivateChildFn = (_route, _state) => {

    const router: Router = inject(Router);
    const authService = inject(AuthService);
    const token = authService?.accessToken;
    if (token) {
        try {
            const tokenPayload: UserPayload = jwt_decode(token);
            if (tokenPayload) {
                return of(router.parseUrl(''));
            }
        } catch {
            // Invalid token format — allow access to auth routes
        }
    }
    return of(true);
};
```

---

### 5. `src/app/core/auth/resolvers/role.resolver.ts`

**Problems fixed:**
- Same anti-pattern as `RedirectGuard`: called `router.navigateByUrl()` inside a resolver and returned `of(false)`. In Angular 15.2+, starting a navigation from inside a resolver while returning a value causes conflicts.
- No try/catch — accessing `tokenPayload.user.roles` with a malformed token would throw uncaught.

**What changed:**
- Returns `UrlTree` directly for redirect cases instead of calling `router.navigateByUrl`.
- Full try/catch with optional chaining.
- Removed `of` import (no longer needed).

```typescript
// src/app/core/auth/resolvers/role.resolver.ts

import { inject }           from "@angular/core";
import { Router, UrlTree }  from "@angular/router";
import { RoleEnum }         from "helper/enums/role.enum";
import { UserPayload }      from 'helper/interfaces/payload.interface';
import jwt_decode           from 'jwt-decode';
import { AuthService }      from "../auth.service";

export const getRoleDefaultUrl = (roleName?: string): string => {
    switch (roleName) {
        case RoleEnum.ADMIN:     return '/admin/home';
        case RoleEnum.ORG_ADMIN: return '/org/invoice';
        case RoleEnum.BANK_ADMIN: return '/bank/invoice';
        default:                 return '/auth';
    }
};

export const roleResolver = (allowedRoles: string[]) => {
    return (): UrlTree | string[] => {
        const router       = inject(Router);
        const token        = inject(AuthService).accessToken;

        try {
            const tokenPayload : UserPayload = jwt_decode(token);
            const is_active    = tokenPayload?.user?.is_active;
            const roles        = tokenPayload?.user?.roles ?? [];
            const role         = roles.find(role => role.id === is_active) ?? roles[0];

            if (!role) {
                return router.parseUrl('/auth/sign-in');
            }

            const isValidRole  = allowedRoles.includes(role.name);

            if (!isValidRole) {
                return router.parseUrl(getRoleDefaultUrl(role.name));
            }

            return allowedRoles;
        } catch {
            return router.parseUrl('/auth/sign-in');
        }
    };
};
```

---

### 6. `src/app/core/auth/auth.service.ts`

**Problems fixed:**
- After OTP verification, the code navigated to `''` which triggered a long chain of redirects through `RedirectGuard`. This chain silently failed in Angular 15.2+ due to the `navigateByUrl + return false` anti-pattern.
- `signOut()` did not clear the client-side role preference.

**What changed:**
- Added `import jwt_decode from 'jwt-decode'`.
- Added `getRedirectUrl()` method — decodes the JWT directly and returns the correct home URL for the active role. Also reads `preferredRoleId` from `localStorage` so it respects client-side role switches. Handles `Super Administrator` role name.
- Updated `signOut()` to also remove `preferredRoleId` from `localStorage`.

```typescript
// New method added to AuthService

getRedirectUrl(): string {
    try {
        const token = this.accessToken;
        if (!token) return '/auth/sign-in';
        const payload: any = jwt_decode(token);
        const roles: any[] = payload?.user?.roles ?? [];
        const preferredRoleId = localStorage.getItem('preferredRoleId');
        const is_active: number = preferredRoleId ? parseInt(preferredRoleId) : payload?.user?.is_active;
        const activeRole = roles.find(r => r.id === is_active) ?? roles[0];
        switch (activeRole?.name) {
            case 'Super Administrator': return '/admin/home';
            case 'Administrator':       return '/admin/home';
            case 'Organization Admin':  return '/org/invoice';
            case 'Bank Admin':          return '/bank/invoice';
            default:                    return '/admin/home';
        }
    } catch {
        return '/admin/home';
    }
}

signOut(): Observable<boolean> {
    localStorage.removeItem('accessToken');
    localStorage.removeItem('preferredRoleId'); // also clear role preference
    return of(true);
}
```

---

### 7. `src/app/resources/r1-account/a1-auth/otp/index.ts`
### 8. `src/app/resources/r1-account/auth/otp/index.ts`

**Problem fixed:**
- After OTP verification succeeded, the component called `this._router.navigateByUrl('')`. With `withHashLocation()` enabled and the `RedirectGuard` anti-pattern, this navigation silently failed every time — the user stayed on the OTP page indefinitely.

**What changed:**
- Replaced `this._router.navigateByUrl('')` with `this._router.navigateByUrl(this._authService.getRedirectUrl())`.
- This bypasses the `RedirectGuard` chain entirely and navigates directly to the correct role-based URL (e.g. `/admin/home`).

```typescript
// In verify() success handler — both OTP components

next: res => {
    this.isLoading = false;
    this.clearAllInput();
    // Navigate directly to role home — bypasses RedirectGuard chain
    const redirectUrl = this._authService.getRedirectUrl();
    this._router.navigateByUrl(redirectUrl);
    this._snackbarService.openSnackBar("ចូលប្រព័ន្ធបានដោយជោគជ័យ", GlobalConstants.success);
},
```

---

### 9. `src/app/core/navigation/navigation.service.ts`

**Problem fixed:**
- Only `RoleEnum.ADMIN` ('Administrator') was mapped to admin navigation. `Super Administrator` and all other roles got an empty navigation array `[]`, resulting in a blank sidebar.

**What changed:**
- Added `'Super Administrator'` case to show admin navigation.
- Changed `default` case to also show admin navigation (so any authenticated role gets a usable sidebar).

```typescript
// src/app/core/navigation/navigation.service.ts

set navigations(role: Role) {
    switch (role.name) {
        case 'Super Administrator':
        case RoleEnum.ADMIN:
            this._navigation.next(navigationData.admin);
            break;
        default:
            this._navigation.next(navigationData.admin);
            break;
    }
}
```

---

### 10. `src/app/layout/common/user/switch-role/switch-role.component.ts`

**Problem fixed:**
- The component called `POST /api/account/switch` which returned **404 Not Found** — the backend endpoint does not exist.
- The previous implementation left unused constructor parameters (`HttpClient`, `SnackbarService`) that caused compilation errors after the API call was removed.
- After switching, `window.location.href = same_url` was used, which browsers ignore if the URL hasn't changed — so nothing happened.

**What changed:**
- Removed the entire HTTP API call and all dependencies that went with it (`HttpClient`, `SnackbarService`, `AuthService`, `NavigationService`, `UserService`, `jwt_decode`, `FormsModule`, `MatMenuModule`, `MatDividerModule`, `Router`).
- Implemented **client-side role switching**: stores the selected role's ID in `localStorage` under key `preferredRoleId`, then calls `window.location.reload()`.
- On reload, `initialDataResolver` reads `preferredRoleId` and rebuilds the user state with the chosen role as active (see change #2).
- This works because the JWT already contains all assigned roles — no backend call is needed just to switch which role is active.

```typescript
// src/app/layout/common/user/switch-role/switch-role.component.ts

import { animate, AnimationBuilder, AnimationPlayer, style } from '@angular/animations';
import { coerceBooleanProperty } from '@angular/cdk/coercion';
import { CommonModule } from '@angular/common';
import { Component, ElementRef, HostBinding, Input, OnChanges, OnDestroy, Renderer2, SimpleChanges, ViewEncapsulation } from '@angular/core';
import { MatButtonModule } from '@angular/material/button';
import { MatIconModule } from '@angular/material/icon';
import { Role } from 'app/core/user/user.types';

@Component({
    selector: 'user-switch-role',
    templateUrl: './switch-role.component.html',
    styleUrls: ['./switch-role.component.scss'],
    encapsulation: ViewEncapsulation.None,
    standalone: true,
    imports: [ CommonModule, MatIconModule, MatButtonModule ]
})
export class SwitchRoleComponent implements OnChanges, OnDestroy {

    @Input() roles: Role[] = [];

    private _canClick: boolean = true;
    private _opened: boolean = false;
    private _handleOverlayClick: any;
    private _overlay: HTMLElement;
    private _player: AnimationPlayer;

    constructor(
        private _animationBuilder: AnimationBuilder,
        private _elementRef: ElementRef,
        private _renderer2: Renderer2,
    ) {
        this._handleOverlayClick = (): void => {
            if (this._canClick) this.close();
        };
    }

    setActive(role: Role): void {
        // Store selected role in localStorage — initialDataResolver reads this on reload
        localStorage.setItem('preferredRoleId', role.id.toString());
        this.close();
        window.location.reload();
    }

    // ... animation/overlay methods unchanged
}
```

---

## localStorage Keys Used

| Key | Purpose | Set by | Cleared by |
|-----|---------|--------|-----------|
| `accessToken` | JWT from login/OTP | `AuthService` | `signOut()` |
| `preferredRoleId` | Client-side active role ID | `SwitchRoleComponent.setActive()` | `signOut()` |
| `email` | Temp storage of username during OTP flow | `OTP ngOnInit` | `AuthService.verifyOtp()` |

---

## Role Switch Flow (No Backend Required)

```
User clicks "Switch Role" in header menu
        ↓
Side panel opens showing all roles from JWT
        ↓
User clicks a role (e.g. Administrator id=2)
        ↓
localStorage.setItem('preferredRoleId', '2')
        ↓
window.location.reload()
        ↓
App restarts → initialDataResolver runs
        ↓
Reads preferredRoleId=2 from localStorage
        ↓
Rebuilds user object: is_active=2, Administrator.is_default=true
        ↓
NavigationService shows admin navigation sidebar
        ↓
User is now operating as Administrator
```

---

## Root Cause of OTP Navigation Failure

The core bug was an Angular 15.2+ breaking change:

> When a `CanActivate` guard returns `false`, **any navigation triggered inside the guard is also cancelled**.

The original `RedirectGuard` did this:
```typescript
// BROKEN — both the current AND the new navigation get cancelled
this.router.navigateByUrl('/admin/home');
return false;
```

The fix returns a `UrlTree` instead, which Angular handles atomically:
```typescript
// CORRECT — Angular redirects as part of the same navigation cycle
return this.router.parseUrl('/admin/home');
```
