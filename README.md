# Email verification in a Laravel 5.6 app

This app is going to send emails to verify that an email address is real when users sign up. Before their account is active users need
to click a special link that will verify their account and that they're in control of that email address. This is a common 
web functionality we've all experienced when signing up for new accounts online. The impetus for this project is that I deployed 
a Laravel app for people to use and put it behind a login form. Many people used garbage emails to sign up and access the app. 
This isn't protected by default with Laravel's auth scaffold, so let's add it real quick!

> Draws heavily from [this tutorial](https://hackernoon.com/how-to-use-queue-in-laravel-5-4-for-email-verification-3617527a7dbf) by Ahmed Khan ([@ahmedkhan847](https://github.com/ahmedkhan847))

> [Source code](https://github.com/connor11528/laravel-email-verification) is available for free on Github.

## Step 1

First step is to create a new Laravel app and generate the generic Laravel auth scaffold:

```
$ php artisan make:auth
```

## Step 2

Head into the Laravel database migrations files and find the **create_users_table** migration. (Hint: it's in **database/migrations**.) 
From there, add two columns to the users table for a token and an is verified field. The verified will default to zero for false.

```
public function up()
{
	Schema::create('users', function (Blueprint $table) {
		$table->increments('id');
		$table->string('name');
		$table->string('email')->unique();
		$table->string('password');
		$table->tinyInteger('verified')->default(0);
		$table->string('email_token')->nullable();
		$table->rememberToken();
		$table->timestamps();
	});
} 
```

## Step 3

Add a queue table for queued jobs and failed jobs, then migrate the database to set it all up.

``` 
$ php artisan queue:table
$ php artisan queue:failed-table
$ php artisan migrate
```

> If you get an error saying access denied, make sure you create a MySQL database for your Laravel app to connect to. You can do that by 
running `mysql -uroot -p` and then the sql command `create database laravelemailverification;`. Download something like [Sequel Pro](https://www.sequelpro.com/) 
to view your tables and database content.

## Step 4

For the migration to work you'll already have to have modified your **.env** file to connect to your database. Now we're going to 
modify it to connect to a queue driver and mail driver. This is going to use the database as a queue driver and gmail for verification
emails.

``` 
APP_URL=http://localhost:8000

QUEUE_DRIVER=database

MAIL_DRIVER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=connorleech@gmail.com
MAIL_PASSWORD=
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=connorleech@gmail.com
MAIL_FROM_NAME="Connor"
```

> For the new values to take effect you may need to run `php artisan config:clear`

> If you are sending emails from Gmail you'll have to turn off additional protections. Head to https://www.google.com/settings/security/lesssecureapps and turn off
this setting. Without this, Google will block your Laravel app's sign in attempts.

## Step 5

Create the email that you want to send to the user. There is a handy artisan command for scaffolding emails:

``` 
$ php artisan make:mail EmailVerification
```

That file is in **app/Mail** and can be modified to look like so: 

``` 
<?php

namespace App\Mail;

use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Contracts\Queue\ShouldQueue;

class EmailVerification extends Mailable
{
    use Queueable, SerializesModels;
    
    protected $user;

    /**
     * Create a new message instance.
     *
     * @return void
     */
    public function __construct($user)
    {
        $this->user = $user;
    }

    /**
     * Build the message.
     *
     * @return $this
     */
    public function build()
    {
        return $this->view('email.verify_account')->with([
            'email_token' => $this->user->email_token    
        ]);
    }
}
```

## Step 6

Create the email view within **resources/views/email/verify_account.blade.php**:

``` 
<h3>Click the Link To Verify Your Email</h3>
Click the following link to verify your email {{ url('/verifyemail/' . $email_token) }}
```

## Step 7

Now we need to make a new job that will fire off the email:

``` 
$ php artisan make:job SendVerificationEmail
```

That job will live in **app/Jobs**. We pass in the user and send a verification email within this job. In order to have your queue active 
and listening for jobs you need a separate tab from your `php artisan serve` running `php artisan queue:work`. If you are making frontend changes 
you're going to want to have `npm run watch` running in a third tab, but frontend Javascript is out of the scope of this tutorial.

``` 
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Mail;
use App\Mail\EmailVerification;

class SendVerificationEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $user;

    /**
     * Create a new job instance.
     *
     * @return void
     */
    public function __construct($user)
    {
        $this->user = $user;
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle()
    {
        $email = new EmailVerification($this->user);
        Mail::to($this->user->email)->send($email);
    }
}
```

## Step 8

Update the RegisterController.php:

```
<?php

namespace App\Http\Controllers\Auth;

use App\User;
use App\Http\Controllers\Controller;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Validator;
use Illuminate\Foundation\Auth\RegistersUsers;
use Illuminate\Auth\Events\Registered;
use Illuminate\Http\Request;
use App\Jobs\SendVerificationEmail;

class RegisterController extends Controller
{
    /*
    |--------------------------------------------------------------------------
    | Register Controller
    |--------------------------------------------------------------------------
    |
    | This controller handles the registration of new users as well as their
    | validation and creation. By default this controller uses a trait to
    | provide this functionality without requiring any additional code.
    |
    */

    use RegistersUsers;

    /**
     * Where to redirect users after registration.
     *
     * @var string
     */
    protected $redirectTo = '/home';

    /**
     * Create a new controller instance.
     *
     * @return void
     */
    public function __construct()
    {
        $this->middleware('guest');
    }

    /**
     * Get a validator for an incoming registration request.
     *
     * @param  array  $data
     * @return \Illuminate\Contracts\Validation\Validator
     */
    protected function validator(array $data)
    {
        return Validator::make($data, [
            'name' => 'required|string|max:255',
            'email' => 'required|string|email|max:255|unique:users',
            'password' => 'required|string|min:6|confirmed',
        ]);
    }

    /**
     * Create a new user instance after a valid registration.
     *
     * @param  array  $data
     * @return \App\User
     */
    protected function create(array $data)
    {
        return User::create([
            'name' => $data['name'],
            'email' => $data['email'],
            'password' => Hash::make($data['password']),
            'email_token' => base64_encode($data['email']),
        ]);
    }

    /**
     * Handle a registration request for the application.
     *
     * @param \Illuminate\Http\Request $request
     * @return \Illuminate\Http\Response
     */
    public function register(Request $request)
    {
        $this->validator($request->all())->validate();
        event(new Registered($user = $this->create($request->all())));

        dispatch(new SendVerificationEmail($user));

        return view('verification');
    }

    /**
     * Handle a registration request for the application.
     *
     * @param $token
     * @return \Illuminate\Http\Response
     */
    public function verify($token)
    {
        $user = User::where('email_token', $token)->first();
        $user->verified = 1;

        if($user->save()){
            return view('emailconfirm', ['user' => $user]);
        }
    }
}
```

## Step 9

and add the views...

**resources/views/email/verify_account.blade.php**

```
<h1>Click the Link To Verify Your Email</h1>
Click the following link to verify your email {{ url('/verifyemail/' . $email_token) }}
```

**resources/views/verification.blade.php**

```
@extends('layouts.app')
@section('content')
    <div class="container">
        <div class="row">
            <div class="col-md-8 col-md-offset-2">
            <div class="panel panel-default">
                <div class="panel-heading">Registration</div>
                <div class="panel-body">
                    You have successfully registered. An email is sent to you for verification.
                </div>
            </div>
        </div>
    </div>
@endsection
```

**resources/views/emailconfirm.blade.php**

```
@extends('layouts.app')
@section('content')
    <div class="container">
        <div class="row">
            <div class="col-md-8 col-md-offset-2">
            <div class="panel panel-default">
                <div class="panel-heading">Registration Confirmed</div>
                <div class="panel-body">
                    Your Email is successfully verified. Click here to <a href="{{ url('/login') }}">login</a>
                </div>
            </div>
        </div>
    </div>
    </div>
@endsection
```

## Fin!

That's it! Register for your application locally and you'll be redirected to the "we sent you an email screen". Check your email, click the link 
and your account will be verified. From there you'll be able to login successfully. All of our testing here is done locally. To get your application
live to the internet you could [deploy to heroku](http://connorleech.info/blog/Deploy-a-Laravel-5-app-to-Heroku/). As an exercise for the reader you 
could switch your queue driver to Redis with Laravel Horizon, or leave it as is and start building the rest of your app.

If you have any questions shoot me an email (above) or hit me up on [twitter](https://twitter.com/Connor11528).

> [Source code](https://github.com/connor11528/laravel-email-verification) is available for free on Github.

---

**Update 04/06/18**: For sending a confirmation email after the user changes their password check out [@smayzes](https://github.com/smayzes) tutorial [here](https://medium.com/@smayzes/send-email-after-changing-password-in-laravel-55f1f656fdc9)
