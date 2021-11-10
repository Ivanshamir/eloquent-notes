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
2. 
