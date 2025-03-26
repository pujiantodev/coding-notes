
# ✅ RANGKUMAN OPTIMASI QUERY UNTUK DATA BESAR DI LARAVEL (10 JUTA ROWS)

---

## 🎯 Tujuan
Mengoptimalkan performa query list data dari tabel besar (`mutations`, 10 juta baris), agar:
- ⚡ Cepat (sub-second query time)
- ✅ Modular & scalable
- 💡 Mudah dirawat dan dikembangkan
- 🔐 Aman dengan validasi input

---

## 🧩 Masalah Awal
- Query lambat (hingga 45 detik), meskipun sudah ada index.
- Banyak `when()` di controller membuat kode kompleks.
- Menggunakan `simplePaginate()` yang masih menghitung total data.
- Relasi berat (`with('mutationRetur')`) selalu di-load.

---

## ✅ Solusi Utama

### 1. Gunakan pendekatan **modular filter class**:
- Setiap filter dibuat di class-nya sendiri: `Search.php`, `CashFlow.php`, dll.
- Mudah dikembangkan saat filter bertambah.
- Dapat digunakan lintas fitur: list, export, reporting.

### 2. Gunakan `cursorPaginate()` untuk performa tinggi:
- Tidak menghitung total data.
- Lebih ringan dari `paginate()` dan `simplePaginate()`.
- Cocok untuk dataset besar.

### 3. Validasi input request pakai **FormRequest**:
- Lebih aman, mencegah query tidak valid.
- Menampilkan pesan error otomatis.

---

## 🔁 Dua Pendekatan Modular Filtering

---

### 🧩 Pendekatan 1: Manual panggil filter class di controller

#### Contoh Filter Class: `Search.php`
```php
namespace App\Filters\MutationFilters;

use Illuminate\Database\Eloquent\Builder;

class Search
{
    public static function apply(Builder $query, $value, $type)
    {
        if (!$value) return $query;

        return match ($type) {
            'description' => $query->where('description', 'ilike', "%$value%"),
            'debit'       => $query->where('debit', $value),
            'credit'      => $query->where('credit', $value),
            default       => $query->where('key_code', $value),
        };
    }
}
```

#### Di Controller:
```php
use App\Filters\MutationFilters\Search;

public function index(Request $request)
{
    $query = Mutation::select(...)->with('bank:id,name')->orderByDesc('id');
    $query = Search::apply($query, $request->search, $request->type ?? 'key_code');
    return $query->cursorPaginate(20);
}
```

---

### 🧱 Pendekatan 2: Dynamic Modular Filtering dengan Trait `Filterable`

#### Filterable.php
```php
namespace App\Traits;

use Illuminate\Http\Request;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Support\Str;

trait Filterable
{
    public function scopeFilter(Builder $query, Request $request): Builder
    {
        $modelName = class_basename($query->getModel());
        $namespace = 'App\\Filters\\' . $modelName . 'Filters\\';

        foreach ($request->all() as $key => $value) {
            $class = $namespace . Str::studly($key);

            if (class_exists($class) && method_exists($class, 'apply')) {
                $query = $class::apply($query, $value);
            }
        }

        return $query;
    }
}
```

#### Di Model:
```php
use App\Traits\Filterable;

class Mutation extends Model
{
    use Filterable;
}
```

#### Di Controller:
```php
public function index(ListMutationRequest $request)
{
    return Mutation::with('bank:id,name')
        ->filter($request)
        ->orderByDesc('id')
        ->cursorPaginate(20);
}
```

---

## 🛡️ Validasi dengan Form Request

### ListMutationRequest.php
```php
class ListMutationRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'search' => ['nullable', 'string'],
            'type' => ['nullable', 'in:key_code,description,debit,credit'],
            'start_date' => ['nullable', 'date'],
            'end_date' => ['nullable', 'date'],
            'is_key_code_null' => ['nullable', 'in:true,false'],
            'bank' => ['nullable', 'exists:banks,id'],
            'cash_flow' => ['nullable', 'in:credit,debit'],
            'doesnt_have_refund' => ['nullable', 'in:true'],
        ];
    }
}
```

---

## 📦 Perbandingan dengan Spatie Laravel Query Builder

| Fitur | Custom Filterable (Punyamu) | Spatie Laravel Query Builder |
|-------|------------------------------|-------------------------------|
| Performa Data Besar | ✅ Sangat cocok (`cursorPaginate`) | ❌ Tidak mendukung |
| Modular & Clean | ✅ | ✅ |
| Include relasi | Manual | Built-in `allowedIncludes()` |
| Sorting | Manual | Built-in `allowedSorts()` |
| Validasi | Laravel-native | Harus ditangani manual |
| Dokumentasi & Komunitas | Custom | Lengkap dan populer |

**Kesimpulan:** Pendekatan `Filterable + cursorPaginate` adalah pilihan terbaik untuk dataset besar seperti milikmu.

---

## 🧠 Tips Tambahan
- Gunakan index di kolom: `key_code`, `post_date`, `bank_id`, `debit`, `credit`
- Gunakan `EXPLAIN ANALYZE` untuk cek performa query
- Hindari relasi berat kecuali dibutuhkan (`mutationRetur`)
- Gunakan `withQueryString()` jika ingin pagination mempertahankan filter

---

## ✅ Penutup

Dengan pendekatan ini, kamu sekarang punya sistem:
- 💨 Super cepat
- 🧱 Modular
- 🧼 Bersih & maintainable
- 🔒 Aman (dengan validasi FormRequest)
