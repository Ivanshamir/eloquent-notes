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
});
```
8. **Using unions to run query independetly:** About union: when you run union between multiple queries, each query in that union must return exact number of columns. If we want to run a single query instead of multiple query of previous query, then in raw sql:
```
select *
from users
where id in (
    select id from (
        select id
        from users
        where first_name like 'bill%' or last_name like 'bill%'
        
        union
        
        select users.id
        from users
        inner join companies on companies.id = users_company_id
        where companies.name like 'bill%'
    ) as matches
) and id in (
    select id from (
        select id
        from users
        where first_name like 'microsoft%' or last_name like 'microsoft%'
        
        union
        
        select users.id
        from users
        inner join companies on companies.id = users_company_id
        where companies.name like 'microsoft%'
    ) as matches
) 
```
Now time to implement it via eloquent:
```
public function scopeSearch($query, string $terms=null){
    collect(str_getcsv($terms, ' ', '"'))->filter()->each(function ($terms) use ($query){  //The str_getcsv() function parses a string for fields in CSV format and returns an array containing the fields read.
        $term = $term. '%';
        $query->whereIn('id', function ($query) use ($term){  
            $query->select('id')
                ->from(function ($query) use ($term){
                          $query->select('id')
                          ->from('users')
                          ->where('first_name', 'like', $term)
                          ->orWhere('last_name', 'like', $term)
                          ->union(
                              $query->newQuery()
                                  ->select('users.id')
                                  ->from('users')
                                  ->join('companies', 'companies.id', '=', 'users.company_id')
                                  ->where('companies.name', 'like', $term)
                          );
                }, 'matches');
        });
    });
});
```
9. **Searching using regular expressions:** If our string has quotation/others then searching query and indexing will not work. For this we add a virtual column in database/migration file of searched columns. Like in previous query, we need to add name virtual column in company table and vice versa. So in migartion file:
```
$table->string('name_normalized')->virtualAs("regexp_replace(name, '[^A-Za-z0-9]', '')")->index();
```
We will update users migartion file also. So the final look of previous query will be:
```
collect(str_getcsv($terms, ' ', '"'))->filter()->each(function ($terms) use ($query){  //The str_getcsv() function parses a string for fields in CSV format and returns an array containing the fields read.
        $term = preg_replace('/[^A-Za-z0-9]/', '', $term). '%';
        $query->whereIn('id', function ($query) use ($term){  
            $query->select('id')
                ->from(function ($query) use ($term){
                          $query->select('id')
                          ->from('users')
                          ->where('first_name_normalized', 'like', $term)    //in here first_name_normalized is the virtual column
                          ->orWhere('last_name_normalized', 'like', $term)
                          ->union(
                              $query->newQuery()
                                  ->select('users.id')
                                  ->from('users')
                                  ->join('companies', 'companies.id', '=', 'users.company_id')
                                  ->where('companies.name_normalized', 'like', $term)
                          );
                }, 'matches');
        });
    });
```
10. **Compound indexes:** if we use multiple order_by columns in query, for better performance we use them together by composite indexes. For composite indexes, we need to add ```$table->index(['last_name', 'first_name']);``` in migartion file. Note: we need to sorting accordingly by query or there will be a performance issue.
11. **Ordering by has-one relationship:** When use order-by using has-one relationships, you can use joining. This is more faster than subquery. Using join:
```
$users = User::query()
         ->select('users.*')  //by default laravel will fetch all columns both users table and companies table
         ->join('companies', 'companies.user_id', '=', 'users.id')
         ->orderBy('companies.name')
         ->with('company')
         ->paginate();
```
12. **Ordering by has-many relationships:** By ordering of has-many relationships, using sub-query is more faster than joining. 
```
$users = User::query()
          ->orderByDesc(Login::select('created_at')
              ->whereColumn('user_id', 'users.id')
              ->latest()
              ->take(1)
          )
          ->withLastLogin()
          ->paginate()
```
13. **Ordering with nulls always last:** If we want to set null value in last during sort, we can use the query in `orderByRaw` method. Like: in a table where we want to sort by clicking a specific column, so:
```
$users = User::query()
          ->when(request('sort') === 'town', function($query){
              $query->orderByRaw('town is null')
              ->orderBy('town', request('direction'));
          })
          ->orderBy('name')
          ->paginate();
