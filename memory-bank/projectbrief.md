# Akaunting Roles Module Fix

## Project Overview

Akaunting is a Laravel-based accounting software platform with a modular architecture. The Roles module provides role-based access control (RBAC) functionality for managing user permissions and roles.

## Issue Description

The Roles module menu items were not working when clicked:

- "Vai trò" (Roles) menu item not functioning
- "Quyền" (Permissions) menu item not functioning
- "Phân quyền người dùng" (User Roles) menu item not functioning

## Root Cause Analysis

1. **Route Registration Issue**: The Roles module was not using the proper Akaunting route macro system. It was loading routes directly without the required `{company_id}` prefix and admin middleware.

2. **Missing View Files**: Critical view files were missing for the permissions and user-roles functionality:
    - `modules/Roles/Resources/views/permissions/index.blade.php`
    - `modules/Roles/Resources/views/permissions/create.blade.php`
    - `modules/Roles/Resources/views/user-roles/index.blade.php`
    - `modules/Roles/Resources/views/user-roles/create.blade.php`
    - `modules/Roles/Resources/views/user-roles/edit.blade.php`

## Solution Implemented

1. **Fixed Route Registration**: Modified `modules/Roles/Providers/Main.php` to use the proper `Route::admin()` macro
2. **Created Missing Views**: Implemented all missing view files following Akaunting's UI patterns
3. **Fixed Route Configuration**: Updated route definitions to work with the macro system
4. **Created Missing Permissions**: Added all required permissions for Roles module and assigned them to admin role
5. **Added Authorization Middleware**: Added permission middleware to all controllers to enforce access control

### Permissions Created:

- `create-roles-roles` - Tạo vai trò
- `read-roles-roles` - Xem vai trò
- `update-roles-roles` - Sửa vai trò
- `delete-roles-roles` - Xóa vai trò
- `create-roles-permissions` - Tạo quyền
- `read-roles-permissions` - Xem quyền
- `update-roles-permissions` - Sửa quyền
- `delete-roles-permissions` - Xóa quyền
- `create-roles-user-roles` - Phân quyền người dùng
- `read-roles-user-roles` - Xem phân quyền người dùng
- `update-roles-user-roles` - Sửa phân quyền người dùng
- `delete-roles-user-roles` - Xóa phân quyền người dùng

All 12 permissions have been assigned to the admin role (ID: 1).

## Technical Details

- **Framework**: Laravel with Akaunting's custom route system
- **Module Structure**: Standard Akaunting module structure
- **UI Framework**: Akaunting's custom Blade components (x-layouts.admin, x-form, x-table)
- **Icons**: Material Icons (not Heroicons) - uses `<span class="material-icons-outlined">icon_name</span>`
- **Language**: Vietnamese (vi-VN) translations already in place

## Recent Fix (2025-07-19)

### Issue: 500 Error on User Roles Page

**Error**: `Unable to locate a class or view for component [heroicon-o-pencil]`

**Root Cause**: The user-roles/index.blade.php view was using Heroicons (`<x-heroicon-o-pencil>`, `<x-heroicon-o-trash>`) but Akaunting uses Material Icons.

**Solution**:

- Replaced `<x-heroicon-o-pencil class="w-4 h-4" />` with `<span class="material-icons-outlined text-lg">edit</span>`
- Replaced `<x-heroicon-o-trash class="w-4 h-4" />` with `<span class="material-icons-outlined text-lg">delete</span>`
- Fixed `x-delete-link` component usage - changed from `href="{{ route() }}"` to `:model="$user" route="roles.user-roles.destroy"`
- Cleared configuration and view cache

### Additional Issue: x-delete-link Component Usage

**Error**: `Call to a member function getTable() on bool` at DeleteLink.php:72

**Root Cause**: The `x-delete-link` component requires `:model` attribute to pass the model object, not just `href`.

**Fix**: Changed `<x-delete-link href="{{ route('roles.user-roles.destroy', $user->id) }}">` to `<x-delete-link :model="$user" route="roles.user-roles.destroy" />`

**Files Modified**:

- `modules/Roles/Resources/views/user-roles/index.blade.php`

### Additional Issue: Missing Role View Files

**Error**: `View [roles.show] not found`

**Root Cause**: Missing view files for roles show and edit functionality:

- `modules/Roles/Resources/views/roles/show.blade.php`
- `modules/Roles/Resources/views/roles/edit.blade.php`

**Solution**: Created both missing view files with comprehensive UI following Akaunting patterns:

**Created Files**:

- `modules/Roles/Resources/views/roles/show.blade.php` - Role detail view with permissions display and user assignments
- `modules/Roles/Resources/views/roles/edit.blade.php` - Role edit form with grouped permissions and interactive checkboxes

### Additional Issue: Database Schema Issue

**Error**: `SQLSTATE[42S22]: Column not found: 1054 Unknown column '08q_user_roles.user_type' in 'field list'`

**Root Cause**: The Role model's relationship with users is trying to access a `user_type` column that doesn't exist in the `user_roles` table. This appears to be a Laratrust configuration issue.

**Temporary Solution**:

- Changed `$role->users->count()` to `$role->users()->count()` to use query builder instead of eager loading
- Temporarily disabled assigned users display sections in both show and edit views to prevent the relationship error
- Added comments explaining the temporary nature of this fix

**Files Modified**:

- `modules/Roles/Resources/views/roles/show.blade.php` - Fixed users count and disabled users list
- `modules/Roles/Resources/views/roles/edit.blade.php` - Fixed users count and disabled users preview

**Note**: This is a temporary fix. The proper solution would be to fix the Laratrust configuration or database schema to include the missing `user_type` column.
