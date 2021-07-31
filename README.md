Laravel multiple auth =========================

user auth ------------------------------

1. install laravel 8 using below command
-> composer create-project laravel/laravel example-app

2. open project without artisan
-> rename server.php to index.php in root folder then move .htaccess file from public folder to root folder

3. run project using localhost/<projectName>

4. create user authentication using below commands
-> composer require laravel/ui
-> php artisan ui vue --auth (you can use bootstrap/vue/React etc insted of vue)
-> npm install
-> npm run dev

5. configure database in .env and run below command
-> php artisan migrate



admin auth ------------------------------

1) create migration for admin table
-> php artisan make:migration create_admins_table --create=admins and put below code in newly created migration
    $table->increments('id');
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();

2. run migration using "php artisan migrate"

3. create a folder "Admin" in app/Models/ then copy User.php model in Admin folder then do below changes
-> rename model User.php to Admin.php and change class User to Admin
->  change name space namespace App\Models\Admin;
Path : app\Models\Admin\Admin.php

4. open config/auth and set guards and providers like below
Example :
// guards
'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],

        'api' => [
            'driver' => 'token',
            'provider' => 'users',
            'hash' => false,
        ],

        'admin' => [
            'driver' => 'session',
            'provider' => 'admins',
        ],
    ],

//Providers

'providers' => [
        'users' => [
            'driver' => 'eloquent',
            'model' => App\Models\User::class,
        ],

        'admins' => [
            'driver' => 'eloquent',
            'model' => App\Models\Admin\Admin::class,
        ],
    ],



5. put "protected $guard = "admin";" in Admin model(app\Models\Admin\Admin.php)

dashboard creating after login -------------------------

6. in app\Http\Controllers\ do the below steps
->create a folder named "Admin" in app\Http\Controllers\ ,
->then copy app\Http\Controllers\HomeController.php then paste in app\Http\Controllers\Admin\ and rename it to AdminController.php and change its class name as the controller name change code in index method like ( return view('admin.home');)
->change constructor in AdminController.php
    public function __construct()
    {
        $this->middleware('auth:admin');
    }

7. open routes\web.php and add this route "Route::get('/Admin-home', [App\Http\Controllers\Admin\AdminController::class, 'index'])->name('admin.home');"

8. create a folder "admin" in resources/views then do below procedure
-> copy layouts folder and home.blade.php and pest it into resources/views/admin/
-> open home.blade.php and change @extends('layouts.app') to @extends('admin.layouts.app')

creating auth controllers --------------------------------------------------

9. copy auth folder from app\Http\Controllers\ and paste it in app\Http\Controllers\Admin and do below steps

1) change name space of all controllers of Admin\Auth\ like "namespace App\Http\Controllers\Admin\Auth";

creating auth routes ----------------------------------------------------------

use App\Http\Controllers\Admin\AdminController;
use App\Http\Controllers\HomeController;
use App\Http\Controllers\Admin\Auth\LoginController;
use App\Http\Controllers\Admin\Auth\ForgotPasswordController;
use App\Http\Controllers\Admin\Auth\ResetPasswordController;
use App\Http\Controllers\Admin\Auth\RegisterController;

Route::prefix('admin')->group(function () {
// Authentication Routes...
    Route::get('/login', [LoginController::class, 'showLoginForm'])->name('admin.login');
    Route::post('/login', [LoginController::class, 'Login']);
    Route::post('/logout', [LoginController::class, 'logout'])->name('admin.logout');

    // // Registration Routes...
    // Route::get('/register', '[RegisterController::class], 'showRegistrationForm')->name('register');
    // Route::post('/register', '[RegisterController::class], 'register');

    // // Password Reset Routes...
    Route::post('/password/email', [ForgotPasswordController::class, 'sendResetLinkEmail'])->name('admin.password.email');
    Route::get('/password/reset', [ForgotPasswordController::class, 'showLinkRequestForm'])->name('admin.password.request');
    Route::post('/password/reset', [ResetPasswordController::class, 'reset']);
    Route::get('/password/reset/{token}', [ResetPasswordController::class, 'showResetForm'])->name('admin.password.reset');
    Route::get('/', [AdminController::class, 'index'])->name('admin.home');
});


admin login -----------------------------------------------------------------

1) open app\Http\Controllers\Admin\Auth\LoginController.php and put below code

    public function showLoginForm()
    {
        return view('admin.auth.login');
    }

2) copy auth folder from resources\views\ and paste it in resources\views\admin

3) change form action in resources\views\admin\auth\login.blade.php like "{{ route('admin.login') }}"

4) and put below code in app\Http\Controllers\Admin\Auth\LoginController.php with name space "use Illuminate\Http\Request;"

 /**
     * login
     *
     * @param  mixed $request
     * @return void
     */
    public function login(Request $request)
    {

        // Validate the form data
        $this->validate($request, [
        'email'   => 'required|email',
        'password' => 'required|min:6'
      ]);

        // Attempt to log the user in
        if (Auth::guard('admin')->attempt(['email' => $request->email, 'password' => $request->password], $request->remember)) {

            // if successful, then redirect to their intended location
            return redirect()->intended(route('admin.home'));
        }
        // if unsuccessful, then redirect back to the login with the form data
        return redirect()->back()->withInput($request->only('email', 'remember'));
    }

5) put below code with use Illuminate\Support\Arr; in app\Exceptions\Handler.php for redirect to right login page if not authenticated user