```
We can write this command easily in postgres by using `nulls last` command. So we add this in existing query by using macro. So in `AppServiceProvider`:
```
Builder::macro('orderByNullsLast', function($column, $direction = 'asc'){
    $column = $this->getGrammar()->wrap($column);
    $direction = strtolower($direction) === 'asc' ? 'asc' : 'desc';
    return $this->orderByRaw("$column $direction nulls last");
});
```
Now in original query:
```
$users = User::query()
          ->when(request('sort') === 'town', function($query){
              $query->orderByNullsLast('town', request('direction'));
          })
          ->orderBy('name')
          ->paginate();
```
14. **Ordering by custom algorithm:** Suppose we have feature table. now we want to sort by column in different manner. So in Feature query:
```
$query = Feature::query()
          ->withCount('comments', 'votes')
          ->when(request('sort'), function($query, $sort){
              switch($sort){
                  case 'title': return $query->orderBy('title', request('direction'));
                  case 'status': return $query->orderByStatus(request('direction'));
                  case 'activity': return $query->orderByActivity(request('direction'));
              }
          })
          ->latest()
          ->paginate();
```
Now first we create `orderByStatus` `scope` in `Feature` model:
```
public dunction scopeOrderByStatus($query, $direction)
{
    $query->orderBy(DB::raw('
        case 
            when status = "Requested" then 1   //we assume this number and sort by this number
            when status = "Approved" then 2
            when status = "Completed" then 3
        end
    '), $direction);
}
```
Now we want to add index of this custom algorithm in features migration:
```
$table->rawIndex('(
    case 
          when status = "Requested" then 1 
          when status = "Approved" then 2
          when status = "Completed" then 3
    end
)', 'features_status_ranking_index');
```
Now we add scope for `orderByActivity` in features model. As we already used `withCount` , we can easily use `comments_count` and `votes_count`. Because eloquent run subqueries and assign then in new column name `comments_count` and `votes_count`.
```
public function scopeOrderByActivity($query, $direction)
{
    $query->orderBy(
        DB::raw('-(votes_count + (comments_count * 2))'), //because we nee to show lowest value first
        $direction
    );
}
```
We cant use index because this actually depends on subquery
15. **Filtering and sorting date month:** 
```
$query->orderByRaw('date_format(birth_date, "%m-%d")');
```
Now for faster process we need to add composite index in migration file:
```
$table->rawIndex("(date_format(birth_date, '%m-%d')), name", 'users_birthday_name_index');
```
Now if we want to show only birthday of current week:
```
$dates = Carbon::now()->startOfWeek()
          ->daysUntil(Carbon::now()->endOfWeek())
          ->map(fn ($date) => $date->format('m-d'));
$query->whereRaw('date_format(birth_date, '%m-%d') in (?,?,?,?,?,?,?)', iterator_to_array($dates));
```
16. **Full Text Search:** First we neet to create full text search index in Posts migration file. Since laravel doesn't support that, we need to add statement in manually.
```
DB::statement('
    create fulltext index posts_fulltext_index
    on posts(title, body)
    with parser ngram
');
```
Now in query:
```
$posts = Post::query()
          ->with('author')
          ->when(request('search'), function($query, $search){
              $query->selectRaw('*, match(title, body) against(? in boolean mode) as score', [$search])
              ->whereRaw('match(title, body) against(? in boolean mode)', [$search]);
          })
```
For more details: [Mysql Notes](/sql.md)

17. **Distance between geographic points:** Suppose we add a scope with main query and in scope we need to check the all columns have been selected
```
public function scopeSelectDistanceTo($query, array $coordinates)
{
    if(is_null($query->getQuery()->columns)){
        $query->select('*');
    }
    $query->selectRaw('ST_Distance(
        ST_SRID(Point(longitude, latitude), 4326),   //4326 is the special reference identifier and it means world
        ST_SRID(Point(?,?), 4326)
    ) as distance', $coordinates);
}
```
By default the result will be in meters.
18. **Filtering by geographic distance:** If we want to add distance, then with previous query:
```
 $query->selectRaw('ST_Distance(
        ST_SRID(Point(longitude, latitude), 4326),  
        ST_SRID(Point(?,?), 4326)
    ) as distance <= ?', [...$coordinates, $distance]);
```
Now if we want to make more faster then instead of two columns name langitude and longitude in migration file, we can add just one column:
```
$table->point('location', 4326);
```
So in query:
```
 $query->selectRaw('ST_Distance(
        location, 
        ST_SRID(Point(?,?), 4326)
    ) as distance <= ?', [...$coordinates, $distance]);
```
19. **Ordering by distance** This will almost same of previous query:
```
$direction = strtolower($direction) === 'asc' ? 'asc' : 'desc';
 $query->orderByRaw('ST_Distance(
        location, 
        ST_SRID(Point(?,?), 4326)
    ) '.$directon, $coordinates);
```
20. 
