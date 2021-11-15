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
4. **Counting totals using conditional aggregates:** If we want to show aggreagation by different type of condition, we can use `case` in sql:
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
5. 
