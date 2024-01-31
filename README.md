```ts
type Author = {
    name: 'Chris Thoburn';
    alias: '@runspired';
    title: 'Software Engineer';
    description: 'OpenSource Contributor to Data Frameworks';
    topics: 'WarpDrive' | 'EmberData' | 'EdgeData' | 'ServerSideData';
};
```

## Posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>