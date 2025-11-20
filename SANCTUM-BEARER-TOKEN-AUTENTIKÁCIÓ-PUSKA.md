# SANCTUM – BEARER TOKEN AUTENTIKÁCIÓ

Ez a rövid "puska" összefoglalja, hogyan állíts be Laravel Sanctum Bearer token alapú autentikációt, egy egyszerű AuthController-rel (register, login, logout) és egy Reservation erőforrást jogosultságokkal (admin vs user).

---

## 1. Laravel Sanctum telepítése

A token alapú autentikációhoz fel kell telepíteni a Sanctum komponenst. Ez a csomag képes úgynevezett "Personal Access Token"-eket létrehozni, amelyeket a kliens minden API kérés során Bearer Token formában küld vissza.
Futtasd a következő parancsokat:

```bash
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```

A második parancs telepíti a Sanctum konfigurációját és migrációit.
A migráció létrehoz egy táblát, ahol a felhasználói tokenek lesznek tárolva.
Ne felejtsd el: a `User` modellben engedélyezni a tokeneket.

---

## 2. User modell módosítása

A User modellben engedélyezni kell az API tokenek támogatását.
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
Magyarázat:
A HasApiTokens trait biztosítja, hogy a felhasználó képes legyen tokeneket létrehozni ($user->createToken()).

Az is_admin mező egy boolean, amely megkülönbözteti az admin felhasználót a normál usertől.

Ezt kézzel kell felvenni az adatbázisban, migrációval vagy manuálisan.

`is_admin` mezővel jelöljük az admin felhasználókat (boolean).

---

## 3. AuthController
### Register, Login, Logout részletes magyarázattal

Példa controller metódusok:
A regisztráció és bejelentkezés nyilvános, a kijelentkezés csak autentikált felhasználóknak érhető el.

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
Magyarázat:

A paraméterek validálásra kerülnek.

A jelszó mindig hashelve kerül mentésre (Hash::make).

A is_admin alapértelmezésben false, így a felhasználó nem lesz admin.

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
Magyarázat:

A rendszer megkeresi a felhasználót email alapján.

Hash::check ellenőrzi a jelszót.

Siker esetén létrejön egy új personal access token.

A token sima szövegként kerül visszaadásra (plainTextToken).

A kliens ezt köteles eltárolni és Bearer Tokenként küldeni.

- logout()

```php
public function logout(Request $r)
{
    $r->user()->tokens()->delete();
    return ['message' => 'Logout OK'];
}
```
Magyarázat:

A bejelentkezett felhasználó összes tokenjét törli.

A felhasználó így ki van jelentkeztetve minden eszközön.

Fontos: a `createToken()` a Plain Text tokenet adja vissza, amit a kliensnek el kell tárolnia (pl. Authorization: Bearer <token>).

---

## 4. API útvonalak (routes/api.php)

Ez a fájl tartalmazza az összes API végpontot.

### 4.1 Publikus útvonalak

```php
Route::post('/register', [AuthController::class, 'register']);
Route::post('/login', [AuthController::class, 'login']);

Route::middleware('auth:sanctum')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout']);
    Route::apiResource('reservations', ReservationController::class);
});
```
Magyarázat:

Ezekhez NEM szükséges token.

A login végpont visszaadja a Bearer tokent.

A `auth:sanctum` middleware biztosítja, hogy csak bejelentkezett felhasználók érhessék el a védett végpontokat.

### 4.2 Védett útvonalak (auth:sanctum)

```php
Route::middleware('auth:sanctum')->group(function(){
    Route::post('/logout',[ReservationController::class, 'logout']);
    Route::apiResource('reservations', ReservationController::class);
});
```
Magyarázat:

A auth:sanctum middleware megköveteli, hogy a kérés tartalmazzon érvényes Bearer tokent.

A apiResource('reservations') automatikusan létrehozza:

GET /reservations → index

POST /reservations → store

GET /reservations/{id} → show

