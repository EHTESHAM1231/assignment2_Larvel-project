# Skill Swap Hub - CHT2520 Assignment 2
## Muhammad Ehtesham Siddiqui (U2366447)

## 1. Introduction

This project extends the Assignment 1 Skill Swap Hub application with advanced Laravel features. The original application allowed users to share skills they can teach others. Assignment 2 adds authentication, authorization, OAuth social login, database relationships, and a modern responsive UI using Tailwind CSS.

### New Features Implemented
- User authentication with Laravel Breeze
- OAuth 2.0 social login (Google & GitHub) via Laravel Socialite
- Authorization policies for resource protection
- Eloquent relationships (User → SkillOffers, Category → SkillOffers)
- Database normalization with categories
- Advanced search and filtering
- Responsive Tailwind CSS design
- User dashboard with statistics

---

## 2. Authentication & Authorization

### 2.1 Laravel Breeze Authentication

**Implementation**: Laravel Breeze with Blade templates  
**Location**: `app/Http/Controllers/Auth/`, `resources/views/auth/`

Laravel Breeze provides a minimal, clean authentication scaffolding including registration, login, password reset, and email verification.

```php
// routes/web.php - Protected routes
Route::middleware('auth')->group(function () {
    Route::get('/skill-offers/create', [SkillOfferController::class, 'create']);
    Route::post('/skill-offers', [SkillOfferController::class, 'store']);
    // ... other protected routes
});
```

**Critical Analysis**:
- **Chosen because**: Lightweight, follows Laravel conventions, includes Tailwind CSS
- **Problems solved**: Secure user registration, session management, CSRF protection
- **Limitations**: Basic authentication only; no built-in 2FA or advanced security features

### 2.2 OAuth Social Login (Laravel Socialite)

**Implementation**: Laravel Socialite for Google and GitHub OAuth  
**Location**: `app/Http/Controllers/Auth/SocialiteController.php`

```php
// SocialiteController.php
public function callback(string $provider): RedirectResponse
{
    $socialUser = Socialite::driver($provider)->user();
    
    // Find or create user
    $user = User::where('provider', $provider)
        ->where('provider_id', $socialUser->getId())
        ->first();

    if (!$user) {
        $user = User::create([
            'name' => $socialUser->getName(),
            'email' => $socialUser->getEmail(),
            'provider' => $provider,
            'provider_id' => $socialUser->getId(),
            'avatar' => $socialUser->getAvatar(),
        ]);
    }

    Auth::login($user, true);
    return redirect()->intended(route('dashboard'));
}
```

**Critical Analysis**:
- **Chosen because**: Simplifies OAuth implementation, supports multiple providers
- **Problems solved**: Users can login without creating passwords, improved UX
- **Limitations**: Requires external API credentials, dependent on third-party services

### 2.3 Authorization Policies

**Implementation**: Laravel Policies for resource-level authorization  
**Location**: `app/Policies/SkillOfferPolicy.php`

```php
// SkillOfferPolicy.php
public function update(User $user, SkillOffer $skillOffer): bool
{
    return $user->id === $skillOffer->user_id;
}

public function delete(User $user, SkillOffer $skillOffer): bool
{
    return $user->id === $skillOffer->user_id;
}
```

**Critical Analysis**:
- **Chosen because**: Clean separation of authorization logic, reusable across controllers
- **Problems solved**: Prevents unauthorized editing/deletion of other users' content
- **Limitations**: Requires manual policy registration for new models

---

## 3. Database Relationships & Normalization

### 3.1 Eloquent Relationships

**Implementation**: One-to-Many relationships  
**Location**: `app/Models/`

```php
// User.php
public function skillOffers()
{
    return $this->hasMany(SkillOffer::class);
}

// SkillOffer.php
public function user()
{
    return $this->belongsTo(User::class);
}

public function category()
{
    return $this->belongsTo(Category::class);
}

// Category.php
public function skillOffers()
{
    return $this->hasMany(SkillOffer::class);
}
```

### 3.2 Database Schema

```php
// Migration: add_user_id_to_skill_offers_table.php
Schema::table('skill_offers', function (Blueprint $table) {
    $table->foreignId('user_id')->nullable()->constrained()->onDelete('cascade');
});

// Migration: create_categories_table.php
Schema::create('categories', function (Blueprint $table) {
    $table->id();
    $table->string('name')->unique();
    $table->string('slug')->unique();
    $table->text('description')->nullable();
    $table->string('color', 7)->default('#3B82F6');
    $table->timestamps();
});
```

**Critical Analysis**:
- **Chosen because**: Normalizes data, enables efficient queries with eager loading
- **Problems solved**: Data redundancy, referential integrity, organized skill categorization
- **Limitations**: Increased query complexity, requires careful eager loading to avoid N+1 problems

---

## 4. Frontend Implementation

### 4.1 Tailwind CSS

**Implementation**: Tailwind CSS 4.0 with Vite  
**Location**: `resources/css/app.css`, `tailwind.config.js`

The UI uses Tailwind's utility classes for responsive, modern design:

```html
<!-- Responsive grid for skill cards -->
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
    @foreach ($skillOffers as $offer)
        <div class="bg-gray-50 rounded-lg border border-gray-200 
                    overflow-hidden hover:shadow-lg transition-shadow duration-200">
            <!-- Card content -->
        </div>
    @endforeach
</div>
```

**Critical Analysis**:
- **Chosen because**: Utility-first approach, highly customizable, excellent documentation
- **Problems solved**: Consistent styling, responsive design, rapid development
- **Limitations**: HTML can become verbose, learning curve for utility classes

### 4.2 Alpine.js Interactivity

**Implementation**: Alpine.js for lightweight JavaScript interactions  
**Location**: Included via Laravel Breeze

```html
<!-- Mobile navigation toggle -->
<nav x-data="{ open: false }">
    <button @click="open = ! open">Menu</button>
    <div :class="{'block': open, 'hidden': ! open}">
        <!-- Navigation items -->
    </div>
</nav>
```

---

## 5. Advanced Features

### 5.1 Search and Filtering

```php
// SkillOfferController.php - index method
public function index(Request $request)
{
    $query = SkillOffer::with(['user', 'category']);

    if ($request->filled('search')) {
        $search = $request->input('search');
        $query->where(function ($q) use ($search) {
            $q->where('skill_name', 'like', "%$search%")
              ->orWhere('name', 'like', "%$search%");
        });
    }

    if ($request->filled('category')) {
        $query->where('category_id', $request->input('category'));
    }

    if ($request->filled('level')) {
        $query->where('skill_level', $request->input('level'));
    }

    return view('skill_offers.index', [
        'skillOffers' => $query->paginate(6)->withQueryString(),
        'categories' => Category::orderBy('name')->get()
    ]);
}
```

### 5.2 User Dashboard

The dashboard displays personalized statistics and the user's skill offers:

```php
// dashboard.blade.php
<div class="grid grid-cols-1 md:grid-cols-3 gap-6">
    <div class="bg-white p-6 rounded-lg shadow">
        <div class="text-2xl font-bold text-indigo-600">
            {{ Auth::user()->skillOffers()->count() }}
        </div>
        <div class="text-sm text-gray-500">Skills Shared</div>
    </div>
</div>
```

---

## 6. Installation & Setup

### Prerequisites
- PHP 8.2+
- Composer
- Node.js & npm
- SQLite (or MySQL/PostgreSQL)

### Installation Steps

```bash
# 1. Clone the repository
git clone [your-repository-url]
cd larvel-project

# 2. Install PHP dependencies
composer install

# 3. Install Node dependencies
npm install

# 4. Environment setup
cp .env.example .env
php artisan key:generate

# 5. Database setup (SQLite)
touch database/database.sqlite
php artisan migrate --seed

# 6. Build frontend assets
npm run build

# 7. Start the development server
php artisan serve
```

### OAuth Configuration (Optional)

To enable Google/GitHub login, add credentials to `.env`:

```env
# Google OAuth (https://console.cloud.google.com/apis/credentials)
GOOGLE_CLIENT_ID=your-client-id
GOOGLE_CLIENT_SECRET=your-client-secret
GOOGLE_REDIRECT_URI=http://localhost:8000/auth/google/callback

# GitHub OAuth (https://github.com/settings/developers)
GITHUB_CLIENT_ID=your-client-id
GITHUB_CLIENT_SECRET=your-client-secret
GITHUB_REDIRECT_URI=http://localhost:8000/auth/github/callback
```

---

## 7. Project Structure

```
larvel-project/
├── app/
│   ├── Http/Controllers/
│   │   ├── Auth/
│   │   │   └── SocialiteController.php    # OAuth handling
│   │   ├── ProfileController.php
│   │   └── SkillOfferController.php       # Main CRUD controller
│   ├── Models/
│   │   ├── Category.php                   # Category model
│   │   ├── SkillOffer.php                 # Skill offer model
│   │   └── User.php                       # User model with OAuth
│   └── Policies/
│       └── SkillOfferPolicy.php           # Authorization policy
├── database/
│   ├── migrations/                        # Database migrations
│   └── seeders/                           # Sample data seeders
├── resources/views/
│   ├── auth/                              # Authentication views
│   ├── layouts/                           # Layout templates
│   ├── profile/                           # Profile management
│   └── skill_offers/                      # Skill offer views
└── routes/
    ├── auth.php                           # Auth routes (Breeze)
    └── web.php                            # Application routes
```

---

## 8. Conclusion

This assignment demonstrates proficiency in:
- **Authentication**: Laravel Breeze with session-based auth
- **OAuth Integration**: Laravel Socialite for social login
- **Authorization**: Policy-based access control
- **Eloquent ORM**: Relationships and eager loading
- **Database Design**: Normalized schema with foreign keys
- **Modern Frontend**: Tailwind CSS responsive design
- **Best Practices**: MVC architecture, validation, security

The Skill Swap Hub now provides a complete, secure platform for users to share and discover skills within their community.