# ed-laravel-multi-lang-routes
Working with Multi-languages in Laravel Without Packages

by [article](https://medium.com/@styles77/working-with-multi-languages-in-laravel-without-packages-b4dc8bfa9ab8)

## Localization
[Laravel Doc](https://laravel.com/docs/10.x/localization#main-content)

## 1 Set up your supported languages

I have a fresh Laravel installed on my machine, and the first thing we will do is set up our supported languages. Create a new file in the config folder called languages.php.

```
<?php 

return [
    'en',
    'pt',
    'fr',
    'jp'
];
```

Done. Now we will use this config file to work across the app.

## 2. Create multi-language routes

According to the Laravel’s documentation: “By default, the Laravel application skeleton does not include the lang directory. If you would like to customize Laravel’s language files or create your own, you should scaffold the lang directory via the lang:publish Artisan command.”

```
php artisan lang:publish
```

Now, let’s create routes that support all languages. We will loop through the supported languages to create unique routes for every language.

But before that, let’s create some folders for our supported languages in the lang directory. I will name the file paths.php since it will store all of our route paths.

The folder will look something like this:

```
├── lang/       
│   ├── en/
│   │   ├── paths.php
│   ├── fr/
│   │   ├── paths.php
│   ├── jp/
│   │   ├── paths.php
│   ├── pt/
│   │   ├── paths.php
```

The content inside the paths.php is:

**lang/en/paths.php**

```
<?php
return [
    'about' => 'about',
];
```

**lang/fr/paths.php**

```
<?php
return [
    'about' => 'a-propos',
];
```

**lang/jp/paths.php**

```
<?php
return [
    'about' => 'about',
];
```

**lang/pt/paths.php**

```
<?php
return [
    'about' => 'sobre',
];
```

Now, we’ll define our routes in web/routes.php:

```
foreach (config('languages') as $locale) {
    if (config('app')['locale'] == $locale) {
        Route::get(__('paths.about', [], $locale), AboutController::class)->name("about.{$locale}");
    } else {
        Route::get("{$locale}/".__('paths.about', [], $locale), AboutController::class)->name("about.{$locale}");
    }
}
```

In the code above, you will register some new routes for all available languages:
![image](https://github.com/GrytsenkoAndrey/ed-laravel-multi-lang-routes/assets/63291871/b8a76639-170c-4ae3-932b-9614b57964dc)

If you want to access the path in the English version use /about but if you want to access the path in other languages use /pt/sobre for example.

## 3. Automatically translate the page into the selected language

Now, you probably notice that, when you access the page, the translation won’t be applied. That’s because we haven’t set our locale according to the path.

Let’s create a middleware to translate the page into the selected locale.

```
php artisan make:middleware SetLocale
```

Paste the following code for the handle method in the app/Http/Middleware/SetLocale.php

```
public function handle(Request $request, Closure $next): Response
{
    $lang = $request->segment(1);

    if(strlen($lang) === 2 && in_array($lang, config('languages'))){
        app()->setLocale($lang);
    }

    return $next($request);
}
```

The code above will set the language according to segment 1 in the request, if segment 1 is equal to our supported languages we will set it as the current language.

Now register the middleware in the app/Http/Kernel.php

```
protected $routeMiddleware = [
    ...
    'set-locale' => \App\Http\Middleware\SetLocale::class,
];
```

Update our routes to use the middleware:

```
Route::group([
    'middleware' => 'set-locale'
], function() {
    foreach (config('languages') as $locale) {
        if (config('app')['locale'] == $locale) {
            Route::get(__('paths.about', [], $locale), AboutController::class)->name("about.{$locale}");
        } else {
            Route::get("{$locale}/".__('paths.about', [], $locale), AboutController::class)->name("about.{$locale}");
        }
    }
});
```

After that, we are ready to print the page according to the selected language. For example, I have a very simple controller that prints “Hello World” using Laravel Localization.

```
class AboutController extends Controller
{
    public function __invoke(Request $request)
    {
        return __("general.hello_world");
    }
}
```

Before we see the output, we need to create new translation files called “general” for all languages. If you don’t create for all languages, the fallback language will be using the English version.

```
├── lang/       
│   ├── en/
│   │   ├── general.php
│   │   ├── paths.php
│   ├── fr/
│   │   ├── general.php
│   │   ├── paths.php
│   ├── jp/
│   │   ├── general.php
│   │   ├── paths.php
│   ├── pt/
│   │   ├── general.php
│   │   ├── paths.php
```

**lang/en/general.php**

```
<?php
return [
    'hello_world' => 'Hello World',
];
```

**lang/fr/paths.php**

```
<?php
return [
    'hello_world' => 'Bonjour le monde',
];
```

You can continue to apply this to the rest of the language.

## 4. Design and implement a translatable database

Let's say you have normal tables for categories and posts with the following structure:

```
categories
- id
- name
- slug

posts
- id
- category_id
- title
- slug
- content
```

I know that you can simply add a locale or language column in both tables, but that will cause inconsistency for the category_id. For example:

```
categories

id: 1
name: "News EN"
slug: "news-en"
locale: "en"

id: 2
name: "Nouvelles"
slug: "nouvelles"
locale: "fr"

posts

id: 1
category_id: 1
title: "Post English"
slug: "post-english"
locale: "en"
content: "Content post in English"

id: 2
category_id: 2
title: "Post France"
slug: "post-france"
locale: "fr"
content: "Content post in France"
```

As you can see in the plain text above, both categories cover the same meaning, which is “News” but they have different IDs. If you change the category_id for “Post France” to 1, it will cause a problem because the France post is referenced to the English category.

Let’s solve that problem by creating a third table for categories called category_translations and changing some columns in the categories table.

```
categories
- id

category_translations
- id
- category_id
- name
- slug
```

Implement new table structure:

```
categories

id: 1
```

**category_translations**

```
id: 1
category_id: 1
name: "News EN"
slug: "news-en"
locale: "en"

id: 2
category_id: 1
name: "Nouvelles"
slug: "nouvelles"
locale: "fr"
```

**posts**

```
id: 1
category_id: 1
title: "Post English"
slug: "post-english"
locale: "en"
content: "Content post in English"

id: 2
category_id: 1
title: "Post France"
slug: "post-france"
locale: "fr"
content: "Content post in France"
```

Now both posts are referenced to the same ID and we can display the category in English or French.
Let’s Code it in Laravel

I will focus on how we take advantage of the Laravel Relationship for categories and category_translations.

First, we define the migration files:

```
php artisan make:migration create_categories_table

Schema::create('categories', function (Blueprint $table) {
    $table->id();
    $table->timestamps();
});

php artisan make:migration create_category_translations_table

Schema::create('category_translations', function (Blueprint $table) {
    $table->id();
    $table->foreignId('category_id');
    $table->string('name');
    $table->string('slug');
    $table->string('locale');
    $table->timestamps();
});
```

If you don’t know how to make a migration, check out this documentation.

Now, let’s create our models:

```
php artisan make:model Category

php artisan make:model CategoryTranslation
```

We will give our Category model access to the CategoryTranslation by creating a relationship:

```
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Category extends Model
{
    use HasFactory;

    protected $fillable = [
        'id',
    ];

    public function translations()
    {
        return $this->hasMany(CategoryTranslation::class);
    }

    public function en()
    {
        return $this->hasOne(CategoryTranslation::class)->where('locale', 'en');
    }

    public function fr()
    {
        return $this->hasOne(CategoryTranslation::class)->where('locale', 'fr');
    }

    public function pt()
    {
        return $this->hasOne(CategoryTranslation::class)->where('locale', 'pt');
    }

    public function jp()
    {
        return $this->hasOne(CategoryTranslation::class)->where('locale', 'jp');
    }
}
```

In the code above, when you select a single category, you can access the English version of the category using **$category->en** as well as other versions like **$category->fr, $category->jp,** etc


![image](https://github.com/GrytsenkoAndrey/ed-laravel-multi-lang-routes/assets/63291871/aac257b5-b70f-4f9c-a5ed-dc66fa25acad)


And you can also be able to access the whole version of the category translations using **$category->translations.**


![image](https://github.com/GrytsenkoAndrey/ed-laravel-multi-lang-routes/assets/63291871/14db1516-cee0-4075-aa90-bf377fc13e4a)

