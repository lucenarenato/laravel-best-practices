![Boas Práticas Laravel](/images/logo-english.png?raw=true)

O que é descrito aqui não é uma adaptação ao principio SOLID, padrões e etc. Aqui você irá encontrar as melhores práticas que geralmente são ignoradas em um projeto Laravel na vida real.

## Conteúdo

[Princípio da responsabilidade única](#princípio-da-responsabilidade-única)

[Models gordos, controllers finos](#models-gordos-controllers-finos)

[Validação](#validação)

[Lógica de negócio deve ser posta em classes](#lógica-de-negócio-deve-ser-posta-em-classes)

[Não se repita (Don't repeat yourself: DRY)](#não-se-repita-dont-repeat-yourself-dry)

[Usar o Eloquent em vez de Query Builder e consultas SQL puras (raw SQL). Usar collections no lugar de  arrays](#usar-o-eloquent-em-vez-de-query-builder-e-consultas-sql-puras-raw-sql-usar-collections-no-lugar-de--arrays)

[Mass assignment](#mass-assignment)

[Não executar consultas no Blade templates e usar eager loading (N + 1)](#não-executar-consultas-no-blade-templates-e-usar-eager-loading-n--1)

[Comente seu código, mas prefira um método descritivo e nomes de variáveis em vez de comentários](#comente-seu-código-mas-prefira-um-método-descritivo-e-nomes-de-variáveis-em-vez-de--comentários)

[Não coloque JS e CSS em templates Blade. Não coloque HTML em classes PHP](#não-coloque-js-e-css-em-templates-blade-não-coloque-html-em-classes-php)

[Use arquivos de linguagem e configuração. Constantes em vez de texto no código](#use-arquivos-de-linguagem-e-configuração-constantes-em-vez-de-texto-no-código)

[Use ferramentas padrões do Laravel aceitas pela comunidade](#use-ferramentas-padrões-do-laravel-aceitas-pela-comunidade)

[Siga a conversão de nomes usada no Laravel](#siga-a-conversão-de-nomes-usada-no-laravel)

[Tente sempre usar sintaxes pequenas e legíveis](#tente-sempre-usar-sintaxes-pequenas-e-legíveis)

[Use contaneirs IoC (inversão de controle) ou facades no lugar de classes](#use-contaneirs-ioc-inversão-de-controle-ou-facades-no-lugar-de-classes)

[Não recupere informaçẽos diretamente do `.env`](#não-recupere-informaçẽos-diretamente-do-env)

[Armaze datas em formatoes padrões. Use "accessors" and "mutators" para modificar o formato das datas](#armaze-datas-em-formatoes-padrões-use-accessors-and-mutators-para-modificar-o-formato-das-datas)

[Outras boas práticas](#outras-boas-práticas)

### **Princípio da responsabilidade única**

Classes e metódos devem possui somente uma responsabilidade.

Ruim:

```php
public function getFullNameAttribute()
{
    if (auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified()) {
        return 'Sr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
    } else {
        return $this->first_name[0] . '. ' . $this->last_name;
    }
}
```

Bom:

```php
public function getFullNameAttribute()
{
    return $this->isVerifiedClient() ? $this->getFullNameLong() : $this->getFullNameShort();
}

public function isVerifiedClient()
{
    return auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified();
}

public function getFullNameLong()
{
    return 'Sr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
}

public function getFullNameShort()
{
    return $this->first_name[0] . '. ' . $this->last_name;
}
```

[🔝 Voltar para o início](#conteúdo)

### **Models gordos, controllers finos**

Coloque toda a lógica relacionada a banco em modelos Eloquent ou em repositórios caso você esteja usando Query Builder ou consultas SQL.

Ruim:

```php
public function index()
{
    $clients = Client::verified()
        ->with(['orders' => function ($q) {
            $q->where('created_at', '>', Carbon::today()->subWeek());
        }])
        ->get();

    return view('index', ['clients' => $clients]);
}
```

Bom:

```php
public function index()
{
    return view('index', ['clients' => $this->client->getWithNewOrders()]);
}

class Client extends Model
{
    public function getWithNewOrders()
    {
        return $this->verified()
            ->with(['orders' => function ($q) {
                $q->where('created_at', '>', Carbon::today()->subWeek());
            }])
            ->get();
    }
}
```

[🔝 Voltar para o início](#conteúdo)

### **Validação**

Não use validações em controllers e sim em classes de Requisição.

Ruim:

```php
public function store(Request $request)
{
    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);

    ....
}
```

Bom:

```php
public function store(PostRequest $request)
{    
    ....
}

class PostRequest extends Request
{
    public function rules()
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
            'publish_at' => 'nullable|date',
        ];
    }
}
```

[🔝 Voltar para o início](#conteúdo)

### **Lógica de negócio deve ser posta em classes**

Controller devem ter somente uma responsabilidade, então mova lógica de negócio para outros serviços.

Ruim:

```php
public function store(Request $request)
{
    if ($request->hasFile('image')) {
        $request->file('image')->move(public_path('images') . 'temp');
    }

    ....
}
```

Bom:

```php
public function store(Request $request)
{
    $this->articleService->handleUploadedImage($request->file('image'));

    ....
}

class ArticleService
{
    public function handleUploadedImage($image)
    {
        if (!is_null($image)) {
            $image->move(public_path('images') . 'temp');
        }
    }
}
```

[🔝 Voltar para o início](#conteúdo)

### **Não se repita (Don't repeat yourself: DRY)**

Reutilize seu código sempre que possível. A ideia da responsabilidade única ajuda você a evitar duplicação. Isso serve também para templates Blade.

Ruim:

```php
public function getActive()
{
    return $this->where('verified', 1)->whereNotNull('deleted_at')->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->where('verified', 1)->whereNotNull('deleted_at');
        })->get();
}
```

Bom:

```php
public function scopeActive($q)
{
    return $q->where('verified', 1)->whereNotNull('deleted_at');
}

public function getActive()
{
    return $this->active()->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->active();
        })->get();
}
```

[🔝 Voltar para o início](#conteúdo)

### **Usar o Eloquent em vez de Query Builder e consultas SQL puras (raw SQL). Usar collections no lugar de  arrays**

Eloquent permite que você escreva código legível e mantível mais fácil. Além disso, Eloquent possui ferramentas ótimas para implementar "soft deletes", eventos, escopos e etc.

Ruim:

```sql
SELECT *
FROM `articles`
WHERE EXISTS (SELECT *
              FROM `users`
              WHERE `articles`.`user_id` = `users`.`id`
              AND EXISTS (SELECT *
                          FROM `profiles`
                          WHERE `profiles`.`user_id` = `users`.`id`)
              AND `users`.`deleted_at` IS NULL)
AND `verified` = '1'
AND `active` = '1'
ORDER BY `created_at` DESC
```

Bom:

```php
Article::has('user.profile')->verified()->latest()->get();
```

[🔝 Voltar para o início](#conteúdo)

### **Mass assignment**

Ruim:

```php
$article = new Article;
$article->title = $request->title;
$article->content = $request->content;
$article->verified = $request->verified;
// Adicionar categoria em artigos
$article->category_id = $category->id;
$article->save();
```

Bom:

```php
$category->article()->create($request->all());
```

[🔝 Voltar para o início](#conteúdo)

### **Não executar consultas no Blade templates e usar eager loading (N + 1)**

Ruim (para 100 usuários, 101 consultas são feitas):

```php
@foreach (User::all() as $user)
    {{ $user->profile->name }}
@endforeach
```

Bom (para 100 usuários, duas consultas são feitas):

```php
$users = User::with('profile')->get();

...

@foreach ($users as $user)
    {{ $user->profile->name }}
@endforeach
```

[🔝 Voltar para o início](#conteúdo)

### **Comente seu código, mas prefira um método descritivo e nomes de variáveis em vez de  comentários**

Ruim:

```php
if (count((array) $builder->getQuery()->joins) > 0)
```

Melhor:

```php
// Determine if there are any joins.
if (count((array) $builder->getQuery()->joins) > 0)
```

Bom:

```php
if ($this->hasJoins())
```

[🔝 Voltar para o início](#conteúdo)

### **Não coloque JS e CSS em templates Blade. Não coloque HTML em classes PHP**

Ruim:

```php
let article = `{{ json_encode($article) }}`;
```

Melhor:

```php
<input id="article" type="hidden" value="{{ json_encode($article) }}">

Ou

<button class="js-fav-article" data-article="{{ json_encode($article) }}">{{ $article->name }}<button>
```

No javascript:

```php
let article = $('#article').val();
```


[🔝 Voltar para o início](#conteúdo)

### **Use arquivos de linguagem e configuração. Constantes em vez de texto no código**

Ruim:

```php
public function isNormal()
{
    return $article->type === 'normal';
}

return back()->with('message', 'Your article has been added!');
```

Bom:

```php
public function isNormal()
{
    return $article->type === Article::TYPE_NORMAL;
}

return back()->with('message', __('app.article_added'));
```

[🔝 Voltar para o início](#conteúdo)

### **Use ferramentas padrões do Laravel aceitas pela comunidade**

Preferir usar funcionalidades do próprio Laravel e pacotes da comunidade em vez de pacotes de terceiros. Qualquer desenvolvedor que irá trabalhar em seu sistema terá que aprender novas ferramentas no futuro. Além disso, ter ajuda da comunidade do Laravel se torna significativamente menor quando você utiliza um pacote ou ferramenta de terceiros.

Tarefas | Ferramentas padrões | Pacotes de terceiros
------------ | ------------- | -------------
Autorização | Policies | Entrust, Sentinel e outros pacotes
Compilar assets | Laravel Mix | Grunt, Gulp, pacotes de terceiros
Ambiente de desenvolvimento | Homestead | Docker
Deployment | Laravel Forge | Deployer e outras soluções
Testes unitários | PHPUnit, Mockery | Phpspec
Teste em navegador | Laravel Dusk | Codeception
DB | Eloquent | SQL, Doctrine
Templates | Blade | Twig
Trabalhando com dados | Laravel collections | Arrays
Validação de formulários | Request classes | pacotes de terceiros, validação no controller
Autenticação | Nativo | pacotes de terceiros, sua propria solução
Autenticação API | Laravel Passport | JWT e pacotes OAuth 
Criar API | Nativo | Dingo API e similares
Trabalhando com estrutura de DB | Migrações | Trabalhar com banco diretamente
Localização | Nativo | pacotes de terceiros
Interface em tempo real | Laravel Echo, Pusher | pacotes de terceiros e trabalhar com WebSockets diretamente
Gerar dados de teste | Seeder classes, Model Factories, Faker | Criar testes manualmente
Agendar tarefas | Laravel Task Scheduler | Scripts e pacotes de terceiros
DB | MySQL, PostgreSQL, SQLite, SQL Server | MongoDB

[🔝 Voltar para o início](#conteúdo)

### **Siga a conversão de nomes usada no Laravel**

 Siga [PSR standards](http://www.php-fig.org/psr/psr-2/).

 Siga também a conversão de nomes aceita pelo a comunidade Laravel:

O que | Como | Bom | Ruim
------------ | ------------- | ------------- | -------------
Controller | singular | ArticleController | ~~ArticlesController~~
Route | plural | articles/1 | ~~article/1~~
Named route | snake_case with dot notation | users.show_active | ~~users.show-active, show-active-users~~
Model | singular | User | ~~Users~~
hasOne or belongsTo relationship | singular | articleComment | ~~articleComments, article_comment~~
All other relationships | plural | articleComments | ~~articleComment, article_comments~~
Table | plural | article_comments | ~~article_comment, articleComments~~
Pivot table | singular model names in alphabetical order | article_user | ~~user_article, articles_users~~
Colunas em tabelas | snake_case without model name | meta_title | ~~MetaTitle; article_meta_title~~
Model property | snake_case | $model->created_at | ~~$model->createdAt~~
Foreign key | singular model name with _id suffix | article_id | ~~ArticleId, id_article, articles_id~~
Chaves primárias | - | id | ~~custom_id~~
Migrações | - | 2017_01_01_000000_create_articles_table | ~~2017_01_01_000000_articles~~
Métodos | camelCase | getAll | ~~get_all~~
Métodos em controllers | [table](https://laravel.com/docs/master/controllers#resource-controllers) | store | ~~saveArticle~~
Métodos e classes de teste | camelCase | testGuestCannotSeeArticle | ~~test_guest_cannot_see_article~~
Variáveis | camelCase | $articlesWithAuthor | ~~$articles_with_author~~
Collection | descriptive, plural | $activeUsers = User::active()->get() | ~~$active, $data~~
Object | descriptive, singular | $activeUser = User::active()->first() | ~~$users, $obj~~
Config e arquivos de linguagem | snake_case | articles_enabled | ~~ArticlesEnabled; articles-enabled~~
View | kebab-case | show-filtered.blade.php | ~~showFiltered.blade.php, show_filtered.blade.php~~
Config | snake_case | google_calendar.php | ~~googleCalendar.php, google-calendar.php~~
Contract (interface) | adjective or noun | Authenticatable | ~~AuthenticationInterface, IAuthentication~~
Trait | adjective | Notifiable | ~~NotificationTrait~~

[🔝 Voltar para o início](#conteúdo)

### **Tente sempre usar sintaxes pequenas e legíveis**

Ruim:

```php
$request->session()->get('cart');
$request->input('name');
```

Bom:

```php
session('cart');
$request->name;
```

Mais exemplos:

Sintaxe comum | Pequena e mais legíveis
------------ | -------------
`Session::get('cart')` | `session('cart')`
`$request->session()->get('cart')` | `session('cart')`
`Session::put('cart', $data)` | `session(['cart' => $data])`
`$request->input('name'), Request::get('name')` | `$request->name, request('name')`
`return Redirect::back()` | `return back()`
`is_null($object->relation) ? $object->relation->id : null }` | `optional($object->relation)->id`
`return view('index')->with('title', $title)->with('client', $client)` | `return view('index', compact('title', 'client'))`
`$request->has('value') ? $request->value : 'default';` | `$request->get('value', 'default')`
`Carbon::now(), Carbon::today()` | `now(), today()`
`App::make('Class')` | `app('Class')`
`->where('column', '=', 1)` | `->where('column', 1)`
`->orderBy('created_at', 'desc')` | `->latest()`
`->orderBy('age', 'desc')` | `->latest('age')`
`->orderBy('created_at', 'asc')` | `->oldest()`
`->select('id', 'name')->get()` | `->get(['id', 'name'])`
`->first()->name` | `->value('name')`

[🔝 Voltar para o início](#conteúdo)

### **Use contaneirs IoC (inversão de controle) ou facades no lugar de classes**

"new Class" sintaxe cria maior acoplamento classes e teste. Use IoC ou facades em vez disso.

Ruim:

```php
$user = new User;
$user->create($request->all());
```

Bom:

```php
public function __construct(User $user)
{
    $this->user = $user;
}

....

$this->user->create($request->all());
```

[🔝 Voltar para o início](#conteúdo)

### **Não recupere informaçẽos diretamente do `.env`**

Coloque os dados em arquivos de configuração e recupere através do helper `config()`.

Ruim:

```php
$apiKey = env('API_KEY');
```

Bom:

```php
// config/api.php
'key' => env('API_KEY'),

// Use data
$apiKey = config('api.key');
```

[🔝 Voltar para o início](#conteúdo)

### **Armaze datas em formatoes padrões. Use "accessors" and "mutators" para modificar o formato das datas**

Ruim:

```php
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->toDateString() }}
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->format('m-d') }}
```

Bom:

```php
// Model
protected $dates = ['ordered_at', 'created_at', 'updated_at']
public function getSomeDateAttribute($date)
{
    return $date->format('m-d');
}

// View
{{ $object->ordered_at->toDateString() }}
{{ $object->ordered_at->some_date }}
```

[🔝 Voltar para o início](#conteúdo)

### **Outras boas práticas**

Nunca coloque lógica em arquivos de rota.

Minimize o uso de vanilla PHP em templates Blade.

[🔝 Voltar para o início](#conteúdo)


Translations:

[Nederlands](https://github.com/Protoqol/Beste-Laravel-Praktijken) (by [Protoqol](https://github.com/Protoqol))

[한국어](https://github.com/xotrs/laravel-best-practices) (by [cherrypick](https://github.com/xotrs))

[日本語](japanese.md) (by [2bo](https://github.com/2bo))

[漢語](chinese.md) (by [xiaoyi](https://github.com/Shiloh520))

[ภาษาไทย](thai.md) (by [kongvut sangkla](https://github.com/kongvut))

[বাংলা](bangla.md) (by [Anowar Hossain](https://github.com/AnowarCST))

[فارسی](persian.md) (by [amirhossein baghaie](https://github.com/amirbagh75))

[Português](https://github.com/jonaselan/laravel-best-practices) (by [jonaselan](https://github.com/jonaselan))

[Українська](ukrainian.md) (by [Tenevyk](https://github.com/tenevyk))

[Русский](russian.md)

[Tiếng Việt](https://chungnguyen.xyz/posts/code-laravel-lam-sao-cho-chuan) (by [Chung Nguyễn](https://github.com/nguyentranchung))

[Español](spanish.md) (by [César Escudero](https://github.com/cedaesca))

[Français](french.md) (by [Mikayil S.](https://github.com/mikayilsrt))

[Polski](https://github.com/maciejjeziorski/laravel-best-practices-pl) (by [Maciej Jeziorski](https://github.com/maciejjeziorski))

[Türkçe](turkish.md) (by [Burak](https://github.com/ikidnapmyself))

[Deutsch](german.md) (by [Sujal Patel](https://github.com/sujalpatel2209))

[Italiana](italian.md) (by [Sujal Patel](https://github.com/sujalpatel2209))

[Azərbaycanca](https://github.com/Maharramoff/laravel-best-practices-az) (by [Maharramoff](https://github.com/Maharramoff))

It's not a Laravel adaptation of SOLID principles, patterns etc. Here you'll find the best practices which are usually ignored in real life Laravel projects.

- Renato Lucena
> @cpdrenato
