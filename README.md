# eloquent-notes
This repository is for eloquent notes for different type of optimization.

1. If memory usage is so high after using eager load, we select necessary columns of both parent query and eager load query:
```
$posts = Post::query()
          ->select('id', 'title', 'slug', 'publised_at')
          ->with('author:id, name')
          ->latest('published_at')
          ->get()
          ->groupBy(fn ($post) => $post->published_at->year);
```
2. **Getting one record from has many relationships:** If we face high memory problem after using eager load of has many relationships, we can use `addSelect` in query. From doc: Eloquent also offers advanced subquery support, which allows you to pull information from related tables in a single query.
```
$users = User::query
          ->addSelect(['last_login_at' => Login::select('created_at')
                    ->whereColumn('user_id', 'users.id')
                    ->latest()
                    ->take(1)
          ])
          ->orderBy('name')
          ->paginate();
```
If we want to change column value, we can use query time casting. So if we want to add query casting in this query after addSelect, we can add: `->withCasts(['last_login_at' => 'datetime'])`

3. **Dynamic relationships using sub-queries:** If we need to multiple column value of above query, instead of adding `addSelect` one by one, we can use dynamic relationships. So that first we add `belongsTo` relation in `User` model.
```
public function lastLogin()
{
    return $this->belongsTo(Login::class);
}
```
Now add a scope in model:
```
public function scopeWithLastLogin($query)
{
    $query->addSelect(['last_login_at' => Login::select('created_at')
                    ->whereColumn('user_id', 'users.id')
                    ->latest()
                    ->take(1)
          ])->with('lastLogin');
}
```
So the real query will be:
```
$users = User::query
          ->withLastLogin()
          ->orderBy('name')
          ->paginate();
```
4. **Calculating totals using conditional aggregates:** If we want to show aggreagation by different type of condition, we can use `case` in sql:
```
SELECT 
    count(case when status = 'Requested' then 1 end) AS requested,
    count(case when status = 'Planned' then 1 end) AS planned
FROM featured
```
Now in eloquent model:
```
Feature::toBase()
    ->selectRaw("count(case when status = 'Requested' then 1 end) AS requested")
    ->selectRaw("count(case when status = 'Planned' then 1 end) AS planned")
    ->first();
```
In here we use `toBase` because of we dont want to return the model but we want to use collection of this model. In poistgres we can use `filter`:
```
Feature::toBase()
    ->selectRaw("count(*) filter (when status = 'Requested') AS requested")
    ->selectRaw("count(*) filter (when status = 'Planned') AS planned")
    ->first();
```
5. **Optimizing Circular Relationships:** If eager load operates same queries in multiple time we can use in-memory relationships. When you eager load a relationship, using the $album->songs property shorthand doesn't perform a new query; instead it fetches the Songs from a memoized property on the Album.

We can pretend to eager load a relationship manually using the setRelation method:
```
  public function test_can_calculate_total_duration()
  {
      $album = factory('Album')->create();

      $songs = new Collection([
          factory('Song')->make(['duration' => 291]),
          factory('Song')->make(['duration' => 123]),
          factory('Song')->make(['duration' => 100]),
      ]);

     $album->songs()->saveMany($songs);
     $album->setRelation('songs', $songs);

      $this->assertEquals(514, $album->duration);
  }
```
Now we can get rid of the DatabaseMigrations trait and use make to build our Album instead of create, leaving us with a test that looks like this:
```
class AlbumTest extends TestCase
{
    public function test_can_calculate_total_duration()
    {
        $album = factory('Album')->make();

        $songs = new Collection([
            factory('Song')->make(['duration' => 291]),
            factory('Song')->make(['duration' => 123]),
            factory('Song')->make(['duration' => 100]),
        ]);

        $album->setRelation('songs', $songs);

        $this->assertEquals(514, $album->duration);
    }
}
```
Running this test still passes, but this time it only takes about 3ms to finish! That's thirty times faster.
6. **Multi column searching:** Suppose one user belongs to one compnay but one company has multiple users. So we want to search user's first name, last name, company name in such a way that every search terms must be recorded by any columns. So:
```
$results = User::query()
                 ->search(request('search'))
                 ->with('company')
                 ->paginate()
```
Now we create seacrh scope in User model.
```
public function scopeSearch($query, string $terms=null){
    collect(' ', $terms)->filter()->each(function ($terms) use ($query){
        $term = '%'. $term. '%';
        $query->where(function ($query) use ($term){   //we use callback because we need every search term in isolation
            $query->where('first_name', 'like', $term)
                  ->orWhere('last_name', 'like', $term)
                  ->orWhereHas('company', function($query) use ($term){
                      $query->where('name', 'like', $term);
                  })
        });
    });
}
```
7. **Run Additional queries:** The previous query takes huge execution time. Also after adding index, it doesn't executes indexes. We can experiment it by placing before `EXPLAIN` command of raw query of mysql. So first we need to remove wildcard(`%`) from `$term = '%'. $term. '%';` this line. Also need to execute additional query instead of  `orWhereHas`:
```
public function scopeSearch($query, string $terms=null){
    collect(str_getcsv($terms, ' ', '"'))->filter()->each(function ($terms) use ($query){  //The str_getcsv() function parses a string for fields in CSV format and returns an array containing the fields read.
        $term = $term. '%';
        $query->where(function ($query) use ($term){   //we use callback because we need every search term in isolation
            $query->where('first_name', 'like', $term)
                  ->orWhere('last_name', 'like', $term)
                  ->orWhereIn('company_id', Company::query()
                      ->where('name', 'like', $term)
                      ->pluck('id')
                  );
        });
    });
}
```
8. 
