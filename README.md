# Laravel notes
This repo contains notes, warnings, tricks and tips from my development experience with Laravel.

## Change Timezone
config folder - app.php file
in config array on "timezone" key change the value from UTC to date time you want, search in [PHP DOCS](https://www.php.net/manual/es/timezones.php)

## Work with SqlServer:
- "SA" is the root user, password is from installation or you could create a user with SQL Server Manager 
- Enable TCP protocol from SQL Server Config
- If use Linux could install from [Microsoft guide] ()
- if use Windows you will need ODBC installer and SQL Server Manager
- dates not work as well as in Mysql or Sql lite so you need to change dates from migrations in Models files, you have two options:

Cast the property $dateFormat that join created_at and updated_at fields with the next code: 
  
```php
    protected $dateFormat = 'Ymd H:i:s';
```
  
 Or overwrite the original method that writes the date in Eloquent with a trait, something like this:
 
```php
  namespace App;
  trait SqlServerGetDateFormat
  {
      public function getDateFormat()
      {
          return 'Y-m-d H:i:s';
      }

  }
```

 This allow to add the trait to all models from app folder with and elegant way and no need to cast all models dateFormat fileds, like.
 
 ```php
  namespace App;
  use Illuminate\Notifications\Notifiable;
  use Illuminate\Contracts\Auth\MustVerifyEmail;
  use Illuminate\Foundation\Auth\User as Authenticatable;
  class User extends Authenticatable
  {
      use Notifiable, SqlServerGetDateFormat;
      ...
  }
 ```
 
 Other way to fix this is to cut the length of the default date format of Laravel migrations to:
 
  ```php
    $table->timestamps(4);
  ```
  ## Migrations
  
 From Laravel 5.6 the auth could have a user migration with type "bigIncrement" instead of "increment".
 both could be use but if a migration have bigIncrement in some field to be referenced as foreign key you will need
 to be explicit with it as "unsignedInteger" if its "increment" or "unsignedBigInteger" if its "bigIncrement".
 you could read more about it [here a Larvel Daily post](https://laraveldaily.com/be-careful-laravel-5-8-added-bigincrements-as-defaults/)
 
 File Of strong type model.
 
 ```php
  Schema::create('transactions_types', function (Blueprint $table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->timestamps();
  });
  ```
  File of week type
  
 ```php
  Schema::create('transactions', function (Blueprint $table) {
      $table->bigIncrements('id');
      $table->bigInteger('transaction_type_id')->unsigned();//--------------------------Here is the change from integer to bigInteger.
      $table->foreign('transaction_type_id')->references('id')->on('transactions_types');
      $table->decimal('amount');
      $table->timestamps();
  });
 ```
  - You can change the type of some migration property in model by cast a property 
  in model file but instead of this you can add directly in your migrations the type boolean and no need to cast an integer to bool
  
```php
  Schema::create('products', function (Blueprint $table) {
      $table->bigIncrements('id');
      $table->string('name');
      $table->boolen('status');//instead of integer
      $table->timestamps();
  });
 ```
  - You can set default values for some fields directly in migration and saves you 
  add it to all factories or even as a default value when the app is on production.
  
```php
  Schema::create('products', function (Blueprint $table) {
      $table->bigIncrements('id');
      $table->string('name');
      $table->boolen('status')->default(0);//instead of integer
      $table->timestamps();
  });
 ```
  
  ## Model changes
  
  - you could cast values of certain fields
  - you could use the convention of model as singular and table as plural but you could change this as: 
  
 ```php
  protected $table = '*TableNamePlaceholder*';
 ```
  - you could maintain the convention of table id as 'id', but if you change it the model binding and the find() method
  will fail so to use it as usually you need to change the primary key value explicit as:
  
 ```php
  protected $primaryKey = '*idPlaceholder*';
 ```
  - either you could change the default name of timestamps values 'created_at' and 'updated_at' like this:
  
 ```php
  const CREATED_AT = '*createdAtTitleChangedPlaceholder*';
  const UPDATED_AT = '*updatedAtTitleChangedPlaceholder*';
 ```
  - Remember to fill data in table you need to be explicit to avoid the massive assignment prevention of Laravel as:
  
 ```php
  protected $fillable = [ 'row1', 'row2' .... ];
 ```
 
  - a pretty interesting setting in model is the ability to set some property as softdelete to protect it and the delete method work only as a softdelete, you could use softDelete() method but in some cases you will set this in the model to prevent to delete a specific field and to work as some interface to other developers or the same in the future to protect from him or herself.
  
 ```php
  use Illuminate\Database\Eloquent\SoftDeletes;
  class post extends Model 
  {
    use SoftDeletes;//this set a softDelete only to this field on DB or property of model. (here delete() works same as softDelete() ).
    protected $date = 'nameOfTheDateRowOrWhateverRowTypeOrNameIsJustAnExample';
  }
 ```
 
  - To make get data of an object binded by foreign key you can create custom functions in models to get data related 
    with the id of the foreign key a classic example is the relation that have a post from a blog with an author, 
    in this example a post just can have one author so the relation is one to many, 
    one author to many posts that can be written by the same author, so a post belongs to one author:
    
  ```php
  public function author()
  {
    //if user model follows the convention of id with name 'id'
    return $this->belongsTo(App\user);
    //else you can add a second param with the custom id field name
    return $this->belongsTo(App\user, 'id_author');
  }
 ```
 
 ## Auth validation
 
  - To make a manual compare from Password enter by a user and password from DB you could use the method Hash::check()
  it requeries two params, first the password write and send by user and the password of a model 
  but this cant be fetch directly as "model->password" so you could use "model->getAuthPassword()" to get the model certainly you
  will need something as id or email or user to find the model to compare with find() method 
  so a very complete example will be something like this.
  
 ```php
  public function index(Request $request)
  {
    $user = $this->repo->getUserCredentials($request->email);//Repo get user from DB by email
    //send the model from repo and the request with data provided by user.
    $this->validates(['user' => $user, 'request' => $request]);
  {
  
  public function validates($data)//data is the array
  {
       if ($data['user']) {//if user exists because findOrfail redirect and find returns null or false
            if (Hash::check($data['request']->password, $data['user']->getAuthPassword())) {
                return 'OK You are logged in';
            }
        }
        return 'Auth Error!';
  {
```
 
 ## Faker tricks
 
  - you could need to fill data in a factory with some data related from user instead of use an array with users ids, and a random value from those with native function rand(minId, MaxId), you could do this:
  
```php
  $factory()->define(App\SomeModel::class, function(Faker $faker) {
    return [
      'author' => App\User::all()->random(),
      ...
    ];
  });
```

 - If want to simulate a password
 
 ```php
  $faker->bcrypt('passwordStringToTest');
```

 - If want to simulate a names or phrases with some number of words, faker support with some methods like name, paragraph, text, address ect two arguments the number of words to display and false as second param:
 
 ```php
  $faker->paragraph(3, false);
```

## Request

 - If you want to get all data submitted from any request you could get it as an object with:
```php
  $Request();
```
 - If you want to get data from a request but not a form request, (REST API) for example
 ```php
  $Request()->user;//or whatever property that be part of the request object
```
 - you could get inputs data from a request as :
 
```php
  $Request()->input('inputNameFromNameAttributeInHTML');
```

## Validations

 - unique() to not repeat some data in DB.
 - unique() have an exception you could pass as argument the id of current model to access update method and update some data but if the unique field is the same it will accept an exception from the same model id.
 
 - you can use validate() method from request to validate data in a very elegant way:
 
```php
  $Request()->validate([ 'name' => 'required' ]);
  //if we set more than one rule the values of the array key that represent the name of 
  //the request property must be an array or string with separated by comas
  $Request()->validate([ 'email' => 'required, email, min:5' ]);
  //or params as array more elegant
  $Request()->validate([ 'email' => ['required', 'email', 'min:5'] ]);
  //some example with unique in this case (unique:"tableNamePlaceholder*") refers to db column name to search that be unique
  //and concat exception id if the record updated is the same
  $Request()->validate([ 'email' => ['required', 'email', 'min:5', 'unique:users'.$user->id] ]);
  //From Laravel 5.8 we could do this to pass the column name from db as string and chain a ignore method with user id
   Rule::unique('users')->ignore($user->id),
  //this method Rule support model binding so instead of the int key you could pass entire model
  Rule::unique('users')->ignore($user)
  //if you not follow the id convention of call any id as 'id' in migrations you will need to add the id column name as the second param
  Rule::unique('users')->ignore($user->id, 'user_id')
  // you could set more than one field unique as this
  Rule::unique('users', 'email_address')->ignore($user->id),
```
 
 [Reference Laravel Doc](https://laravel.com/docs/5.8/validation#rule-unique)
 
## Artisan commands

 - To make a migration and seed joined you could write a migration with a flag:
 
```php
  php artisan migrate:fresh/refresh --seed// to make a migration and fill data at once
```

 - If you are working with a CRUD you could use some flags to avoid creating all related files from a certain model one by one.
 you could create a model with this flags:
 
```php
  php artisan make model ModelNameInSingularToMaintainConvention -a  
```

 It makes a model with a model with convention to be related with current controller, 
 a controller with boilerplate of a source controller and related with a model, a migration and a factory related.
 all at once and with all boilerplate correctly related and reference it.
 
 ## Routes
 
  - To generate all default routes from a CRUD you could use resource type:
   
```php
  Route::resource('post', 'PostController');
  //to generate post urls in PostController file with the convention routes pointing to convention controller methods  
```

## Blade
 
 - To display data with too much length laravel provide a helper str_limit() and requires two params first the string to show
 and second the number of letters to count and after that the helper display three dots like ... good for long texts.
 
## Composer issues
  - to update a Laravel version just write:
```bash
  composer update
```
 - if any project will need to be deploy write:
```bash
  composer install
```
 - if composer returns an error like Cannot update packages anymore "Fatal error: Out of memory (allocated 1392771072) (tried to allocate 268435456 bytes)" write:
 
```bash
  sudo /bin/dd if=/dev/zero of=/var/swap.1 bs=1M count=1024
  sudo /sbin/mkswap /var/swap.1
  sudo /sbin/swapon /var/swap.1
```

## Dates with Carbon

 - to change format of a date you will need a carbon instance and you can reorder a date to any other format:
 ```php
  $date = Carbon::now();
  $date = $date->format('d-m-Y');//print date like 07-05-2015
  $date->format('l jS \\of F Y h:i:s A'); // print date like Thursday 7th of May 2015 01:17:56 PM
```
 - If you recibe another date format to became it a Carbon instance just:
 
```php
  $otherDate = '2019-05-28';
  $date = Carbon::createfromformat('Y-m-d', $otherDate);
```
 
  - Create format of date customized to became 2019-05-28 date as string to 28/04/2019 as a latin date string or other countries just add the format helper
  
```php
   Carbon::createFromFormat('Y-m-d', $date)->format('d/m/Y')
```
  
  - To create a custome date string you can use isoFormat helper
 
```php
   Carbon::createFromFormat('Y-m-d', $date)->isoFormat('Do MMMM YYYY')
```
  

## Manage images

 - Upload an image to store folder
 from a request you could call the method store from request object (https://laravel.com/docs/5.8/requests#storing-uploaded-files)
 ```php
 //store image in storage/app/images
  $path = $request->photo->store('images');
  //if you need to store it in subfolder you can pass it as second param to store it in storage/app/public/uploads
  $path = $request->photo->store('uploads', 'public');
```
 
 - validate an image
  you can validate as any other request object as:
```php
if(request()->has('nameOfInputInForm')){
  //execute anything
} 
```
 - other type of validation is to use the validate method from request object

```php
if(request()->validate([
  'imageInputNameFromForm' => 'required|mimes:jpeg,jpg,png|max:5000',//max will validate length in string but in images validate weight of the file
])){
  //execute anything
} 
```

 - Store an image into DB
 You can add it to a model as any other string in a property using the store method it return a string with the path to just assign it    to a model property like:

```php
 $user->image = request()->image->store('uploads', 'public');
```
 
 - create a symbolic link
   laravel have an artisan command since 5.3 version to create a symbolic link to show files stored in store folder into a public folder    just type the command:
   
```bash
 php artisan storage:link
```
 - Show the image from store folder
 
 ```php
 {{ asset('storage/'.$user->image) }} // asset helper write the app address and concatenate it with the $user->image in this example
 //it will be something like youapp.com/storage/uploads/user-avatar.png
```

 - Delete file stored in store folder the path to file is relative from storage disk so if path is storage/public/uploads/avatar.png:
```php
 Storage::delete('public/'.$user->image);//$user->image is = to 'uploads/avatar.png'
```
## Packages

### Searchable (https://github.com/nicolaslopezj/searchable)

The readme is very clear so just follow it as the entire search text bullet indicates, but in some configs the package throws an error as: 
  - Laravel : Syntax error or access violation: 1055 Error
  In this case the solution is simple just go to config folder and database.php file and in mysql array change 
  the key value from true   to false.
  
```php
  'strict' => false,
```

## Send emails
     - You can implement email name of the app and email sender when someone recibed an email just fill the file in config/mail.php
     and change the example value and example email with real data:
```php
    'from' => [
      'address' => env('MAIL_FROM_ADDRESS', 'yourapp@mail.com'),
      'name' => env('MAIL_FROM_NAME', 'Your app'),
    ],
```

  - Then for development you can work with gmail and in production you can also if you follow some steps:
  change env file of mail config

```env
MAIL_DRIVER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=465
MAIL_USERNAME=yourmailwithoutcommas@gmail.com
MAIL_PASSWORD=yourpasswordwithoutcommas
MAIL_ENCRYPTION=ssl
```

  - Then you need to access to your account and config security section and allow not safe apps 
  (no matter because its you)(link)[https://myaccount.google.com/lesssecureapps?utm_source=google-account&utm_medium=web]
  
  - In some rarely cases you need to disable recaptcha you can do it 
  with this (link)[https://accounts.google.com/b/0/DisplayUnlockCaptcha]
  
  - If you work with AWS or Heroku depends on where it host you need to aprove access of your account from other device
  ------------------------------------------------------------------------------------------------------------------------------
  THIS STEP TAKE A FEW MINUTES TO SEND AN EMAIL TO YOUR ACCOUNT TO GIVE ACCESS OR VERIFY IF ITS OK TO GIVE ACCESS TO 
  YOUR ACCOUNT WITH OTHER DEVICE.
  
  YOU CAN USE THIS LINK TO UNLOCK CAPTCHA AGAIN
  (LINK)[https://accounts.google.com/b/0/DisplayUnlockCaptcha]
  OR GO TO YOUR EMAIL AND WAAAIT SOME TIME LIKE 5 MINUTES OR 10 AND ACCEPT THE ACCESS OF YOUR ACCOUNT FROM OTHER DEVICE
  ------------------------------------------------------------------------------------------------------------------------------
  
  And thats all you have your gmail account ready to send emails with Laravel in development and in production enviroments.
  for massive email feature its not recommended you need some tool like mailgun, or other you can consult 
  (this guide)[https://scotch.io/tutorials/ultimate-guide-on-sending-email-in-laravel]
  
  - Customize the reset notification: 
  
    ```
    ## IF YOU NEED JUST TRANSLATE THE EMAIL YOU CAN USE (LARAVEL CAOUECS PACKAGE)[https://github.com/caouecs/Laravel-lang], 
    JUST FOLLOW README IT HAVE VERY WELL AND EASY DOCS
    OR ANY OTHER THERE ARE MANY TO HANDLE THIS.
    ```
    
  
  1. Create a new notification: name not matter but to follow the Laravel Standard
  
```env
  php artisan make:notification ResetPasswordNotification
```
  2. Now in app folder we have new notifications folder and then inside we have the file ResetPasswordNotification.php
  
  3. Now we are going to copy all content of the class 
  ResetPassword in vendor/laravel/src/Illuminate/Auth/notifications/ResetPassword.php and we are going to paste it 
  in our custome app/notifications/ResetPasswordNotification.php file in the body of the class
  ```php 
      /**
     * The password reset token.
     *
     * @var string
     */
    public $token;

    /**
     * The callback that should be used to build the mail message.
     *
     * @var \Closure|null
     */
    public static $toMailCallback;

    /**
     * Create a notification instance.
     *
     * @param  string  $token
     * @return void
     */
    public function __construct($token)
    {
        $this->token = $token;
    }

    /**
     * Get the notification's channels.
     *
     * @param  mixed  $notifiable
     * @return array|string
     */
    public function via($notifiable)
    {
        return ['mail'];
    }

    /**
     * Build the mail representation of the notification.
     *
     * @param  mixed  $notifiable
     * @return \Illuminate\Notifications\Messages\MailMessage
     */
    public function toMail($notifiable)
    {
        if (static::$toMailCallback) {
            return call_user_func(static::$toMailCallback, $notifiable, $this->token);
        }

        return (new MailMessage)
            ->subject(Lang::getFromJson('Reset Password Notification'))
            ->line(Lang::getFromJson('You are receiving this email because we received a password reset request for your account.'))
            ->action(Lang::getFromJson('Reset Password'), url(config('app.url').route('password.reset', 
            ['token' => $this->token, 'email' => $notifiable->getEmailForPasswordReset()], false)))
            ->line(Lang::getFromJson('This password reset link will expire in :count minutes.', 
            ['count' => config('auth.passwords.users.expire')]))
            ->line(Lang::getFromJson('If you did not request a password reset, no further action is required.'));
    }

    /**
     * Set a callback that should be used when building the notification mail message.
     *
     * @param  \Closure  $callback
     * @return void
     */
    public static function toMailUsing($callback)
    {
        static::$toMailCallback = $callback;
    }
```
    
    4. Now we need to copy the trait sendPasswordResetNotification from the file vendor/laravel/src/Auth/passwords/CanResetPassword.php
    and paste in our User model, in app/User.php
   
```php
    /**
     * Send the password reset notification.
     *
     * @param  string  $token
     * @return void
     */
    public function sendPasswordResetNotification($token)
    {
        $this->notify(new ResetPasswordNotification($token));
        //ResetPasswordNotification this name is the same because we create it with same name to follow the Laravel convention
    }
```

  5. Now we need to import our custome notification in User model:
    
```php
// User model
use App\Notifications\ResetPasswordNotification;
class User extends Authenticatable
{
....
}
```
  6. Now we just need to customize the data to send in app/notifications/ResetPasswordNotification in 
  sendPasswordResetNotifications trait, with our custome data as for example:
  
```php
        return (new MailMessage)
            ->subject(Lang::getFromJson('Reset Password Notification'))
            ->line(Lang::getFromJson('Well well you forgot your password again.. :)'))
            ->action(Lang::getFromJson('Reset Password ;)'), url(config('app.url').route('password.reset', ['token' => $this->token, 
            'email' => $notifiable->getEmailForPasswordReset()], false)))
            ->line(Lang::getFromJson('You only have :count minutes to use this link so quickly!.', ['count' => 
            config('auth.passwords.users.expire')]))
          
```

  * since Laravel 5.7 many files autogenerated have the Lang::getFromJson helper or the blade syntax for the same {{ __('') }} , 
  if is not your case its recommendable to add it.
    
  
  ## Deploy

  # Heroku
  To deploy to heroku you will follow this guide, this guide is based in different Heroku documentations:
  
```bash
  composer create-project laravel/laravel --prefer-dist the_name_of_your_app
  cd the_name_of_your_app
  git init
  git add .
  git commit -m "new laravel project"
  echo "web: vendor/bin/heroku-php-apache2 public/" > Procfile
  git add .
  git commit -m "Procfile for Heroku"
  heroku create the_name_of_your_app
  heroku config:set APP_KEY= the_app_key_of_your_project_env_file
  git push heroku master
  heroku open
```

  - You need to all values of env file in Heroku by command like "heroku config:set APP_KEY=132..." or use the Heroku dashboard
  you need to add the next env values:
    - APP_NAME=Laravel
    - APP_ENV=local, or production depends on purpose of server
    - APP_KEY=your_app_key_generated_by_laravel_by_default_or_with_composer_key_generate_command
    - APP_DEBUG=true
    - APP_URL=replace_with_app_vars_provided_by_heroku_in_dashboard
    - LOG_CHANNEL=stack
    - DB_CONNECTION=pgsql
    - DB_HOST=replace_with_app_vars_provided_by_heroku_in_dashboard
    - DB_PORT=5432
    - DB_DATABASE=the_name_of_your_app
    - DB_USERNAME=replace_with_app_vars_provided_by_heroku_in_dashboard
    - DB_PASSWORD=replace_with_app_vars_provided_by_heroku_in_dashboard

  - and change log file to get generated logs and view files, to view logs in Heroku you need 
  to change the log in your laravel project in config/logging.php
 
 ```php
 <?php
return [
    'default' => env('LOG_CHANNEL', 'stack'),
    'channels' => [
        'stack' => [
            'driver' => 'stack',
            'channels' => ['single'],
        ],
        'single' => [
            'driver' => 'errorlog',
            'level' => 'debug',
        ],
…
```
 
   - Then you need to add and push your changes to Heroku app

```bash
  git add config/logging.php
  git commit -m "use errorlog"
  git push heroku master
```
  - this workflow is same as git and with every change to be reflect in Heroku you need to type those commands
  
  - To add load balancers in Heroku or Amazon Web Service the guide will be added in some time but refference is in this (link)
  [https://devcenter.heroku.com/articles/getting-started-with-laravel#trusting-the-load-balancer]
  
  
  ## Handling API
  
  - to return any error request as a json you can add You can just set header key Accept :application/json and 
  it will return json response.
  
  - other option is :
  
  - You can use api-key in a header to handle request between server and client it is a string and can be generated by 
  the helper/facades Hash::make('string-to-convert')
  
  - Its important to handle API routes in some order, you could but technically is not neccesary to add API routes in routes/api
  but its a good practice
  
  - You can handle routes by resource routes but instead you can handle API routes as 
  
  
  - you can generate a route for api with the artisan command: 
```php
  php artisan make:controller PhotoController --resource
```  
  - also you can use other command to create the correspond controller for specific API RESOURCE WITH
```php  
  php artisan make:controller API/PhotoController --api
```
  - You can add a prefix to a group of routes to handle in a very commond and effective way to handle API versioning
  or even add middleware to a group of routes
  
```php
Route::group(['prefix' => 'v1', 'middleware' => 'VerifyAccessKey' ], function() {
    
    Route::post('login', 'AuthApiController@index');
    Route::get('properties', 'PropertyController@showEvents');
    Route::get('events/{id}', 'EventsController@show');
    
});
```

  - You can use a special resource for API for single route with:
```php 
  Route::apiResource('photos', 'PhotoController');
```
  - Or use for many routes like: 
```php
  Route::apiResources([
    'photos' => 'PhotoController',
    'posts' => 'PostController'
]);
```
  - if you need to add params to the specific routes generated by your Route::apiResource you can see first how its build by
  running
```php  
  php artisan route:list
```
  - if you need to add changes to add params to specific route in a group of api resources you can add the next helper
```php
  Route::resource('users', 'AdminUserController')->parameters([
    'users' => 'admin_user'
]);
```

  * parameters helper generate a change to accept the route: /users/{admin_user}
  
  - to return errors from validate helper in json format you also can use this guide (Click Here)[https://jamesmills.co.uk/2019/06/05/how-to-return-json-from-laravel-form-request-validation-errors/]