PUT/PATCH /reservations/{id} → update

DELETE /reservations/{id} → destroy
---

## 5.Reservation migráció

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
Magyarázat:

user_id: a foglalást létrehozó user.

reservation_time: dátum és idő.

guests: vendégek száma.

note: opcionális megjegyzés.

A foreign key biztosítja, hogy ha egy user törlődik, a foglalásai is törlődjenek.

---

## 6. ReservationController – részletes magyarázat minden metódushoz

A ReservationController felelős a foglalások kezeléséért.

Az admin látja és módosíthatja MINDENKI foglalását.

A sima felhasználó csak a sajátját láthatja és módosíthatja.

### 6.1 index() – foglalások listázása

```php
public function index(Request $request){
    $user = $request->user();
    if ($user->isAdmin){
        $reservations = Reservation::all();
    } else {
        $reservations = Reservation::where('user_id', $user->id)->get();
    }

    return response()->json($reservations, 200);
}
```
Magyarázat:

Admin: minden foglalást lát.

Felhasználó: csak a saját foglalásait.

A user() metódus mindig a tokenhez tartozó user-t adja vissza.

### 6.2 show() – egy adott foglalás lekérése

```php
public function show(Request $request, $id){
    $user = $request->user();
    $reservation = Reservation::findOrFail($id);
    
    if(!$user->is_admin && $reservation->user_id != $user->id){
        return response()->json(['message' => 'Nincs jogosultságod megtekinteni ezt a foglalást!'], 403);
    }
    
    return response()->json($reservation, 200);
}
```
Magyarázat:

A rendszer megkeresi az adott foglalást.

Jogosultságellenőrzés:

admin: mindig láthatja,

user: csak a sajátját.

### 6.3 store() – új foglalás létrehozása

```php
public function store(Request $request){
    $validated = $request->validate([
        'reservation_time' => 'required|date',
        'guests' => 'required|integer|min:1',
        'note' => 'nullable|string',
    ]);

    $reservation = Reservation::create([
        'user_id' => $request->user()->id,
        'reservation_time' => $validated['reservation_time'],
        'guests' => $validated['guests'],
        'note' => $validated['note'] ?? null,
    ]);

    return response()->json($reservation, 201);
}
```
Magyarázat:

Kötelező mezők validálása.

A foglalás mindig az adott userhez tartozik.

201-es státuszkód jelzi a sikeres létrehozást.

### 6.4 update() – foglalás módosítása

```php
public function update(Request $request, $id){
    $user = $request->user();
    $reservation = Reservation::findOrFail($id);
    
    if(!$user->is_admin && $reservation->user_id != $user->id){
        return response()->json(['message' => 'Nincs jogosultságod módosítani ezt a foglalást!'], 403);
    }

    $validated = $request->validate([
        'name' => 'sometimes|required|string',
        'email' => 'sometimes|required|email',
        'reservation_time' => 'sometimes|required|date',
        'guests' => 'sometimes|required|integer|min:1',
        'note' => 'nullable|string',
    ]);

    $reservation->update($validated);
    return response()->json($reservation, 200);
}
```
Magyarázat:

A sometimes azt jelenti, hogy a mező nem kötelező, de ha küldik, validálni kell.

Jogosultság:

admin: bármit módosíthat,

user: csak a saját foglalását.

A update() automatikusan módosítja a mezőket.

### 6.5 destroy() – foglalás törlése

```php
public function destroy(Request $request, $id){
    $user = $request->user();
    $reservation = Reservation::findOrFail($id);

    if(!$user->is_admin && $reservation->user_id != $user->id){
        return response()->json(['message' => 'Nincs jogosultságod törölni ezt a foglalást!'], 403);
    }

    $reservation->delete();
    return response()->json(['message' => 'Foglalás törölve.'], 200);
}
```
Magyarázat:

Admin törölhet bármit.

A user csak a sajátját.

A delete() végleg törli a rekordot.
