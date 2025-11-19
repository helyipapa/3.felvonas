# SANCTUM – BEARER TOKEN AUTENTIKÁCIÓ

Ez a rövid "puska" összefoglalja, hogyan állíts be Laravel Sanctum Bearer token alapú autentikációt, egy egyszerű AuthController-rel (register, login, logout) és egy Reservation erőforrást jogosultságokkal (admin vs user).

---

## 1. Telepítés

Futtasd a következő parancsokat:

```bash
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```

Ne felejtsd el: a `User` modellben engedélyezni a tokeneket.

---

## 2. User modell (User.php)

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;

    protected $fillable = [
        'name',
        'email',
        'password',
        'is_admin'
    ];
}
```

`is_admin` mezővel jelöljük az admin felhasználókat (boolean).

---

## 3. AuthController (register, login, logout)

Példa controller metódusok:

- register()

```php
public function register(Request $r)
{
    $r->validate([
        'name' => 'required',
        'email' => 'required|email|unique:users',
        'password' => 'required|min:6'
    ]);

    return User::create([
        'name' => $r->name,
        'email' => $r->email,
        'password' => Hash::make($r->password),
        'is_admin' => false
    ]);
}
```

- login()

```php
public function login(Request $r)
{
    $user = User::where('email', $r->email)->first();

    if (!$user || !Hash::check($r->password, $user->password)) {
        return response()->json(['msg' => 'Invalid'], 401);
    }

    return [
        'access_token' => $user->createToken('auth')->plainTextToken,
        'token_type' => 'Bearer'
    ];
}
```

- logout()

```php
public function logout(Request $r)
{
    $r->user()->tokens()->delete();
    return ['message' => 'Logout OK'];
}
```

Fontos: a `createToken()` a Plain Text tokenet adja vissza, amit a kliensnek el kell tárolnia (pl. Authorization: Bearer <token>).

---

## 4. API útvonalak (routes/api.php)

```php
Route::post('/register', [AuthController::class, 'register']);
Route::post('/login', [AuthController::class, 'login']);

Route::middleware('auth:sanctum')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout']);
    Route::apiResource('reservations', ReservationController::class);
});
```

A `auth:sanctum` middleware biztosítja, hogy csak bejelentkezett felhasználók érhessék el a védett végpontokat.

---

## 5. Reservation migráció

Példa migráció a `reservations` tábla létrehozásához:

```php
Schema::create('reservations', function (Blueprint $table) {
    $table->id();
    $table->unsignedBigInteger('user_id');
    $table->dateTime('reservation_time');
    $table->integer('guests');
    $table->string('note')->nullable();
    $table->foreign('user_id')->references('id')->on('users')->onDelete('cascade');
    $table->timestamps();
});
```

---

## 6. ReservationController — jogosultságok és műveletek

Alapfeltételek: a kontrollerben minden releváns akció előtt ellenőrizd a jogosultságokat (admin lát mindent, sima user csak a saját foglalásait).

- index()

```php
public function index(Request $request)
{
    $user = $request->user();

    if ($user->is_admin) {
        return Reservation::all();
    }

    return Reservation::where('user_id', $user->id)->get();
}
```

- show()

```php
public function show(Request $request, Reservation $reservation)
{
    $user = $request->user();

    if (! $user->is_admin && $reservation->user_id !== $user->id) {
        return response()->json(['Unauthorized'], 403);
    }

    return $reservation;
}
```

- update()

Validáció és frissítés:

```php
public function update(Request $r, Reservation $reservation)
{
    $user = $r->user();

    if (! $user->is_admin && $reservation->user_id !== $user->id) {
        return response()->json(['Unauthorized'], 403);
    }

    $r->validate([
        'reservation_time' => 'sometimes|date',
        'guests' => 'sometimes|integer|min:1',
        'note' => 'nullable|string'
    ]);

    $reservation->update($r->only(['reservation_time', 'guests', 'note']));

    return $reservation;
}
```

- destroy()

```php
public function destroy(Request $r, Reservation $res)
{
    $user = $r->user();

    if (! $user->is_admin && $res->user_id != $user->id) {
        return response()->json(['Unauthorized'], 403);
    }

    $res->delete();

    return response()->json(['message' => 'Deleted'], 200);
}
```

---

## Rövid lényeg

- ✔ Sanctum Bearer token: használd a User::createToken()->plainTextToken-et
- ✔ auth:sanctum middleware → csak bejelentkezve érhető el a védett API
- ✔ Admin: minden foglalást lát
- ✔ User: csak a saját foglalásait látja / módosíthatja / törölheti
- ✔ ReservationController-ben minden művelet előtt jogosultság-ellenőrzés

---

Ha szeretnéd, készítek egy komplett mintaprojektet vagy egy Pull Request-et a kód beillesztéséhez a repo-dba (pl. AuthController és ReservationController teljes file-okkal), vagy hozzáadok példákat Postman/HTTP kérésekre (register/login + használat Bearer tokennel). Mit szeretnél következő lépésként?