/**
     * Convert an authentication exception into an unauthenticated response.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Illuminate\Auth\AuthenticationException  $exception
     * @return \Illuminate\Http\Response
     */
    protected function unauthenticated($request, AuthenticationException $exception)
    {
        if ($request->expectsJson()) {
            return response()->json(['error' => 'Unauthenticated.'], 401);
        }

        $guard = Arr::get($exception->guards(), 0);

        switch ($guard) {
          case 'admin':
            $login = 'admin.login';
            break;

          default:
            $login = 'login';
            break;
        }
        return redirect()->guest(route($login));
    }

6) open middleware app\Http\Middleware\RedirectIfAuthenticated.php and change handle function like below

 public function handle(Request $request, Closure $next, ...$guards)
    {
        $guards = empty($guards) ? [null] : $guards;

        foreach ($guards as $guard) {
            switch ($guard) {
                case 'admin':
                if (Auth::guard($guard)->check()) {
                    return redirect()->route('admin.home');
                }
                break;

                default:
                if (Auth::guard($guard)->check()) {
                    return redirect('/home');
                }
                break;
           }
        }

        return $next($request);
    }

admin logout-----------------------------------------

1) add route for admin login

 Route::post('/logout', [LoginController::class, 'logout'])->name('admin.logout');

2) add this logout method to app\Http\Controllers\Admin\Auth\LoginController.php
/**
     * Log the user out of the application.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\RedirectResponse|\Illuminate\Http\JsonResponse
     */
    public function logout()
    {
        Auth::guard('admin')->logout();

        return redirect()->route('admin.login');
    }

3) change constructor in app\Http\Controllers\Admin\Auth\LoginController.php

 $this->middleware('guest:admin', ['except'=>['logout']]);

forgot password ---------------------------------------------------

1) open config\auth.php and modify password broker like below

'passwords' => [
    'users' => [
        'provider' => 'users',
        'table' => 'password_resets',
        'expire' => 60,
        'throttle' => 60,
    ],
    'admins' => [
        'provider' => 'admins',
        'table' => 'password_resets',
        'expire' => 60,
        'throttle' => 60,
    ],
],

2) open app\Http\Controllers\Admin\Auth\ResetPasswordController.php and put below code
use Illuminate\Http\Request;
use Password;
use Auth;

-> modify redirectTo

protected $redirectTo = '/admin';

-> modify __consrtuct() function as below

/**
 * Creating a new controller instance
 *
 * @return void
 */
public function __consrtuct()
{
  $this->middleware('guest:admin');
}

->put a guard like below

/**
 * [guard description]
 * @return [type] [description]
 */
  protected function guard()
  {
    return Auth::guard('admin');
  }

-> put password broker function

/**
 * [broker description]
 * @return [type] [description]
 */
  protected function broker()
  {
    return Password::broker('admins');
  }




3) put construct function in app\Http\Controllers\Admin\Auth\ForgotPasswordController.php

use Password;

/**
 * Creating a new controller instance
 *
 * @return void
 */
public function __consrtuct()
{
  $this->middleware('guest:admin');
}

 -> put password broker like below

 /**
  * [broker description]
  * @return [type] [description]
  */
   protected function broker()
   {
     return Password::broker('admins');
   }

  -> take code from Illuminate\Foundation\Auth\SendsPasswordResetEmails; and modify like this it is use for view

  /**
   * Display the form to request a password reset link.
   *
   * @return \Illuminate\View\View
   */
  public function showLinkRequestForm()
  {
      return view('admin.auth.passwords.email');
  }



 4) setup views

 1) open resources\views\admin\auth\passwords\email.blade.php and change below things

   ->form action change :  action="{{ route('admin.password.email') }}
   ->form name change   : {{ __('Admin Reset Password') }}

  2) open resources\views\admin\auth\passwords\reset.blade.php and change below places

  ->form action change :  action="{{ route('admin.password.request') }}
  ->form name change   : {{ __('Admin Reset Password') }}

  5) setup notification

  1) put below code in admin model app\Models\Admin\Admin.php

  use App\Notifications\AdminResetPasswordNotification;

  /**
   * Send the password reset notification.
   *
   * @param  string  $token
   * @return void
   */
  public function sendPasswordResetNotification($token)
  {
     $this->notify(new AdminResetPasswordNotification($token));
  }

  2) run below command "php artisan make:notification AdminResetPasswordNotification" and open app\Notifications\AdminResetPasswordNotification.php and do the change as below
->
  public $token;

  /**
   * Create a new notification instance.
   *
   * @return void
   */
  public function __construct($token)
  {
      $this->token = $token;
  }

  /**
   * Get the mail representation of the notification.
   *
   * @param  mixed  $notifiable
   * @return \Illuminate\Notifications\Messages\MailMessage
   */
  public function toMail($notifiable)
  {
      return (new MailMessage)
                  ->line('You are receiving this email because we receive a password reset request for your account.')
                  ->action('Reset Password', route('admin.password.reset', $this->token))
                  ->line('If you did not request a password reset, no further action is required,');
  }

  3) open app\Http\Controllers\Auth\ResetPasswordController.php and add below code

  /**
   * Display the password reset view for the given token.
   *
   * If no token is present, display the link request form.
   *
   * @param  \Illuminate\Http\Request  $request
   * @return \Illuminate\Contracts\View\Factory|\Illuminate\View\View
   */
  public function showResetForm(Request $request)
  {
      $token = $request->route()->parameter('token');

      return view('admin.auth.passwords.reset')->with(
          ['token' => $token, 'email' => $request->email]
      );
  }

  4) open resources\views\admin\auth\login.blade.php and change route like below

  @if (Route::has('admin.password.request'))
      <a class="btn btn-link" href="{{ route('admin.password.request') }}">
          {{ __('Forgot Your Password?') }}
      </a>
  @endif
